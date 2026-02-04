# Why Use OCT: Fresh Certification Data for Disconnected Environments

*Brandon Palm, Principal Software Engineer, Telco Security*

---

If you're running CNF certification tests in a disconnected environment, you've probably hit this frustrating scenario: your newly certified operator fails the `affiliated-certification` test because certsuite's embedded catalog doesn't know about it yet.

The fix is simple—use OCT—but the README didn't make that obvious. So I added a "Why Use OCT" section to fix that.

## The Problem

The [OCT (Offline Catalog Tool)](https://github.com/redhat-best-practices-for-k8s/oct) container is updated 4x daily with fresh certification data from Red Hat's catalog. Meanwhile, certsuite releases may be months apart, shipping with increasingly stale embedded catalogs.

When partners run certification tests in air-gapped environments:
1. Certsuite can't reach Red Hat's online catalog
2. It falls back to the embedded offline catalog
3. If that catalog is outdated, recently certified artifacts show as "not certified"
4. Tests fail despite the artifact being legitimately certified

## The Solution

I added a clear "Why Use OCT" section to the README that highlights:

- **Fresh certification data** — Updated 4x daily vs potentially stale certsuite releases
- **Disconnected environment support** — Works in air-gapped environments
- **Avoid false test failures** — Prevents `affiliated-certification` failures from outdated catalogs
- **Simple workflow** — Pull container, run in dump-only mode, point certsuite to the database
- **Complete coverage** — Operators, containers, and Helm charts

## The Change

```markdown
# Why Use OCT

- **Fresh certification data** — The OCT container is updated 4x daily,
  while certsuite releases may be months apart with increasingly stale
  embedded catalogs
- **Disconnected environment support** — Enables certification testing
  in air-gapped or offline environments where Red Hat's online catalog
  is unreachable
- **Avoid false test failures** — Prevents `affiliated-certification`
  test failures caused by newly certified artifacts missing from
  outdated embedded catalogs
- **Simple workflow** — Just pull the container, run it in dump-only
  mode, and point certsuite to the extracted database files
- **Complete coverage** — Includes certified operators, containers,
  and Helm charts from Red Hat's catalog
```

## Why This Matters

Good documentation reduces support burden. When engineers can quickly understand *why* they should use a tool, they're more likely to use it correctly. The existing "Motivation" section in the README provides deep technical context, but engineers scanning the README needed a quick summary of benefits upfront.

This small change should help partners:
- Understand OCT's value proposition in 30 seconds
- Know when to use OCT vs relying on embedded catalogs
- Avoid hours of debugging false certification failures

---

*[PR #405](https://github.com/redhat-best-practices-for-k8s/oct/pull/405) | [OCT Repository](https://github.com/redhat-best-practices-for-k8s/oct)*
