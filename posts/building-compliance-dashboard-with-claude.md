# Building a Compliance Dashboard with Claude: Visualizing OpenShift Security at Scale

*Brandon Palm, Principal Software Engineer, Telco Security*

---

When you're tracking 178 compliance checks across multiple OpenShift versions, spreadsheets don't cut it. I needed a way to visualize remediation progress, link Jira tickets to specific checks, and give my team a single source of truth for our hardening work. What I didn't expect was how quickly Claude would help me build exactly that.

## The Problem: Compliance Data Chaos

The OpenShift Compliance Operator generates `ComplianceCheckResult` objects—lots of them. Each scan produces results for CIS benchmarks, NIST controls, and other security profiles. The data exists, but making sense of it requires:

- Tracking which checks are passing vs. failing
- Mapping failures to Jira tickets and PRs
- Understanding severity distribution at a glance
- Monitoring progress over time across OCP versions

I had scripts to collect the data. What I lacked was visibility.

## How Claude Changed the Approach

Instead of manually designing mockups and writing boilerplate, I described what I needed to Claude: a Jekyll-based dashboard hosted on GitHub Pages that could visualize compliance data from JSON exports.

Claude didn't just write code—it helped me think through the problem:

| Challenge | Claude's Contribution |
|-----------|----------------------|
| Data structure | Designed the JSON schema linking checks to tracking metadata |
| Visual hierarchy | Suggested severity-based grouping with color-coded badges |
| UX details | Added keyboard shortcuts, dark mode, expandable remediation details |
| Integration | Created the export script that pulls live data from clusters |

The iterative process was key. I'd describe a feature, Claude would implement it, and I'd refine based on what I saw. Within hours, not days, I had a working dashboard.

## What We Built

### Version Overview Cards

The main dashboard shows each OCP version at a glance:

```
OCP 4.21
Passing: 100  Failing: 78
🔵 5 In Progress  🟡 3 Pending  ⚪ 1 On Hold  🟢 2 Complete
Last scan: 2026-01-14
```

A coverage bar provides an instant visual of where we stand—red to green gradient based on pass rate.

### Remediation Tracking Tables

Each failing check links directly to:
- **Jira ticket** for the remediation work
- **GitHub PR** implementing the fix
- **Status badge** showing implementation progress
- **Expandable details** with the full remediation YAML

No more hunting through multiple systems. Everything lives in one view.

### Severity-Based Organization

Checks are grouped by severity (HIGH/MEDIUM/LOW) with distinct color coding. This lets the team prioritize effectively—we tackle high-severity items first, and the visual grouping makes that obvious.

### Dark Mode and Keyboard Navigation

Because engineers live in terminals, the dashboard supports:
- `j`/`k` for vim-style row navigation
- `/` to focus search
- `d` to toggle dark mode
- `1-4` to filter by tracking status

Small details, but they make the tool feel native to how we work.

## The Skills Advantage

What made this project different was using Claude's skills capability. I created a custom skill that understands my compliance workflow—it knows about the repository structure, the data formats, and the relationships between components.

When I ask Claude to update the dashboard, it doesn't start from scratch. It understands:
- Where the tracking data lives (`docs/_data/tracking.json`)
- How version pages are structured (`docs/versions/`)
- The CSS patterns we've established
- How to export fresh data from a connected cluster

This context persistence is the difference between a tool that helps occasionally and one that feels like a team member who remembers previous conversations.

## Technical Implementation

The stack is intentionally simple:

```
OpenShift Cluster
    ↓ (oc get compliancecheckresults)
export-compliance-data.sh
    ↓
docs/_data/ocp-4_21.json
    ↓
Jekyll + Liquid Templates
    ↓
GitHub Pages (auto-deployed on push)
```

No backend servers. No databases. Just JSON files processed by Jekyll and served as static HTML. Updates happen by running a make target and pushing to main.

The CSS handles the heavy lifting for visual design—1700+ lines covering responsive layouts, theme switching, and accessibility. Claude helped generate this iteratively, with each feature building on established patterns.

## Results

| Metric | Before | After |
|--------|--------|-------|
| Time to find check status | 5-10 minutes (multiple systems) | Instant |
| Tracking accuracy | Spreadsheet drift | Single source of truth |
| Team visibility | Individual knowledge | Shared dashboard |
| Update frequency | Manual, sporadic | Automated on each scan |

The dashboard now tracks 17 remediation groups across OCP 4.21, with clear visibility into what's done, what's in progress, and what's blocked.

## Lessons Learned

**Start with data, not design.** Claude helped most when I could describe the data I had and the questions I needed answered. The visual design emerged from those requirements.

**Iterate in public.** Pushing early versions to GitHub Pages meant the team could see progress and provide feedback. Claude made iterating on that feedback fast enough to be practical.

**Skills create continuity.** The ability to teach Claude about my specific project structure turned one-off assistance into ongoing collaboration. Each session builds on the last.

**Simple stacks scale.** Jekyll and GitHub Pages handle this use case without infrastructure overhead. Claude helped me resist the temptation to over-engineer.

## Try It Yourself

The dashboard and all supporting scripts are open source:

- **Live Dashboard**: [sebrandon1.github.io/compliance-scripts](https://sebrandon1.github.io/compliance-scripts/)
- **Repository**: [github.com/sebrandon1/compliance-scripts](https://github.com/sebrandon1/compliance-scripts)

The `docs/` directory contains the full Jekyll implementation. The `core/` directory has the scripts for collecting compliance data from your own clusters.

---

*Claude Code helped build this dashboard iteratively over multiple sessions. The skills feature maintained context about the project structure, enabling rapid iteration on features without re-explaining the codebase each time.*
