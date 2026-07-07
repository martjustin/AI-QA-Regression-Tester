# User Stories — AI QA Regression Testing Agent

Format: `As a <persona>, I want <capability>, so that <benefit>.`
Each story includes acceptance criteria and a priority (MoSCoW).

---

## Epic 1 — Test Suite Management

### US-1.1 (Must)
**As a** QA engineer, **I want** to define regression test cases as rows in a spreadsheet, **so that** I can add/edit/disable tests without touching the n8n workflow.

**Acceptance criteria:**
- Adding a row with `active=true` makes it eligible for the next run.
- Setting `active=false` removes it from execution without deleting history.
- Required columns are validated on load; a malformed row is skipped and listed in the run report under "Skipped — invalid config," not silently dropped.

### US-1.2 (Should)
**As a** QA engineer, **I want** to tag tests (`smoke`, `checkout`, `critical`, etc.), **so that** I can trigger a targeted subset from CI/CD instead of the full suite.

**Acceptance criteria:**
- Webhook trigger accepts an optional `tags` array in the payload.
- If provided, only tests whose `tags` intersect the payload tags run; if omitted, all active tests run.

---

## Epic 2 — Execution

### US-2.1 (Must)
**As a** release manager, **I want** the full regression suite to run automatically every night, **so that** issues are caught before the team starts work.

**Acceptance criteria:**
- Schedule trigger fires at a configurable time.
- A run completes and posts a summary even if some individual tests fail (no single failure aborts the run).

### US-2.2 (Must)
**As a** developer, **I want** to trigger the regression suite on-demand from my CI/CD pipeline after a deploy, **so that** I get fast feedback on my change.

**Acceptance criteria:**
- Webhook accepts a POST with optional `tags` and `build_id`.
- Workflow responds synchronously with a JSON result (`status`, `pass_count`, `fail_count`, `risk_rating`) suitable for a pipeline gate.

### US-2.3 (Should)
**As a** QA engineer, **I want** tests executed in controlled batches, **so that** the regression run doesn't hammer the target environment or hit rate limits.

**Acceptance criteria:**
- Batch size is configurable (default 5 concurrent).
- A single slow/hanging test times out (configurable, default 30s) without blocking the whole batch indefinitely.

---

## Epic 3 — Intelligent Verdicts

### US-3.1 (Must)
**As a** QA engineer, **I want** exact/structural checks (status code, schema, field values) applied first, **so that** obviously-passing tests don't cost an AI call.

**Acceptance criteria:**
- If all non-dynamic fields match exactly and status/schema match, verdict = `PASS`, no AI call is made.

### US-3.2 (Must)
**As a** developer, **I want** the AI to distinguish a real regression from a harmless dynamic-field mismatch (timestamp, request ID, reordered array), **so that** I'm not paged for noise.

**Acceptance criteria:**
- Fields listed in a test's `dynamic_fields` are excluded from strict diffing and, if they're the *only* mismatch, are auto-classified without needing AI judgment call on content — but any other mismatch triggers full AI review.
- AI verdict includes `root_cause_category` and a one-paragraph `reasoning` string.

### US-3.3 (Should)
**As a** QA lead, **I want** the AI to flag when a failure looks like a stale/outdated expectation rather than a real bug, and propose the corrected expected value, **so that** I can approve suite maintenance quickly instead of manually re-deriving the correct value.

**Acceptance criteria:**
- When `root_cause_category = STALE_EXPECTATION`, a row is appended to a "Suggested Updates" sheet with `test_id`, `field`, `old_expected`, `suggested_expected`, `reasoning`, `confidence`.
- Nothing in the live test suite is modified automatically.

### US-3.4 (Could)
**As a** QA engineer, **I want** the agent to retry a failing test once before finalizing a `FAIL` verdict, **so that** genuinely flaky/environment-timing issues aren't reported as regressions on the first strike.

