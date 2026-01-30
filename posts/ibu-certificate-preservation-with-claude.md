# Discovering IBU Certificate Preservation with Claude: AI-Assisted Validation Testing

*Brandon Palm, Principal Software Engineer, Telco Security Team*

---

We've been telling customers that cert-manager certificates are lost during Image-Based Upgrades. Turns out that's only half the story. Using Claude CLI, I built a test framework in a single session that proved certificates **can** be preserved—if you know how to label them.

## The Problem

Image-Based Upgrade (IBU) is how we upgrade Single Node OpenShift clusters in telco RAN deployments. A documented limitation is that cert-manager certificates get regenerated after the upgrade because private keys aren't backed up with the seed image.

But customers kept asking: *"Is there any way to preserve specific certificates?"*

I had a hypothesis that the Lifecycle Agent's `lca.openshift.io/apply-label` annotation might work for cert-manager resources. Rather than spending days setting up test environments manually, I paired with Claude to find out.

## Building the Test Framework with Claude

I described what I wanted to validate, and Claude helped me build a complete test framework in about an hour:

**What Claude Generated:**
- Shell scripts for two test scenarios (loss vs. preservation)
- YAML manifests for Velero Backup/Restore CRs
- A labeling script that applies LCA-style labels to cert-manager resources
- Validation logic that compares SHA256 checksums before and after
- Makefile targets for easy execution
- Documentation explaining how the simulation maps to real IBU

The key insight Claude brought was structuring the comparison—capture cryptographic checksums of TLS secrets before the simulated upgrade, then compare after restore. Different checksums = regenerated. Same checksums = preserved.

```bash
# The test is now a one-liner
make test-ibu-preserved
```

## What We Discovered

### Scenario 1: Default Behavior (No Labels)

```
Before: 1a4cae83857ec4b2...
After:  635d029dc2c6bb71...
Status: CHANGED
```

Certificates regenerated. This matches the documented behavior.

### Scenario 2: With LCA Labels

```
Before: 1a4cae83857ec4b2...
After:  1a4cae83857ec4b2...
Status: UNCHANGED
```

**Certificates preserved.** Same private keys, same certificate data.

## How It Works

The Lifecycle Agent uses OADP (Velero) under the hood. When you label resources with `lca.openshift.io/backup`, and the Backup CR uses a matching `labelSelector`, Velero backs up the actual secret data—including private keys.

```bash
# Label the certificate and its secret
oc label certificate my-cert -n my-namespace \
  lca.openshift.io/backup=my-backup-name

oc label secret my-cert-tls -n my-namespace \
  lca.openshift.io/backup=my-backup-name
```

The Backup CR then targets only labeled resources:

```yaml
spec:
  labelSelector:
    matchLabels:
      lca.openshift.io/backup: my-backup-name
```

On restore, the original certificate data comes back intact. cert-manager sees a valid certificate and doesn't regenerate.

## How Close Is This to Real IBU?

| Component | Real IBU | Our Simulation |
|-----------|----------|----------------|
| Backup technology | OADP/Velero | OADP/Velero |
| Label mechanism | `lca.openshift.io/backup` | Same |
| Backup/Restore CRs | Created by LCA | Identical CRs |
| cert-manager behavior | Reconciles after restore | Same |

The only differences are orchestration (LCA vs. manual) and the absence of an actual OS reboot—neither affects certificate behavior. The simulation uses the exact same Velero mechanism that LCA uses.

## When to Use Each Approach

| Certificate Type | Recommendation |
|------------------|----------------|
| Short-lived, auto-renewed | Allow regeneration (default) |
| Long-lived CA certificates | Preserve with LCA labels |
| Shared with external systems | Preserve with LCA labels |
| Where regeneration breaks things | Preserve with LCA labels |

## The AI Advantage

What would have taken a day or more of manual scripting took about an hour with Claude:

1. **Rapid prototyping** - Described the test scenarios, got working scripts
2. **Edge case handling** - Claude added proper error handling and prerequisites checks
3. **Documentation** - Generated comprehensive docs explaining the simulation validity
4. **Iteration** - Easy to refine and add Scenario 2 after Scenario 1 worked

The framework is now reusable. Anyone can run `make test-ibu-both` to validate both scenarios on their cluster.

## Try It Yourself

```bash
git clone https://github.com/sebrandon1/cert-manager-scripts
cd cert-manager-scripts

# Install prerequisites (MinIO + OADP)
make install-ibu-prereqs

# Run both scenarios
make test-ibu-both
```

Full report with detailed findings: [IBU Certificate Validation Report](https://gist.github.com/sebrandon1/71f33b35aea2aa4cf9edda855201c8fc)

## Conclusion

The assumption that cert-manager certificates are always lost during IBU isn't quite right. With the LCA labeling mechanism, you can explicitly preserve certificates that need to survive the upgrade. And with AI-assisted development, validating this took a fraction of the time it would have otherwise.

---

*Resources:*
- [cert-manager-scripts repository](https://github.com/sebrandon1/cert-manager-scripts)
- [Full validation report](https://gist.github.com/sebrandon1/71f33b35aea2aa4cf9edda855201c8fc)
- [IBU documentation](https://docs.openshift.com/container-platform/latest/edge_computing/image_based_upgrade/cnf-about-image-based-upgrade.html)
- [LCA apply-label reference](https://docs.openshift.com/container-platform/4.17/edge_computing/image_based_upgrade/preparing_for_image_based_upgrade/cnf-image-based-upgrade-prep-resources.html)
