**⚠️ AI-Generated Report** — This report is AI-generated and advisory. Always review AI-generated content prior to use.

# 🚀 Release Readiness Report

## 🎯 Summary

insights-chrome PR #3702 reworks Unleash feature-flag failure handling (fetch timeouts, per-user cache scoping, degraded-state atom, GlobalFilter permission gating) with strong, multi-layer test coverage; no DB/infra/auth changes, residual risk is limited to a few targeted behavioral edge cases.

**Recommendation:** ✅ Recommended for release

No blocking concerns found.

---

## 🔍 Risk Analysis

### Concerns

***Severity:** 🔥 Critical · ⚠️ High · 🟡 Medium · 🟢 Low*

| | Details |
|----------|---------|
| 🟡 | src/components/FeatureFlags/FeatureFlagsProvider.tsx FeatureFlagsStateBridge.notifyInitialFlags() calls the private/internal `client.emit('update')` API of @unleash/proxy-client-react (not a documented public method) to force consumers using `useFlag` to re-render after stored toggles load on init. If a future @unleash/proxy-client-react upgrade changes or removes `emit`'s behavior, flag consumers could silently stop updating after the client initializes from cache, with no compile-time signal (TypeScript allows it because UnleashClient extends EventEmitter). No dependency version bump is included in this diff, but this is a latent fragility introduced by the change. |
| 🟡 | src/components/GlobalFilter/GlobalFilter.tsx changes behavior so that `getUserPermissions` is now called even when `flagsError` is true (flagsResolved = flagsReady \|\| !!flagsError), reversing prior fail-closed behavior confirmed by the flipped assertion in src/components/GlobalFilter/GlobalFilter.test.tsx (`should call getUserPermissions when flags fail to load`). This is an intentional policy change documented in docs/navigation.md's new 'Unleash outage policy' section, but it means the global filter's permission-gated UI will now attempt to render for all users during an actual Unleash outage, which previously fully suppressed it; worth confirming `getUserPermissions`'s own downstream error handling is robust under outage conditions since it's now reachable in a new failure state. |
| 🟢 | src/components/FeatureFlags/unleashClient.ts clearLegacyFeatureFlagsCache() deletes old unscoped `unleash:repository:*` localStorage keys on every client (re)creation. This causes a one-time forced re-fetch of feature flags for every existing user on first load after deploy (previously cached toggles are discarded), and recurs on every identity change (login/logout/account switch) since client recreation re-invokes it. Low impact given the 5s initial timeout and graceful degraded-state handling, but will cause a brief synchronized increase in `/api/featureflags/v0` requests right after rollout. |
| 🟢 | COMPOUND: cache-scoping change (unscoped to org+user-scoped Unleash localStorage keys in src/components/FeatureFlags/unleashClient.ts) combined with the new degraded-state atom (src/state/atoms/degradedStateAtom, referenced but not included in this diff) means the full picture of global degraded-state consumers outside this PR's file set (e.g. any other component reading degradedStateAtom) could not be verified from the provided patch set alone — that file is imported by FeatureFlagsProvider.tsx and the cypress harness but was not part of the 17 changed files, so it must already exist pre-PR; its current shape/consumers elsewhere in the app were not visible for review. |

### Positive Factors
- Exhaustive new unit coverage in src/components/FeatureFlags/FeatureFlagsProvider.test.tsx (368 lines) exercises HTTP 401/503, network rejection, metrics-failure non-degradation, refresh-failure degradation, cache retention through outage + recovery, request timeout/abort, cross-user/legacy cache isolation, and context-change client recreation — directly covering the failure modes this PR introduces.
- New Cypress component tests (cypress/component/DegradedVisibilityFeatureFlags.cy.tsx + helpers/DegradedVisibilityHarness.tsx, 276 lines combined) verify the shell stays rendered and navigation correctly falls back to cached/default toggle values during a real 503 response, including cross-user cache isolation.
- docs/navigation.md and AGENTS.md were updated in the same PR to codify the new 'Unleash outage policy' (cache scoping, fail-disabled-on-no-value, fail-closed-on-no-client, Sentry severity split between warnings and exceptions, GlobalFilter independence) — policy and implementation match on inspection.
- Per-user/org Unleash cache scoping (src/components/FeatureFlags/unleashClient.ts getFeatureFlagsStoragePrefix) closes a plausible shared-browser cross-account toggle leak, with a dedicated regression test ('does not evaluate toggles cached for another user or by an older Chrome version').
- Sentry reporting severity was deliberately split: HTTP >=400 responses now report as `captureMessage(..., 'warning')` rather than being escalated via `captureException`, reducing error-budget noise from expected outages, verified in the 'logs HTTP %s as a warning' test.
- No database, authentication, or infrastructure/CI files are touched in this change set — blast radius is confined to client-side feature-flag handling and navigation visibility within one repository.