**Acceptance criteria:**
- One automatic retry on failure; if the retry passes, verdict = `FLAKY` (not `PASS`, so it's still visible in trend data) rather than `REGRESSION`.

---

## Epic 4 — Reporting & Alerting

### US-4.1 (Must)
**As a** release manager, **I want** an AI-written executive summary after every run with an overall risk rating, **so that** I can make a go/no-go decision without reading every test result.

**Acceptance criteria:**
- Summary includes: pass rate, count of new regressions vs. known-flaky, top 3 risks in plain English, and a `LOW/MEDIUM/HIGH/CRITICAL` rating.
- Posted to a Slack channel automatically after every run.

### US-4.2 (Must)
**As a** developer, **I want** a ticket auto-filed for each genuine regression with the request/response, expected vs. actual, and the AI's reasoning, **so that** I can start debugging immediately without re-running the test myself.

**Acceptance criteria:**
- Ticket created only for `verdict=FAIL` with `root_cause_category != STALE_EXPECTATION/ENV_ISSUE` above the severity threshold.
- Multiple failures from the same underlying cause (same endpoint + same root cause category) are grouped into one ticket, not one-per-test.

### US-4.3 (Should) — implemented v1.1
**As an** engineering manager, **I want** every run's results stored historically, **so that** I can see pass-rate trends and identify chronically flaky tests worth fixing or retiring.

**Acceptance criteria:**
- Each run and each per-test result is persisted with `run_id`, timestamp, verdict, and root cause (`qa_test_run_history`).
- A test that has been `FLAKY` in ≥3 of the last 5 runs is flagged in the summary as "chronically flaky — recommend review." (`Load Recent Test History` + `Aggregate Run Results`.)
- A root cause recurring in ≥2 of a test's prior runs is flagged as a "recurring issue" with the build it first appeared in, not just called a fresh regression each time.

### US-4.4 (Won't, v1)
**As a** QA lead, **I want** a live dashboard of pass-rate trends, **so that** I don't have to query the history table manually.
*(Deferred to v2 — v1 exposes the raw history table for use in Sheets/BI tools.)*

---

## Epic 5 — Safety & Trust

### US-5.1 (Must)
**As a** QA lead, **I want** the AI to never auto-close tickets, and to only auto-modify a test expectation within a narrow, audited envelope, **so that** the suite's source of truth stays under human control by default, with autonomy earned only for the safest class of change.

**Acceptance criteria:**
- A suggestion auto-applies only if: exactly one field mismatched (no ambiguity about which value to change), confidence ≥ 0.95, test `priority != critical`, and the field isn't on the protected/monetary list.
- Every suggestion, auto-applied or not, is written to `qa_suggested_updates` with an `auto_applied` flag — there is no code path where a suggestion is applied without a corresponding audit row.
- Tickets are never auto-closed; a resolved recurring cluster still requires a human to close its Jira issue.

---

## Epic 6 — Differentiators (v1.1)

### US-6.1 (Must)
**As a** release manager, **I want** the risk rating to be computed the same way every time from the same inputs, **so that** I can trust the go/no-go signal instead of wondering if the AI would call it differently on a re-run.

**Acceptance criteria:**
- `risk_score`/`risk_rating` are computed by a deterministic formula (`Compute Risk Score`) from counts, critical-path failures, and cross-run signals — before the executive-summary AI call.
- The AI's executive-summary response supplies `headline`/`top_risks`/`narrative` only; if it is missing or malformed, `risk_rating` is unaffected.

### US-6.2 (Should)
**As a** QA engineer, **I want** UI test failures adjudicated with an actual before/after screenshot comparison, **so that** I get a description of what visually changed instead of just a similarity score I have to go inspect myself.

**Acceptance criteria:**
- UI-test escalations route to a multimodal Claude call with baseline and current screenshots attached.
- The verdict includes a `visual_diff_description` in addition to the standard `verdict`/`confidence`/`root_cause_category`/`reasoning`.
- A mismatch confined to a field listed in `dynamic_fields` (e.g. a rotating promo banner) is not treated as a failure.

### US-6.3 (Could)
**As a** developer on an engineering-heavy team, **I want** the option to define regression tests in a version-controlled JSON file instead of a spreadsheet, **so that** test changes go through the same PR review as code changes.

**Acceptance criteria:**
- Setting `test_source: "git"` on a webhook-triggered run loads tests from a git raw JSON URL instead of the Sheet.
- Git-sourced and Sheet-sourced tests are validated and executed identically once loaded — no downstream node needs to know which source was used.
- Omitting `test_source` (or the nightly schedule trigger, which has no body) still defaults to the Sheet — this is additive, not a breaking change.

### US-6.4 (Should)
**As a** developer, **I want** a recurring failure to update one Jira ticket instead of spawning a new one every night, **so that** my ticket queue reflects distinct problems, not run count.

**Acceptance criteria:**
- Each failure cluster gets a stable `cluster_label` derived from its root-cause category and the set of failing test IDs.
- If an open Jira issue already carries that label, the agent comments "still failing as of run X" instead of creating a new issue.
- Only a cluster with no matching open issue results in a new ticket, which is then tagged with the label for future runs to find.

### US-5.2 (Should)
**As a** developer, **I want** low-confidence AI verdicts (`confidence < 0.6`) routed to `NEEDS_REVIEW` instead of a hard PASS/FAIL, **so that** uncertain calls surface for a human rather than silently going either way.

**Acceptance criteria:**
- `NEEDS_REVIEW` items appear in a distinct section of the summary and do not, by themselves, trigger a paging alert (unless they involve a `priority:critical` test, in which case they do — erring toward visibility for critical paths).
