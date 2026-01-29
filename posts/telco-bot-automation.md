# Managing 650+ PRs Across 878 Repositories with Claude CLI

*Brandon Palm, Principal Software Engineer, Telco Security Team*

---

The Telco Security team tracks dependency updates across 5 Red Hat organizations containing 878 repositories. Go versions, deprecated packages like `io/ioutil` and `golang/mock`, and security-critical libraries like `golang.org/x/crypto` all need monitoring. Manual tracking at this scale? Impossible.

## What I Built

Telco-bot is an automation toolset with 6 scanners that run daily or weekly via GitHub Actions:

- **Go version checker** - finds outdated Go versions
- **golangci-lint checker** - tracks linter version drift
- **gomock scanner** - finds deprecated `golang/mock` usage
- **io/ioutil scanner** - flags deprecated stdlib usage
- **x/crypto scanner** - monitors security library versions
- **UBI image scanner** - checks container base images

Each scanner creates tracking issues in affected repos and updates a central tracking issue aggregating all findings. Smart caching skips forks (284), abandoned repos (57), and non-Go repos (241) to focus on what matters.

## How Claude CLI Accelerated Development

The x/crypto scanner went through 8+ iterations to get the issue management logic right. Here's a real debugging session:

```
Me: "The xcrypto script is incorrectly closing GitHub issues
    that should stay open. Repos that still have outdated
    x/crypto are getting their tracking issues closed."

Claude: [Analyzed xcrypto-lookup.sh, identified the issue]
    "The problem is in the manage_repo_issue function. When
    checking if a repo is a fork, it's using the fork cache
    but the cache lookup is matching partial strings. A repo
    like 'openshift/kubernetes' is matching against
    'openshift/kubernetes-client-go' in the cache."
```

That bug caused PRs #83 and #85 - fixing the fork detection logic that was incorrectly identifying repos as forks.

Another key win was designing the shared caching system. Originally each scanner had its own cache files. Claude helped consolidate everything into `scripts/caches/` with shared files for forks, abandoned repos, and no-gomod repos. That became PR #72.

Claude also helped create the GitHub Actions workflows that automate all of this: daily runs for Go version, golangci-lint, gomock, and io/ioutil scanning, plus weekly x/crypto scans and automated cache updates. The x/crypto scanner even posts a weekly Slack summary reporting the number of outdated repos with a link to the tracking issue.

## The Results

| Metric | Value |
|--------|-------|
| Total PRs created | 650+ |
| Merge rate | ~95% |
| Organizations covered | 5 |
| Repositories scanned | 878 |
| Active tracking issues | 5 |

Timeline: All built since November 2024.

See the tracking issues:
- [Tracking Out of Date Golang Versions](https://github.com/redhat-best-practices-for-k8s/telco-bot/issues/39)
- [Tracking Deprecated golang/mock Usage](https://github.com/redhat-best-practices-for-k8s/telco-bot/issues/45)
- [Tracking Outdated GolangCI-Lint Versions](https://github.com/redhat-best-practices-for-k8s/telco-bot/issues/49)
- [Tracking Deprecated io/ioutil Package Usage](https://github.com/redhat-best-practices-for-k8s/telco-bot/issues/52)
- [Tracking golang.org/x/crypto Direct Usage](https://github.com/redhat-best-practices-for-k8s/telco-bot/issues/59)

## Tips for Getting Started

1. **Start with a specific problem** - not "use AI for something"
2. **Iterative debugging is a superpower** - paste error output, get targeted fixes
3. **Consistency across files** - Claude excels at applying the same pattern to many files
4. **Keep documentation in sync** - updating READMEs alongside code changes

## Closing

This automation directly supports the Telco Security mission. Less time chasing dependency updates means more time for actual security work. If you're facing similar scale challenges, the Claude CLI is worth exploring.

---

*The telco-bot repository is at [github.com/redhat-best-practices-for-k8s/telco-bot](https://github.com/redhat-best-practices-for-k8s/telco-bot)*
