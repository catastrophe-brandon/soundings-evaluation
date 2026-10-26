# Soundings evaluation for the Platform Experience framework team

**Status: pilot complete — five Soundings runs against real insights-chrome (and one cross-repo) PRs. Final verdict: would not recommend adoption.** See "Final evaluation verdict" below for the conclusion and its basis; the sections that follow document how we got there.

## Provisional assessment (superseded — see "Final evaluation verdict")

*This section captured our thinking before the pilot ran. It's kept for context; the "Final evaluation verdict" section below reflects what we actually found and is the current recommendation.*

Soundings looks like a promising *review assistant* for coordinated releases. It analyzes one or more GitHub or GitLab compare ranges together and reports a computed `release`, `manual review`, or `no-go` verdict with named concerns and actions. Its author describes cross-repository risk detection, which could matter when a framework, consuming modules, and deployment configuration move together. The risk areas it says it examines—API contracts, configuration, security, and compound changes—also overlap with likely framework release concerns. These are fit hypotheses, not results from a team pilot.

There are two adoption constraints to resolve first:

1. The published tool is a **Claude Code plugin**. Its author says a Codex port is being explored, so it is not yet a drop-in part of a Codex workflow.
2. The documented run needs Claude Code, Go, and GitHub/GitLab credentials. The tool fetches repository diffs, comments, and documentation into a temporary local workspace before analysis. The author describes isolation, credential redaction, and cleanup safeguards; this desk review has not independently verified their implementation.

There is also a significant evidence gap: the documented workflow does not describe running tests, ingesting CI results, or measuring changed-code test coverage. The analyst may inspect test files when they support a coverage claim, but that is not equivalent to verifying that changed behavior is exercised or that the tests passed. For release readiness, this should be treated as a limitation unless a pilot or source inspection demonstrates an additional coverage integration.

**Original recommendation (pre-pilot):** run a small, human-reviewed shadow pilot if the team can use Claude Code for the trial and approve its access to selected repositories. Do not use its verdict as an automatic release gate until the pilot demonstrates useful, evidence-backed findings and the team has reviewed the implementation and data-handling behavior. *(That pilot has now run; see the final verdict below.)*

## Pilot design

Use three to five completed or imminent release comparisons, covering multiple repositories where possible. Include known outcomes so reviewers can judge both missed risks and false alarms:

- A release with a known cross-repository compatibility or rollout issue.
- A release that changes shared framework behavior, routing, authentication, or module loading.
- A benign release, such as documentation or isolated UI changes, to measure noisy findings.
- If available, a configuration or deployment change paired with the code that depends on it.

Run Soundings in shadow mode before the normal release decision. Keep the existing review and approval process authoritative. Have two team reviewers independently label each finding as valid, invalid, or uncertain, and record whether it changed the review discussion.

For each run, capture:

| Measure | What to record |
| --- | --- |
| Known-risk recall | Which pre-identified material risks did it surface? |
| Finding precision | Of the material findings, how many reviewers confirmed? |
| Evidence quality | Does each finding point to the relevant change and explain a plausible failure mode? |
| Test evidence | Does the report identify whether changed behavior has relevant tests, and is that supported by actual CI/coverage data or only static inspection? |
| Cross-repository value | Did it identify an interaction that per-repository review missed? |
| Noise | Were blockers or high-severity findings unsupported or irrelevant? |
| Workflow cost | Setup friction, run time, reviewer time, and report usefulness |
| Operational fit | Can it analyze the team’s GitHub/GitLab hosts and coordinated release inputs? |

Agree on numerical acceptance thresholds with the team before running the pilot. A useful pilot should catch the seeded critical risks, provide verifiable evidence for blocking concerns, and add enough cross-repository or review value to justify its setup and review cost. A missed critical risk or unsupported blocker should trigger investigation before any further rollout.

## Security and operational checks before broader use

- Pin the plugin to a reviewed version and inspect its source and dependency/update path.
- Confirm which repositories, comments, documentation links, and credentials it accesses during the run.
- Check separately whether and how it receives CI pass/fail results and changed-code coverage; do not assume the semantic review provides either.
- Verify temporary-data cleanup, report handling, and credential redaction for the team’s actual GitHub/GitLab setup.
- Treat prompt-injection resistance as a claim to validate, not as a substitute for normal review. The author notes that hostile content can still bias the analysis wording or severity.
- Keep the report advisory and require an accountable human to make the release decision.