---

## 📋 Action Items

### ⚠️ Important (Recommended Before Release)
- Manually verify in a staging environment that `client.emit('update')` in src/components/FeatureFlags/FeatureFlagsProvider.tsx:179 actually triggers a re-render for a component using `useFlag()` immediately after a page load with pre-existing localStorage toggles (cold start with warm cache) — the unit test mocks the client but does not fully exercise `@unleash/proxy-client-react`'s real EventEmitter wiring end-to-end in a browser.
- Confirm `getUserPermissions` (called from src/components/GlobalFilter/GlobalFilter.tsx when `flagsError` is true) has its own error handling/backoff, since it is now invoked in a new reachable state (Unleash outage) that previously short-circuited before this call.

### 📝 Follow-up (Post-Release)
- Monitor Sentry for a spike in 'Feature loading error server error!' warning-level messages (captureMessage calls from src/components/FeatureFlags/FeatureFlagsProvider.tsx validateFeatureFlagsResponse) in the first 24h post-release — baseline against pre-release error-level captureException counts for the same code path to confirm the severity reclassification didn't mask a real outage.
- Track `/api/featureflags/v0` request volume for ~1 hour post-deploy to confirm the expected one-time spike from clearLegacyFeatureFlagsCache() clearing old unscoped cache keys resolves back to baseline refresh cadence (60s refreshInterval) and does not plateau higher (which would indicate the new scoped cache isn't persisting as expected).

---

<details>
<summary><strong>🛠️ Technical Details</strong></summary>

### 📝 Code Changes
- Read in full: src/components/FeatureFlags/FeatureFlagsProvider.tsx (medium tier, 271 patch lines) — core rewrite adding createFetchFeatureFlags() with per-phase timeouts (FEATURE_FLAGS_INITIAL_TIMEOUT_MS=5000ms, FEATURE_FLAGS_REFRESH_TIMEOUT_MS=15000ms), validateFeatureFlagsResponse(), and a new FeatureFlagsStateBridge component managing client.start()/stop() lifecycle and degraded-state atom updates.
- Read in full: src/components/FeatureFlags/unleashClient.ts (medium tier, 46 patch lines) — adds getFeatureFlagsStoragePrefix(orgId, userId), clearLegacyFeatureFlagsCache(), setFeatureFlagsError().
- Read in full: src/utils/VisibilitySingleton.ts (medium tier, 84 patch lines) — featureFlag(), isKesselEnabled(), isKesselOrgOnboarded() refactored to a shared matchesFeatureFlag() helper that drops the old getFeatureFlagsError() short-circuit (previously 'error present -> treat as false'); new behavior falls back to the last-cached toggle value during an outage instead of forcing false — confirmed by flipped VisibilitySingleton.test.ts assertions ('uses the cached enabled/disabled value when feature flags have an error').
- Read in full: src/utils/cacheFetch.ts (medium tier, 30 patch lines) — trivial, reuses the new shared FEATURE_FLAGS_INITIAL_TIMEOUT_MS constant instead of a locally duplicated FLAG_READY_TIMEOUT_MS.
- Read in full: src/components/GlobalFilter/GlobalFilter.tsx (medium tier, 33 patch lines) — flagsResolved = flagsReady \|\| !!flagsError gates the permission-fetch effect instead of flagsReady alone.
- Read in full: src/components/FeatureFlags/FeatureFlagsProvider.test.tsx (medium tier, new file, 375 patch lines), src/components/GlobalFilter/GlobalFilter.test.tsx, src/utils/VisibilitySingleton.test.ts, playwright/helpers/feature-flags.ts and .test.ts, playwright/integration/*.spec.ts, cypress/component/DegradedVisibilityFeatureFlags.cy.tsx and helpers/DegradedVisibilityHarness.tsx — read fully given they are the direct evidence for test-coverage claims about the outage-handling behavior.
- Skimmed (low tier, no content concerns found): AGENTS.md, docs/navigation.md diffs (read in full despite low tier since they define the policy being implemented — both tier and content indicated this was worth full reading).
- Not available for review: src/state/atoms/degradedStateAtom module (imported by FeatureFlagsProvider.tsx and the cypress harness) is not part of this diff's 17 files, implying it predates this PR; its other consumers in the codebase were out of scope for this patch set and not independently verifiable here.

### 🏗️ Infrastructure Changes
- No CI/CD, deployment, or environment configuration files are present in this change set.

### 🔗 Dependency Changes
- No package.json/lockfile changes are present in this diff; the PR relies on existing @unleash/proxy-client-react APIs (InMemoryStorageProvider, LocalStorageProvider, UnleashClient.emit/on/off/start/stop) without a version bump.

</details>

---

<details>
<summary><strong>📋 Changelogs</strong></summary>

### [https://github.com/RedHatInsights/insights-chrome](https://github.com/RedHatInsights/insights-chrome/compare/8bc1c8f141192460cba28dd90e7af0c2d7587ab8...7591e82be4ee6f98db669bd8d37a96efe8aecdf4)
*Total commits: 1*

| SHA | Message | Author | PR |
|-----|---------|--------|----|
| [7591e82b](https://github.com/RedHatInsights/insights-chrome/commit/7591e82be4ee6f98db669bd8d37a96efe8aecdf4) | Handle Unleash failures without breaking navigation RHCLOUD-51464 (#3702) | Petr Šimon | [#3702](https://github.com/RedHatInsights/insights-chrome/pull/3702) |

</details>

---

<details>
<summary><strong>📚 Release Documentation</strong></summary>

### 📁 Documentation Sources Analyzed
No repository documentation was found or analyzed.

### 🔍 Overall Assessment
The docs/ entry in index.json was null (no fetch_error flag present either, so this run's fetcher did not retrieve separate repo documentation beyond the in-diff docs/navigation.md and AGENTS.md files). Within the diff itself, documentation quality is good: AGENTS.md was updated with a direct pointer to the new policy section, and docs/navigation.md's new 'Unleash outage policy' section is precise, enumerates exact behaviors (cache key scheme, timeout values, fail-open/fail-closed semantics, Sentry severity split), and matches the implementation on inspection.

### 💡 Improvement Recommendations
Because src/state/atoms/degradedStateAtom was referenced but not included in this diff, a short module-level comment or docs/navigation.md cross-reference listing its known consumers (what else reads `degradedStateAtom` besides this PR's new code) would let future reviewers assess blast radius without needing the full repository checked out. Also consider documenting in docs/navigation.md that `client.emit('update')` is a deliberate use of an internal/undocumented @unleash/proxy-client-react API, with a note to re-verify this call after any future @unleash/proxy-client-react version bump, since it is not covered by that library's own public type contract.

</details>

---

<details>
<summary><strong>📈 Want Better Analysis Results?</strong></summary>

Learn how to get more accurate verdicts and sharper analysis:

👉 **[Guide: Improving Your Release Readiness Analysis](https://github.com/gwenneg/soundings/blob/main/docs/IMPROVING_ANALYSIS.md)**

**Quick tips:**
- Add `.soundings.md` to your repository for context-aware analysis
- Use `/soundings note` comments to provide context the AI can't infer from code
- Keep PRs/MRs focused and reasonably sized for better analysis quality

</details>

---

*🤖 Generated by [Soundings](https://github.com/gwenneg/soundings) | claude-sonnet-5 | 2026-10-23 21:52:47 UTC*
