# Why Use Certsuite: Your Fast Track to CNF Certification

*Brandon Palm, Principal Software Engineer, Telco Partner Ecosystem*

---

Getting your Cloud Native Function (CNF) certified for Red Hat OpenShift shouldn't feel like navigating a maze blindfolded. Yet many partners discover certification blockers late in development—when fixes are expensive and deadlines loom. Certsuite changes that equation by validating your workload against Red Hat's best practices before you submit for certification.

## The Certification Challenge

Partners building CNFs for OpenShift face a common problem: certification requirements span security configurations, networking policies, lifecycle management, and operator packaging. Without a way to validate these requirements during development, teams often discover issues only after submitting for formal certification—triggering costly rework cycles.

I've seen this pattern repeatedly: a vendor spends months building a CNF, submits for certification, and receives a list of issues that require architectural changes. What should have been a two-week certification process becomes a two-month delay.

## What Certsuite Validates

Certsuite is a comprehensive test suite that validates CNF deployments against Red Hat's best practices. It covers 121 test cases across 10 categories:

| Category | Test Cases | What It Validates |
|----------|------------|-------------------|
| Access Control | 28 | Security contexts, RBAC, capabilities |
| Lifecycle | 19 | Pod recreation, scaling, owner references |
| Operator | 12 | OLM packaging, subscription handling |
| Networking | 11 | Network policies, services, connectivity |
| Platform Alteration | 14 | Host access, kernel modifications |
| Preflight | 19 | Container and operator certification status |
| Performance | 7 | Resource limits, CPU pinning |
| Observability | 5 | Logging, monitoring, disruption budgets |
| Certification | 4 | Red Hat catalog verification |
| Manageability | 2 | Configuration management |

Each test maps to specific certification requirements, with clear remediation guidance when issues are found.

## Running Certsuite

Getting started is straightforward. You can run certsuite as a container or compile from source:

```bash
# Container-based execution
podman run --rm \
  -v ~/.kube/config:/home/certsuite/.kube/config:z \
  -v ./config:/config:z \
  -v ./results:/results:z \
  quay.io/redhat-best-practices-for-k8s/certsuite:latest \
  run --config-file=/config/tnf_config.yaml --output-dir=/results

# Or build from source
cd certsuite && make build
./certsuite run --config-file=tnf_config.yaml --output-dir=./results
```

The configuration file specifies which namespaces and workloads to test:

```yaml
targetNameSpaces:
  - name: my-cnf-namespace
podsUnderTestLabels:
  - "app.kubernetes.io/name: my-cnf"
operatorsUnderTestLabels:
  - "operators.coreos.com/my-operator.my-namespace: ''"
```

## Understanding Results

Certsuite produces a "claim file"—a JSON document containing pass/fail status for each test, along with configuration snapshots and detailed failure reasons. You can quickly identify issues:

```bash
# Show all failures
./certsuite claim show failures --claim-file results/claim.json

# Compare results between runs
./certsuite claim compare claim-v1.json claim-v2.json
```

Each failure includes a suggested remediation, making it clear what needs to change. For example:

```
Test: access-control-container-host-port
Status: FAILED
Reason: Container "my-app" in pod "my-app-7d8f9c" uses hostPort 8080
Remediation: Remove hostPort configuration from the container.
             Workloads should avoid accessing host resources.
```

## Scenario-Based Requirements

Not all tests apply to every workload. Certsuite defines four scenarios with different requirement levels:

- **Telco**: Strictest requirements (27 mandatory, 1 optional)
- **Non-Telco**: Standard enterprise workloads (43 mandatory, 28 optional)
- **Far-Edge**: Edge deployment considerations (8 mandatory, 1 optional)
- **Extended**: Additional telco-specific validations (10 mandatory, 3 optional)

Run tests for your specific scenario using label filters:

```bash
./certsuite run --config-file=config.yaml --label-filter="telco"
```

## Integrating with CI/CD

The real power of certsuite comes from running it continuously. Add it to your CI pipeline to catch issues on every commit:

```yaml
# GitHub Actions example
- name: Run Certsuite
  run: |
    ./certsuite run \
      --config-file=config/certsuite_config.yaml \
      --output-dir=./results \
      --label-filter="networking,access-control"

- name: Check for failures
  run: |
    ./certsuite check results --claim-file ./results/claim.json
```

This shift-left approach means certification issues surface during development, not after submission.

## Key Benefits

After helping partners adopt certsuite, I've seen consistent patterns in the value it provides:

1. **Faster certification cycles** - Issues found during development cost hours to fix; issues found during certification cost weeks.

2. **Reduced back-and-forth** - Clear remediation guidance means fewer questions and faster resolution.

3. **Confidence in submissions** - Partners who run certsuite continuously submit with confidence, knowing they've already addressed common blockers.

4. **Learning Red Hat best practices** - Each test teaches something about how to build production-ready workloads for OpenShift.

## Getting Started

The certsuite repository includes everything you need:

- [Full documentation](https://redhat-best-practices-for-k8s.github.io/certsuite/)
- [Test catalog](https://github.com/redhat-best-practices-for-k8s/certsuite/blob/main/CATALOG.md) with all 121 test cases
- [Sample configurations](https://github.com/redhat-best-practices-for-k8s/certsuite/tree/main/config)

For partners serious about Red Hat certification, running certsuite isn't optional—it's the fastest path to getting your CNF production-ready on OpenShift.

---

*[Red Hat Best Practices Test Suite for Kubernetes](https://github.com/redhat-best-practices-for-k8s/certsuite) | [Documentation](https://redhat-best-practices-for-k8s.github.io/certsuite/) | [Test Catalog](https://github.com/redhat-best-practices-for-k8s/certsuite/blob/main/CATALOG.md)*
