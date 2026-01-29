# Reducing Flaky Tests with Precondition Assertions and Strategic Test Grouping

*Brandon Palm, Principal Software Engineer, Telco Security*

---

We had a problem: platform alteration tests passing on OCP 4.18+ but consistently failing on 4.14, 4.16, and 4.17. The failures happened deep in test execution at `ValidateIfReportsAreValid()`, giving us no visibility into *why* they failed. After investigation, we implemented precondition assertions and strategic test grouping that fixed two of three failing versions and dramatically improved our debugging experience.

## The Problem

Our certsuite-qe test harness runs against multiple OCP cluster versions. The test pattern looks like this:

1. Deploy test workloads (Deployments, DaemonSets, StatefulSets)
2. Call `LaunchTests()` to run certsuite
3. Call `ValidateIfReportsAreValid()` to check the claim.json

When tests failed, the error was always the same unhelpful message:

```
failed to get the state of test case "platform-alteration-base-image"
from the claim report file, err: invalid test status skipped instead expected passed
```

This told us *what* failed but not *why*. Was it:
- MCO (MachineConfigOperator) not accessible?
- Pods not actually running?
- Container readiness issues?
- Node configuration problems?

We had no visibility into the state of the test environment before certsuite ran.

## The Investigation

We started by analyzing what the failing tests actually check:

**platform-alteration-base-image:**
- Uses `podman diff` to verify container filesystem layers
- Requires MCO and podman accessible on cluster nodes

**platform-alteration-boot-params:**
- Verifies kernel boot params match MachineConfig configuration
- Requires MCO API access and host filesystem access from pods

The existing precondition checks were minimal:

```go
// Only checked for Kind cluster - no OCP environment validation
if globalhelper.IsKindCluster() {
    Skip("...")
}

// Only checked deployment exists - no pod/container readiness
runningDeployment, err := globalhelper.GetRunningDeployment(...)
Expect(runningDeployment).ToNot(BeNil())
```

## The Solution: Layered Precondition Assertions

We added three layers of assertions that run *before* `LaunchTests()`:

### Layer 1: Environment Health Checks (BeforeEach)

```go
BeforeEach(func() {
    // ... existing setup ...

    By("Verify MCO is healthy and accessible")
    mcoHealthy, err := globalhelper.IsMCOHealthy()
    if err != nil || !mcoHealthy {
        Skip("MCO is not healthy or accessible on this cluster")
    }

    By("Verify cluster has worker nodes")
    if !globalhelper.HasWorkerNodes() {
        Skip("Cluster has no worker nodes")
    }

    By("Verify MachineConfigPools exist")
    mcpList, err := globalhelper.GetAPIClient().MachineConfigPools().List(...)
    if err != nil || len(mcpList.Items) == 0 {
        Skip("No MachineConfigPools found")
    }
})
```

### Layer 2: Workload Readiness Assertions (In Each Test)

```go
It("One deployment, one pod, running test image", func() {
    // Deploy workload
    err := globalhelper.CreateAndWaitUntilDeploymentIsReady(deployment, tsparams.WaitingTime)
    Expect(err).ToNot(HaveOccurred())

    // NEW: Assert pods are actually running with ready containers
    By("Assert pods are running with ready containers")
    podsList, err := globalhelper.GetListOfPodsInNamespace(randomNamespace)
    Expect(err).ToNot(HaveOccurred())
    Expect(len(podsList.Items)).To(BeNumerically(">", 0), "Expected at least one pod")

    for _, p := range podsList.Items {
        Expect(p.Status.Phase).To(Equal(corev1.PodRunning),
            fmt.Sprintf("Pod %s should be running", p.Name))
        for _, cs := range p.Status.ContainerStatuses {
            Expect(cs.Ready).To(BeTrue(),
                fmt.Sprintf("Container %s in pod %s should be ready", cs.Name, p.Name))
        }
    }

    // NOW we can run certsuite with confidence
    By("Start platform-alteration-base-image test")
    err = globalhelper.LaunchTests(...)
})
```

### Layer 3: DaemonSet-Specific Assertions

```go
By("Assert daemonSet has ready pods on nodes")
runningDaemonSet, err := globalhelper.GetRunningDaemonset(daemonSet)
Expect(err).ToNot(HaveOccurred())
Expect(runningDaemonSet.Status.NumberReady).To(BeNumerically(">", 0),
    "DaemonSet should have ready pods")
Expect(runningDaemonSet.Status.NumberReady).To(
    Equal(runningDaemonSet.Status.DesiredNumberScheduled),
    "All scheduled pods should be ready")
```

## Strategic Test Grouping: Reducing Retry Surface Area

