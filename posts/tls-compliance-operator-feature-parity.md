# Continuous TLS Compliance Monitoring for OpenShift: Closing the Gap with the Upstream Scanner

*Brandon Palm, Principal Software Engineer, Telco Partner Readiness*

---

When we audit TLS configurations across an OpenShift cluster, we typically reach for the [openshift/tls-scanner](https://github.com/openshift/tls-scanner) — a batch-mode Job that runs `testssl.sh` against every pod endpoint. It's thorough, but it's a snapshot. Between scans, endpoints can change, certificates can expire, and new services can appear with misconfigured TLS. The [tls-compliance-operator](https://github.com/sebrandon1/tls-compliance-operator) addresses this by running as a Kubernetes operator that continuously monitors TLS endpoints, but it was missing several features the upstream scanner provided. In a focused sprint, we closed 8 of those 10 feature gaps and shipped them as [v0.0.14](https://github.com/sebrandon1/tls-compliance-operator/releases/tag/v0.0.14).

## Why Continuous TLS Monitoring Matters

Telco partners deploying OpenShift at the edge and in Radio Access Network (RAN) environments operate under strict security compliance requirements. TLS misconfigurations — legacy protocol versions, weak cipher suites, missing forward secrecy — represent real compliance violations that can surface during audits. A batch scanner catches these at scan time. A continuous operator catches them as they happen, emitting Kubernetes events and Prometheus alerts before the next audit.

## What We Added

### Post-Quantum Cryptography Readiness Classification

With NIST's standardization of ML-KEM (Module-Lattice Key Encapsulation Mechanism), post-quantum cryptography is no longer theoretical. Go 1.24+ negotiates X25519MLKEM768 with servers that support it, and many public endpoints already do.

We added a `PQCReadiness` enum that classifies every endpoint:

| Level | Meaning |
|-------|---------|
| `PQCReady` | TLS 1.3 + ML-KEM negotiated |
| `TLS13Capable` | TLS 1.3 supported, ML-KEM not yet negotiated |
| `LegacyTLS` | TLS 1.2 or older only |
| `NoPQC` | No TLS detected |

This is paired with a `PQCCompliant` Kubernetes condition and a `tls_compliance_pqc_readiness` Prometheus metric, enabling alerting on endpoints that fall behind on quantum readiness.

### Health Probe Port Filtering

OpenShift pods frequently expose health check endpoints on the same ports used for application traffic. An HTTP liveness probe on port 8443 would trigger a false-positive "NoTLS" report. We now inspect each container's liveness, readiness, and startup probe definitions — including gRPC probes and named port resolution — and skip plaintext probe ports during discovery. HTTPS probes are still scanned.

On a CRC cluster with 69 pods, this correctly filtered 1 false positive and flagged 11 HTTPS probe ports for visibility.

### Forward Secrecy and Key Exchange Details

The operator already graded cipher suites A through F, with grades A and B implying ephemeral key exchange. We made this explicit with a `ForwardSecrecy` boolean and a `KeyExchangeTypes` map showing the exact algorithm (ECDHE, DHE, RSA, or TLS 1.3) per TLS version. Both are available as CRD fields, Prometheus metrics, and in CSV/JSON exports.

### SSLv3 Detection via Raw Socket Probe

Go's `crypto/tls` library removed SSLv3 support entirely — you cannot negotiate it even to detect it. We implemented a raw socket probe that crafts a minimal SSLv3 ClientHello at the byte level, sends it over TCP, and parses the ServerHello response. Endpoints supporting the cryptographically broken SSLv3 protocol are flagged as `NonCompliant`.

### IPv6 and Dual-Stack Support

Pod endpoint extraction now reads `pod.Status.PodIPs` (plural) for dual-stack clusters, creating TLS compliance reports for both IPv4 and IPv6 addresses. Custom resource names handle IPv6 addresses cleanly, and all event messages use `net.JoinHostPort` for correct bracket formatting.

## How It Compares

| Capability | tls-scanner | tls-compliance-operator |
|-----------|-------------|------------------------|
| Architecture | One-shot Job | Continuous operator |
| TLS version detection | SSLv2/v3 + TLS 1.0-1.3 | SSLv3 + TLS 1.0-1.3 |
| PQC readiness | ML-KEM detection + compliance | PQCReadiness enum + condition |
| Forward secrecy | Explicit status | ForwardSecrecy field + metric |
| Health probe filtering | PROBE_PORT status | Skip HTTP/TCP/gRPC, scan HTTPS |
| Prometheus metrics | None | 9 metrics (gauges, histograms, counters) |
| Kubernetes events | None | 8 event types |
| Periodic rescanning | Must re-run Job | Configurable interval (default 1h) |
| Certificate expiry | Via testssl.sh | DaysUntilExpiry + warning threshold |
| kubectl plugin | None | CSV, JSON, JUnit, summary |

The two tools are complementary. The scanner excels at deep one-off audits with `testssl.sh`'s comprehensive protocol analysis. The operator excels at ongoing compliance monitoring with native Kubernetes integration.

## Code Quality Process

Each feature went through an automated three-agent code review checking for reuse opportunities, quality issues, and efficiency concerns. This process caught several real bugs before they shipped:

- An **operator precedence bug** in the SSLv3 byte assembly where `len*2>>8` computed `len*(2>>8)` due to `>>` binding tighter than `*`
- A **data loss bug** where key exchange type extraction only examined the first cipher suite per TLS version
- **Triple duplication** of cipher name prefix checks across three functions, consolidated into a single `KeyExchangeType` source of truth

## Getting Started

Install with a single command:

```bash
kubectl apply -f https://github.com/sebrandon1/tls-compliance-operator/releases/download/v0.0.14/install.yaml
```

View reports:

```bash
kubectl get tlsreport
```

The operator begins scanning immediately, creating `TLSComplianceReport` custom resources for every TLS endpoint it discovers across Services, Ingresses, Routes, and Pods.

## What's Next

The remaining feature gaps — `/proc/net/tcp` port discovery, component filtering, and localhost-only detection — are low-priority items that would require elevated RBAC permissions. The current feature set covers the compliance monitoring needs we see in telco partner environments, and we'll continue to evolve it based on real-world usage.

---

*Resources:*
- [tls-compliance-operator](https://github.com/sebrandon1/tls-compliance-operator) — Kubernetes operator for continuous TLS compliance monitoring
- [v0.0.14 release](https://github.com/sebrandon1/tls-compliance-operator/releases/tag/v0.0.14) — Release with all features described above
- [openshift/tls-scanner](https://github.com/openshift/tls-scanner) — Upstream batch-mode TLS scanner for OpenShift
