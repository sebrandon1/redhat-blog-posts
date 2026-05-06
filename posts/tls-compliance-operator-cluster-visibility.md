# See What Your Cluster Is Really Serving: TLS Visibility with the tls-compliance-operator

*Brandon Palm, Principal Software Engineer, Telco*

---

You know your apps are running. But do you know what TLS versions they're negotiating? What cipher suites they're using? When their certificates expire? Whether they're ready for post-quantum cryptography (PQC)?

Most teams don't. Not because they don't care, but because there's no easy way to get a cluster-wide view of TLS health. The [tls-compliance-operator](https://github.com/sebrandon1/tls-compliance-operator) changes that. One deploy command, zero configuration, and you get immediate visibility into every TLS endpoint in your cluster.

## Overview

The tls-compliance-operator is a Kubernetes operator that auto-discovers every TLS endpoint in your cluster (Services, Ingresses, OpenShift Routes, and Pods) and creates a `TLSComplianceReport` custom resource for each one. It checks TLS version support, grades cipher suites, tracks certificate health, evaluates compliance against [OpenShift TLS security profiles](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/security_and_compliance/tls-security-profiles), and detects post-quantum cryptography readiness.

Deploy it with a single command:

```
kubectl apply -f https://github.com/sebrandon1/tls-compliance-operator/releases/download/v1.0.3/install.yaml
```

Within minutes, the operator discovers all TLS-speaking endpoints and starts creating reports.

## What You See

Here's what `kubectl get tlsreport` shows on a live OpenShift 4.22 cluster with a mix of platform services and demo workloads:

```
$ kubectl get tlsreport
NAME                                             HOST                                       PORT  SOURCE  COMPLIANCE  GRADE  FS    TLS1.3  TLS1.2  PQC           CERTEXPIRY
router-internal-default-openshift-ingress-...    router-internal-default.openshift-ingress   443   Service Compliant   A      true  true    true    PQCReady      703
console-openshift-console-443-...                console.openshift-console                   443   Service Compliant   A      true  true    true    PQCReady      703
web-tls13-tls-blog-demo-443-...                  web-tls13.tls-blog-demo                     443   Service Compliant   A      true  true    false   TLS13Capable  364
web-tls12-tls-blog-demo-443-...                  web-tls12.tls-blog-demo                     443   Service Compliant   A      true  false   true    LegacyTLS     364
web-expired-tls-blog-demo-443-...                web-expired.tls-blog-demo                   443   Service Compliant   A      true  true    true    TLS13Capable  -489
web-http-tls-blog-demo-443-...                   web-http.tls-blog-demo                      443   Service NoTLS              false false   false
```

Every column answers a question at a glance. Is the endpoint compliant? What grade are its ciphers? Does it support forward secrecy? Is TLS 1.3 negotiating? Is the certificate about to expire? Is it ready for post-quantum crypto?

The operator auto-discovered the OpenShift platform services (router, console, API server) alongside our demo workloads. Zero configuration.

## Technical Details

### Deep Inspection

`kubectl describe` gives you the full picture. Here's a TLS 1.3-only endpoint:

```
$ kubectl describe tlsreport web-tls13-tls-blog-demo-443-23323ee0
Status:
  Compliance Status:     Compliant
  Overall Cipher Grade:  A
  Forward Secrecy:       true
  Tls Versions:
    tls12:  false
    tls13:  true
  Cipher Suites:
    TLS 1.3:
      TLS_AES_128_GCM_SHA256
  Key Exchange Types:
    TLS 1.3:  TLS13
  Negotiated Curves:
    TLS 1.3:  X25519
  Certificate Info:
    Issuer:            CN=tls-demo
    Days Until Expiry: 364
    Not After:         2027-05-06T20:41:44Z
  Conditions:
    Type: TLSCompliant      Status: True   Reason: Compliant
    Type: CertificateValid  Status: True   Reason: Valid
    Type: PQCCompliant      Status: False  Reason: TLS13Capable
```

TLS version support, the exact cipher suite negotiated, key exchange type, curve used, certificate details with expiry countdown, and structured conditions you can query programmatically. All in one place.

### Catching Certificate Problems

The operator fires Kubernetes events for expired and expiring certificates. On our test cluster, it caught real issues: `cert-manager-webhook` expiring in 5 days and `kubelet` expiring in 19 days:

```
$ kubectl get events --field-selector reason=CertificateExpiring
Warning  CertificateExpiring  cert-manager-webhook-cert-manager-443-...  TLS certificate expires in 5 days
Warning  CertificateExpiring  kubelet-kube-system-10250-...              TLS certificate expires in 19 days
Warning  CertificateExpiring  web-expiring-tls-blog-demo-443-...         TLS certificate expires in 6 days
```

For expired certificates, the operator provides clear status:

```
$ kubectl describe tlsreport web-expired-tls-blog-demo-443-d38a652c
  Certificate Info:
    Days Until Expiry:  -489
    Is Expired:         true
    Issuer:             CN=expired-demo
    Not After:          2025-01-02T00:00:00Z
  Conditions:
    Type: CertificateValid  Status: False  Reason: Expired
Events:
  Warning  CertificateExpired  8m  tls-compliance-controller  TLS certificate has expired for web-expired.tls-blog-demo:443
```