Beyond assertions, we also looked at test organization. Ginkgo's retry mechanism (`flakyAttempts`) retries the entire `It()` block on failure. If your `It()` block contains multiple logical tests, a failure in any one causes everything to retry.

**Before: Large It() blocks**
```go
It("validates all base image scenarios", func() {
    // Test 1: Deployment with test image
    // Test 2: DaemonSet with test image
    // Test 3: StatefulSet with test image
    // If Test 3 fails, Tests 1 and 2 re-run unnecessarily
})
```

**After: Focused It() blocks**
```go
It("One deployment, one pod, running test image", func() { ... })
It("One daemonSet, running test image", func() { ... })
It("One statefulSet, one pod, running test image", func() { ... })
```

This approach:
- Reduces wasted compute on retries
- Provides clearer failure attribution
- Allows parallel execution of independent tests
- Makes CI logs easier to parse

## Results

| OCP Version | Before | After | Notes |
|-------------|--------|-------|-------|
| 4.14 | FAILURE | FAILURE | Identified as certsuite internal skip (not env issue) |
| 4.16 | FAILURE | SUCCESS | Fixed by precondition assertions |
| 4.17 | FAILURE | SUCCESS | Fixed by precondition assertions |
| 4.18 | SUCCESS | SUCCESS | No change needed |
| 4.19 | SUCCESS | SUCCESS | No change needed |
| 4.20 | SUCCESS | SUCCESS | No change needed |

The 4.14 failure was particularly interesting. Our assertions revealed that:
1. MCO was healthy
2. Worker nodes existed
3. Pods deployed successfully
4. Containers were ready

But certsuite itself was *skipping* the tests. This told us the issue wasn't in our test environment setup but in certsuite's internal logic for OCP 4.14. That's actionable intelligence we didn't have before.

## Key Patterns

### The Assertion Hierarchy

```
BeforeEach (Environment)
    ├── MCO health check
    ├── Worker node check
    └── MachineConfigPool check
        │
        ▼
It() Block (Workload)
    ├── Deploy resource
    ├── Wait for ready
    ├── Assert pod phase == Running
    ├── Assert all containers ready
    └── THEN call LaunchTests()
```

### Helper Functions We Added

```go
// globalhelper/ocp_checks.go

func IsMCOHealthy() (bool, error) {
    // Check MCO deployment exists and is ready
    mcoDeployment, err := GetRunningDeployment(
        "openshift-machine-config-operator",
        "machine-config-operator")
    if err != nil || mcoDeployment == nil {
        return false, err
    }

    // Check we can list MachineConfigs
    _, err = GetAPIClient().MachineConfigs().List(context.TODO(), metav1.ListOptions{})
    return err == nil, err
}

func HasWorkerNodes() bool {
    nodes, err := GetAPIClient().Nodes().List(context.TODO(), metav1.ListOptions{
        LabelSelector: "node-role.kubernetes.io/worker",
    })
    return err == nil && len(nodes.Items) > 0
}
```

## Lessons Learned

1. **Fail fast, fail informatively.** A test that fails immediately with "MCO not accessible" is infinitely more useful than one that times out after 10 minutes with a cryptic claim.json error.

2. **Assertions aren't just for correctness.** They're documentation. When someone reads `Expect(p.Status.Phase).To(Equal(corev1.PodRunning))`, they immediately understand what state the test requires.

3. **Skip vs Fail is a design decision.** We used `Skip()` for environment issues (not the test's fault) and `Expect()` failures for workload issues (something went wrong with our test setup).

4. **Smaller It() blocks = faster feedback.** The cost of test isolation is far outweighed by the benefits of precise failure attribution and reduced retry scope.

5. **Sometimes the fix reveals the real problem.** Adding assertions to 4.14 didn't fix it, but it proved the issue was upstream in certsuite, not in our test infrastructure.

## Closing

Flaky tests erode trust in CI. When tests fail intermittently without clear reasons, teams start ignoring failures, re-running pipelines "just to see if it passes," and losing confidence in their safety nets.

Precondition assertions transform opaque failures into actionable diagnostics. Combined with strategic test grouping, they reduce both the frequency and the blast radius of flaky test failures.

The changes we made were straightforward—about 50 lines of assertion code across 7 test files. The impact was immediate: two previously failing OCP versions now pass consistently, and we have clear evidence for why the third still fails.

---

*Related:*
- [PR #1334: Add precondition assertions to platform alteration tests](https://github.com/redhat-best-practices-for-k8s/certsuite-qe/pull/1334)
- [certsuite-qe repository](https://github.com/redhat-best-practices-for-k8s/certsuite-qe)
- [certsuite repository](https://github.com/redhat-best-practices-for-k8s/certsuite)