## Basis and limits

This assessment uses the author’s [Soundings overview and security description](https://gwenneg.com/2026/09/03/take-soundings-before-you-ship.html). The author also points Red Hat associates shipping through app-interface to a separate [soundings-app-interface adapter](https://github.com/gwenneg/soundings-app-interface), which may be relevant to the team’s deployment workflow. The adapter’s suitability for this team has not been checked.

The claimed fit with framework and consuming-module releases is an inference from Soundings’ stated multi-repository analysis and the team context, not a measured result. PRs #3753 and #3631 are manually inspected cases; no full source-code security review of Soundings was performed.

## Case 1: insights-chrome PR #3753

[PR #3753](https://github.com/RedHatInsights/insights-chrome/pull/3753) merged on 2026-10-22. The comparison was pinned to the PR’s recorded base `a613d12506323496e64d56802f3a04be15546199` and head `e4bc717ea1b07d3c53a8e003591b33061ceeb23c` so later `master` commits are excluded.

Manual change summary: the Lightwell-only `pf-6-next` alias moves from PatternFly 6.7 prerelease.7 to stable 6.7.0, with the lockfile updated. `usePf5Styles` now checks the current router path and prevents PF5 stylesheet injection on Lightwell routes even when the feature flag or local override enables PF5. Cypress coverage adds initial-route cases for both the feature-flag and local-override paths. The PR reports 22 passing Jest tests, 13 passing Cypress component tests, and targeted lint/format checks.

Candidate evaluation signals (manual baseline, not Soundings findings): does the report understand the interaction between the global hook, route changes, and local override; does it recognize the lockfile change as a stable dependency upgrade; and does it distinguish the reported test runs from comprehensive behavior coverage? A reviewer may probe SPA navigation into and out of Lightwell: the added cases initialize at each route but do not appear to exercise navigation between routes. Treat this as a question for validation, not a confirmed defect.

An attempt to run Soundings in this environment did not reach its fetch stage. Its helper MCP tools were unavailable, and an explicit session-only connection was blocked with `MCP server blocked by enterprise policy: plugin_soundings_helper`; that attempt returned no verdict. The plugin and marketplace are installed in Claude’s user configuration, but the current environment’s policy prevents running the helper.

## Case 2: insights-chrome PR #3631 — app breadcrumbs

[PR #3631](https://github.com/RedHatInsights/insights-chrome/pull/3631) merged on 2026-08-17. Its comparison range is base `f274f464c05597358e98860145b7375611c8015c` to head `e8e0777862d0853b7392eb9d1289189d78e9ee6d`.

The PR added `useBreadcrumbs` and `useReplaceBreadcrumbs` hooks for remote applications and exposed them through Module Federation. The hooks wrote to Jotai atoms that Chrome also imported. The follow-up [PR #3671](https://github.com/RedHatInsights/insights-chrome/pull/3671) documents the regression: webpack created separate module instances for the remote hook and Chrome, so app breadcrumb updates were written to a different atom than the one Chrome rendered. #3671 replaced the atoms with a shared store consumed through the federated module path.

Candidate evaluation signals (manual baseline, not Soundings findings): does the analysis trace state ownership across the host and federated modules; identify that unit tests of the hooks, atoms, and rendered breadcrumb merge do not demonstrate that both sides share one runtime instance; and connect that risk to the later fix? PR #3631 added extensive tests in those areas, while its changed test files do not appear to exercise the built Module Federation boundary. PR #3659 was an intervening breadcrumb follow-up; #3671 is the known fix for the duplicate-store regression. This case has not yet been run through Soundings.

## Pilot results to date: regression recall

The MCP block described in Case 1 did not persist: Soundings has since run successfully in this environment, producing full reports for two additional insights-chrome comparisons (plus the Case 1 report, `soundings-pr3753.md`, which did complete despite the block noted above). Each run used the PR's recorded base/head commits so later `master` commits are excluded.

| PR | Role | Known regression | Soundings verdict | Caught? |
| --- | --- | --- | --- | --- |
| [#3631](https://github.com/RedHatInsights/insights-chrome/pull/3631) | Introduced the regression | Yes — `useBreadcrumbs`/`useReplaceBreadcrumbs` wrote to Jotai atoms that webpack instantiated separately for the federated hook vs. Chrome's own import, so app breadcrumb updates never reached the atom Chrome rendered from | ⚠️ Manual review — 1 high concern: consuming the new hooks any way other than via Module Federation's `useRemoteHook` writes to "a disconnected Jotai store" and breadcrumbs "silently never appear" | **Partial.** The top-severity finding names the exact symptom and consequence of the real bug (disconnected store → breadcrumbs silently don't update), quoting the PR's own design-doc warning almost verbatim. But it attributes the cause to *developer misuse* (importing directly instead of via `useRemoteHook`) — exactly what the PR's own docs already called out — rather than independently reasoning that webpack's chunk-splitting could duplicate the module even for correct, by-the-book federated usage, which is what actually shipped and required #3671 to fix. Right symptom, incomplete root cause. |
| [#3671](https://github.com/RedHatInsights/insights-chrome/pull/3671) | Fixed #3631's regression (replaced the atoms with a `globalThis`-anchored shared store) | No confirmed regression shipped from this PR itself (checked: only one additive follow-up touched the same files, `#3676`; the flagged silent `.catch(() => {})` is still unaddressed in master but has no attributed incident) | ⚠️ Manual review — 2 high concerns: the webpack duplicate-module-factory claim backing the fix was unverified in the diff, and a silent `.catch(() => {})` on app-switch | N/A — no regression existed to catch, but the flagged webpack-duplication risk was independently validated as real by [#3722](https://github.com/RedHatInsights/insights-chrome/pull/3722), which hit the same bug for the dark-mode store and reused #3671's fix pattern |
| [#3702](https://github.com/RedHatInsights/insights-chrome/pull/3702) | Unrelated feature PR (Unleash outage handling) with its own regression | Yes — the `useMemo` building the Unleash client gained `isPreview` (and identity fields) as dependencies, so the client is recreated on preview toggle; `@unleash/proxy-client-react`'s `FlagProvider` keeps the client it first mounted with, so `useFlag()` consumers read a stale, detached client after recreation | ✅ Recommended for release — no blocking concerns | **No.** The analyst reasoned about whether *Chrome's own code* reacted correctly to a new client (and called that well-tested) but never checked whether the *library's* `FlagProvider` propagates a replaced client to its own context — the actual failure mode. It raised two adjacent-but-different medium concerns (an undocumented `client.emit('update')` call, and GlobalFilter's fail-open behavior change) that didn't overlap with the real bug |
| [#3753](https://github.com/RedHatInsights/insights-chrome/pull/3753) | Case 1 baseline | No confirmed known regression; manual review only raised an untested-SPA-navigation hypothesis for validation | ⚠️ Manual review — 1 high concern: `usePf5Styles` now calls `useLocation()` unconditionally, re-rendering the shared Chrome root on every route change for all tenants | N/A — no confirmed regression to score against |

A fix for #3702's regression was later identified: [#3747](https://github.com/RedHatInsights/insights-chrome/pull/3747), "Fix Preview toggle leaving React flag consumers on stale client" — title and description match the regression exactly, confirming it as a real shipped-and-fixed bug rather than a hypothesis.

### Cross-repo case: insights-chrome × learning-resources (quickstarts migration)

The four cases above are all single-repo. To test Soundings' claimed multi-repo compound-risk detection, we ran it against a genuinely coordinated two-repo change: [learning-resources #363](https://github.com/RedHatInsights/learning-resources/pull/363) (merged 2026-09-22) and [insights-chrome #3674](https://github.com/RedHatInsights/insights-chrome/pull/3674) (open as of 2026-10-26), which move the Quickstarts/Help Topics runtime out of Chrome into a federated learning-resources module. Both PR descriptions explicitly document a merge-order dependency ("Merge this PR after..." / "wait until stage `fed-mods.json` lists `./QuickstartsRuntime` before that Chrome PR").

No known regression exists for this pair yet since #3674 hasn't merged, so this is a forward-looking test of added value, not a recall test. Two operational notes and one result:

- **Fetch hiccup:** #3674 is from a personal fork (`hossam-farid/insights-chrome`), not a branch on the upstream repo. The first fetch attempt, using the upstream repo's compare URL with the PR's head SHA, failed outright (`fatal: Invalid symmetric difference expression`) because that commit doesn't exist in the upstream clone at all — forks are separate repositories. Re-pointing the compare URL at the fork (`https://github.com/hossam-farid/insights-chrome/compare/<base>...<head>`) fixed it. This is a real limitation for shadow-mode use: **Soundings cannot be pointed at an open PR by its upstream URL if the PR is from a fork** — the caller must know and supply the fork's URL.
- **Tool-restriction friction:** the `soundings:analyze` skill disallows shell/network tools for its own orchestrating turn (by design, to prevent untrusted fetched content from driving tool use) — but this also blocks simple `gh` lookups (like finding a PR's fork origin) needed *before* a correct fetch call can be made. Resolving the fork required exiting the skill flow, running `gh pr view` separately, then re-invoking.
- **Result:** once fetched correctly, the analysis's top concern independently derived the real cross-repo coordination risk from the actual webpack/module-federation config in both repos (insights-chrome flips a shared module to `eager: true` with an inline comment about remotes relying on it; learning-resources' federated module depends on that), rather than just repeating the PR authors' own "merge in this order" notes. It also found a plausible, specific bug on its own (a missing null guard before POSTing quickstart progress, likely sending `accountId: NaN`). Verdict: manual review, 2 high concerns. This is the strongest evidence so far that giving Soundings multiple repos together adds real value over analyzing either alone — though it remains unconfirmed against an actual shipped incident.

Takeaways so far:
- Soundings' one confirmed miss (#3702) was a cross-boundary state-ownership bug: the regression lived in the gap between "our code updates correctly" and "the library actually propagates what we hand it." The analyst's technical notes show it inspected the right file and line but didn't follow the data flow into the third-party library's own re-render semantics.
- #3631 — the cleanest recall test available (a real shipped bug, known root cause, known fix) — produced a **partial catch**. The analysis's top-severity concern independently named the correct symptom (disconnected store, breadcrumbs silently stop updating) because the PR's own design doc already documented that exact failure mode as a "misuse" risk. The analysis repeated that documented risk rather than discovering the deeper, undocumented one: that the same symptom occurs even without misuse, because of webpack's own module-duplication behavior across the host bundle and the MF exposed-entry graph. This suggests the analyst is good at surfacing risks an author already flagged in-repo, but did not independently derive a root cause the authors themselves hadn't written down.
- Across both PRs with confirmed regressions, Soundings has not yet produced a clean full catch: #3702 was missed entirely (verdict: recommended for release), and #3631 was half-caught (right symptom, wrong/incomplete cause, though the verdict was correctly manual-review).
- Both pure manual-review verdicts with no confirmed shipped defect (#3671, #3753) flagged plausible, well-reasoned risks that were never actually proven to be bugs — evidence of reasonable caution, not of catching anything. This cuts against reading "manual review" verdicts as inherently higher-value than "release" verdicts without checking whether the underlying concern was ever real.
- The one cross-repo test produced the most independently-derived finding of the five runs (traced through actual webpack config in both repos rather than repeating a documented risk), but it also surfaced an operational gap: Soundings cannot resolve an open PR from a fork using the upstream repo's URL, and diagnosing that requires `gh`/shell access the analysis skill's own turn deliberately locks out. A team running shadow-mode pilots on open, fork-based PRs needs a human to pre-resolve the correct fork URL before invoking Soundings.

## Final evaluation verdict

**Mixed results. Would not recommend adoption on the evidence gathered in this pilot.**

1. **Inconsistent benefit from following its release recommendations.** It catches issues, but not reliably the regressions that have actually caused problems for this team. Across the two confirmed-regression cases, it missed one outright (#3702, verdict "recommended for release") and half-caught the other (#3631 — right symptom, wrong root cause).
2. **Slow and token-expensive to run.** Each single-repo run took roughly two to five minutes of agent time and tens of thousands of tokens for the risk-analyst stage alone; the cross-repo run took nearly five minutes and over 100,000 tokens. That's a real cost to weigh against the value delivered, especially given point 1.
3. **Running it reliably will cost ongoing upkeep.** We hit a genuine MCP policy block before this environment cleared it, and a fork-resolution failure that required dropping out of the analysis skill to run `gh` by hand before retrying. Both were solvable, but both represent maintenance/support burden a team would absorb repeatedly, not one-time setup cost.
4. **Its verdict label doesn't reliably track actual risk.** The PR with a confirmed regression and low apparent risk got "recommended for release"; two PRs with no confirmed defect got "manual review." In this sample, the headline verdict is not a trustworthy filter on its own — you'd need to read the underlying concerns every time regardless of which verdict it assigns, which undercuts using it as any kind of gate, even an advisory one.
5. **Its best findings tend to restate what the PR already documented, not surface what the author missed.** The #3631 and cross-repo runs both got their top concern largely right because the author's own code comments or design docs already described the risk. The one clear miss (#3631's actual root cause: webpack duplicating a module even under *correct* usage) was exactly the thing nobody had written down — suggesting the tool is better at summarizing documented risk than at independently finding undocumented defects, which is the harder and more valuable half of review.
6. **Verifying its findings took real reviewer time.** Every finding we wanted to trust required us to independently dig through git history, linked PRs, and tickets to confirm or refute it. That verification labor is comparable to doing the review directly, which weakens the case that the tool nets out as a time saver rather than just moving the work around.
7. **The evidence base here is thin, by design of a quick pilot.** Two confirmed-regression cases (one miss, one partial catch) and one cross-repo case with no ground truth yet is not a statistically strong sample. This verdict should be read as "not justified by the evidence gathered," not "proven ineffective" — a larger, more expensive pilot could move the picture, but points 2–3 are reasons to be cautious about committing to that cost without first seeing the team's own appetite for it.

**Bottom line for the Platform Experience framework team:** do not adopt Soundings as a release gate or standard review step based on this pilot. If there is continued interest, the highest-value next step is not more case studies but addressing point 4 and point 1 directly — e.g., running a larger, blinded sample with the team's own reviewers scoring recall/precision against known incidents — before incurring further token and maintenance cost.

## Red Hat MCP policy implications

The local [Verified MCP server list PDF](</Users/btweed/Downloads/Verified Model Context Protocol (MCP) server list - The Source.pdf>) says its listed enterprise application servers are the preferred, internally reviewed path, but that omission from the list is **not** a stop-work notice. Soundings is not on that list.

Soundings is an agentic use of an MCP server: its local Go helper fetches GitHub/GitLab release data and its Claude Code `risk-analyst` agent assesses that data. For consuming an existing MCP server within an internal agentic solution, the PDF directs users to the **Internal AI Agent Path to Production** to determine whether an AI Assessment is required; if it is, submit a **Velocity AI request**. The MCP Risk and Compliance Framework applies if we own, build, or deploy the server. The PDF does not establish that merely using Soundings requires adding it to the verified list.

**Recommended next step:** follow the Internal AI Agent Path to Production for this narrowly scoped evaluation. If it requires an AI Assessment, submit the Velocity AI request with the Soundings version, helper tools and data flow, and the public PR #3753 pilot scope. Then have the Claude Code administrator resolve the current MCP block through that process. Do not treat the runtime warning alone as proof that unlisted Soundings is prohibited, and do not try to bypass the MCP control.

## Pilot: completed

The pilot described below has run (five cases: #3753, #3631, #3702, #3671, and the learning-resources/#3674 cross-repo pair). See "Final evaluation verdict" above for the conclusion. This section is kept as a record of the original pilot design in case the team decides to run a larger follow-on pilot per that verdict's closing recommendation.

Select three to five representative GitHub/GitLab compare URLs, including at least one release with a known issue or known-good outcome. Confirm whether the pilot can use Claude Code and which repositories/hosts it may access. Then run the tool in shadow mode and complete the scorecard above.