These are real findings on a running cluster, the kind of visibility you don't have until something breaks.

The operator also detects non-TLS endpoints (showing `NoTLS`) and ports that aren't responding (`Closed`, `Timeout`), helping you understand what's actually listening across your cluster.

### Post-Quantum Readiness

With [OpenShift driving post-quantum cryptography adoption](https://www.redhat.com/en/blog/road-to-quantum-safe-cryptography-red-hat-openshift) and [PQC support landing in the OCP 4.20 control plane](https://www.redhat.com/en/blog/deeper-look-post-quantum-cryptography-support-red-hat-openshift-420-control-plane), knowing where you stand today is critical. The operator already detects PQC key exchange algorithms.

Here's what a PQC-ready endpoint looks like on our OCP 4.22 cluster:

```
  Negotiated Curves:
    TLS 1.2:  X25519
    TLS 1.3:  X25519MLKEM768
  Pqc Readiness:  PQCReady
  Quantum Ready:  true
```

`X25519MLKEM768` is the hybrid post-quantum key exchange, combining classical X25519 with [ML-KEM](https://www.redhat.com/en/topics/security/post-quantum-cryptography), the NIST-standardized lattice-based algorithm. The operator categorizes every endpoint into one of four PQC readiness levels:

| PQC Status | Meaning |
|---|---|
| **PQCReady** | Already negotiating post-quantum key exchange |
| **TLS13Capable** | Supports TLS 1.3 (upgrade path exists) but not yet negotiating PQC |
| **LegacyTLS** | Only supports TLS 1.2, no path to PQC without a protocol upgrade |
| **NoPQC** | No TLS at all |

On our test cluster, every OCP platform service is already PQCReady. Our TLS 1.3-only demo service is TLS13Capable: it speaks 1.3 but the server's crypto library hasn't enabled ML-KEM yet. And our TLS 1.2-only service is LegacyTLS, needing a TLS 1.3 upgrade before PQC is even possible.

This is exactly the inventory you need for a [PQC migration plan](https://www.redhat.com/en/blog/if-how-year-post-quantum-reality). As [RHEL 10.1 makes PQC generally available](https://www.redhat.com/en/blog/whats-new-post-quantum-cryptography-rhel-101) across Go, OpenSSL, GnuTLS, and NSS, the gap between "platform ready" and "workload ready" is the one you need to close.

## Results

| Metric | Value |
|---|---|
| Time to deploy | One `kubectl apply` command |
| Configuration needed | None |
| Resource types scanned | Services, Ingresses, Routes, Pods |
| Data points per endpoint | TLS versions, ciphers, grades, forward secrecy, cert health, PQC readiness, profile compliance |
| OCP 4.22 platform PQC status | 100% PQCReady (X25519MLKEM768) |
| Real cert issues found | cert-manager-webhook (5 days), kubelet (19 days) |
| Prometheus metrics | 8 metrics covering compliance, cert expiry, PQC, scan performance |

### Prometheus Integration

For teams running Prometheus, the operator exposes metrics you can alert on:

```promql
# What percentage of endpoints are compliant?
sum(tls_compliance_endpoints_total{status="Compliant"}) / sum(tls_compliance_endpoints_total) * 100

# Which certificates expire within 7 days?
tls_compliance_certificate_expiry_days < 7

# Which endpoints are PQC-ready?
tls_compliance_pqc_readiness{readiness="PQCReady"} == 1
```

Pre-built alerting rules and a Grafana dashboard JSON are included in the repo.

## Tips

- **Start with the defaults.** The operator works out of the box. Tune scan intervals and timeouts later if needed.
- **Use events for alerting.** Kubernetes events for `CertificateExpiring` and `CertificateExpired` integrate with existing monitoring without extra setup.
- **Plan PQC migration early.** Use the PQC readiness column to inventory which workloads need TLS 1.3 upgrades before they can negotiate post-quantum key exchange.
- **Check your TLS 1.2-only services.** The `LegacyTLS` PQC status highlights workloads that have no path to PQC without a protocol upgrade.
- **Filter by namespace.** Use `--include-namespaces` or `--exclude-namespaces` flags to scope scanning to specific areas of your cluster.

## Closing

The tls-compliance-operator gives you immediate, continuous visibility into the TLS health of your entire cluster. From cipher grades and certificate expiry to post-quantum readiness, it surfaces the information you need to maintain compliance and plan for the future. All from a single deploy command with zero configuration.

---

*[tls-compliance-operator on GitHub](https://github.com/sebrandon1/tls-compliance-operator) | [Configuring TLS security profiles in OpenShift](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/security_and_compliance/tls-security-profiles) | [The road to quantum-safe cryptography in Red Hat OpenShift](https://www.redhat.com/en/blog/road-to-quantum-safe-cryptography-red-hat-openshift) | [PQC support in the OpenShift 4.20 control plane](https://www.redhat.com/en/blog/deeper-look-post-quantum-cryptography-support-red-hat-openshift-420-control-plane) | [What is post-quantum cryptography?](https://www.redhat.com/en/topics/security/post-quantum-cryptography) | [What's new in PQC in RHEL 10.1](https://www.redhat.com/en/blog/whats-new-post-quantum-cryptography-rhel-101)*
