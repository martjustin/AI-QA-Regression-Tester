# PRD — AI QA Regression Testing Agent

**Product:** QA Sentinel — an n8n-orchestrated, AI-assisted regression testing agent
**Doc owner:** Martin Justin
**Status:** v1.1
**Last updated:** 2026-07-06

**v1.1 adds six capabilities beyond the original scope:** cross-run pattern intelligence (chronic-flaky/recurring-issue detection), tiered-autonomy auto-apply for narrowly-scoped stale-expectation fixes, deterministic risk scoring, multimodal visual diffing for UI tests, a test-as-code alternative to the Sheet, and Jira ticket de-duplication across runs. See FR-15 through FR-20 and `ARCHITECTURE.md` §9 for design rationale.

---

## 1. Problem Statement

Regression suites today fail in one of two directions:

1. **Too brittle** — deterministic assertions (exact string/JSON match) break on every harmless change (a timestamp, a reordered array, a rewritten error message), producing false positives that teams learn to ignore. Alert fatigue sets in and real regressions get lost in the noise.
2. **Too shallow** — to avoid brittleness, teams loosen assertions (status-code-only checks) and miss real regressions in response content, business logic, or copy.

Meanwhile, triage is manual: an engineer has to open every failed run, diff actual vs. expected by eye, decide "is this a real bug, a flaky test, or a stale expectation," and then file a ticket or ping Slack. This is slow, repetitive, and doesn't scale with release cadence.

## 2. Goal

Build an agent that runs regression suites automatically, uses **deterministic checks first** (fast, cheap, unambiguous), and escalates only the **ambiguous or content-sensitive cases** to an LLM (Claude) for semantic judgment — then produces a triaged, prioritized, human-readable report and opens tickets/alerts automatically for real regressions, while suggesting (not auto-applying) updates for stale expectations.

## 3. Goals / Non-Goals

### Goals
- Automatically execute API (and optionally DB/UI-hook) regression tests on a schedule or CI/CD trigger.
- Apply deterministic assertions (status code, JSON schema, key-field equality) as the first pass.
- Use AI to adjudicate only ambiguous results: semantic diffing of text, tolerance for expected-dynamic fields (timestamps, IDs, nonces), classification of failures into `REGRESSION / FLAKY / TEST_DATA_ISSUE / ENV_ISSUE / STALE_EXPECTATION`.
- Produce an executive-level AI-generated run summary with a risk rating.
- Auto-file tickets/alerts for genuine regressions above a severity threshold; keep humans in the loop for anything the AI proposes changing.
- Persist run history for trend analysis (pass-rate over time, recurring flaky tests).
- Be fully configurable from a spreadsheet (no-code test case management for QA engineers).

### Non-Goals (v1)
- Full browser UI automation engine (v1 delegates UI tests to an external Playwright/Selenium runner via webhook; it does not implement browser automation itself).
- Test case authoring/generation from requirements (candidate for v2).
- Auto-merging AI-suggested expectation updates without human approval **as a blanket policy** — v1.1 permits a narrow, audited exception (see FR-16): a single-field, high-confidence, non-critical, non-monetary stale-expectation fix may auto-apply, with a `qa_suggested_updates` audit row on every suggestion regardless. Anything outside that envelope still requires human approval.
- Load/performance testing.
- Replacing existing CI test frameworks — this agent complements them for black-box/API regression coverage.

## 4. Personas

| Persona | Need |
|---|---|
| **QA Engineer** | Maintain test cases in a simple sheet, understand *why* something failed without manually diffing payloads. |
| **Release Manager** | A go/no-go signal before promoting a build, with a clear risk rating and list of blocking regressions. |
| **Backend Developer** | Get a ticket with reproduction details (request, expected, actual, AI's reasoning) instead of a vague "test failed." |
| **Engineering Manager** | Trend visibility: is regression pass rate improving, which tests are chronically flaky and should be fixed or retired. |

## 5. User Stories

See `USER_STORIES.md` for the full backlog. Highlights:

- As a QA engineer, I want to add a new regression test by adding a row to a spreadsheet, so that I don't need to touch workflow code.
- As a release manager, I want an automatic Slack summary after every nightly run with a clear PASS/AT-RISK/BLOCKED verdict, so I can make a go/no-go call quickly.
- As a developer, I want failed tests to auto-file a ticket with the AI's root-cause classification and reasoning, so I can triage faster.
- As a QA lead, I want the agent to tell me when a failure is *probably* a stale expectation (not a real bug) and propose the correction, without silently accepting it, so test-suite maintenance doesn't require full manual re-verification every time.
- As an engineering manager, I want historical run data stored, so I can see pass-rate trends and identify chronically flaky tests.

## 6. Functional Requirements

| ID | Requirement |
|---|---|
| FR-1 | Agent triggers on schedule (nightly) and on-demand via webhook (CI/CD pipeline call, with optional tag filter payload). |
| FR-2 | Test cases are stored in a Google Sheet (or Airtable) with a defined schema (see `sample_test_suite.json`). |
| FR-3 | Only tests marked `active = true` (and matching an optional tag filter) run. |
| FR-4 | Tests are executed in controlled batches (default batch size 5) to avoid overwhelming the target environment. |
| FR-5 | Each test type (`api`, `db`, `ui`) is routed to its own execution method. |
| FR-6 | Deterministic assertions run first: HTTP status match, JSON schema validation, exact-match on all fields *not* listed in `dynamic_fields`. |
| FR-7 | If deterministic result is `PASS`, AI is **not** called (cost control). If `FAIL` or the diff touches a field marked as potentially-dynamic/semantic, the case is escalated to Claude for adjudication. |
| FR-8 | The AI adjudication returns a structured verdict: `verdict` (PASS/FAIL/FLAKY/NEEDS_REVIEW), `confidence` (0–1), `root_cause_category`, `reasoning`, and (if applicable) `suggested_expected_value`. |
| FR-9 | Suggested expectation updates are written to a separate "Suggested Updates" sheet/table for human review — never auto-applied. |
| FR-10 | After all tests complete, an AI-generated executive summary is produced: pass/fail counts, new failures vs. previously-known-flaky, top risks, overall risk rating (`LOW/MEDIUM/HIGH/CRITICAL`). |
| FR-11 | If risk rating ≥ configurable threshold (default `HIGH`) or any test tagged `priority: critical` fails, the agent opens a ticket (Jira/Linear) per failure and posts an @-mention Slack alert. |
| FR-12 | Otherwise, a normal (non-paging) Slack summary is posted to the QA channel. |
| FR-13 | Every run's full results are persisted (Postgres/Airtable) with `run_id`, timestamps, and per-test outcome for trend analysis. |
| FR-14 | If triggered via webhook, the workflow responds synchronously with a JSON summary (for CI/CD gating). |
| FR-15 | Per-test verdict history is persisted every run (`qa_test_run_history`); each run's aggregation reads the last 5 runs per test to flag tests `FLAKY` in ≥3 of 5 runs ("chronically flaky") and root causes recurring in ≥2 prior runs ("recurring issue"), surfaced in Slack and the executive summary. |
| FR-16 | A `STALE_EXPECTATION` suggestion auto-applies to the live test source only if: exactly one field mismatched, confidence ≥ 0.95, test priority ≠ `critical`, and the field is not on the protected/monetary list. Every suggestion — auto-applied or not — is logged to `qa_suggested_updates` with an `auto_applied` flag. |
| FR-17 | The run's `risk_rating`/`risk_score` is computed deterministically from counts, critical-path failures, and cross-run signals, before the executive-summary AI call; the AI writes narrative content around the fixed rating and cannot override it. |
| FR-18 | `test_type: ui` escalations that need AI review are adjudicated with a multimodal call (baseline + current screenshot images) that returns a `visual_diff_description`, in addition to the standard verdict fields. |
| FR-19 | Test cases may be sourced from a git-backed JSON file (`test_source: "git"` in the trigger payload) as an alternative to the Google Sheet, normalized to the same downstream contract. |
| FR-20 | Before filing a Jira ticket for a failure cluster, the agent searches for an existing open ticket carrying that cluster's stable label; if found, it comments instead of creating a duplicate. |

## 7. Non-Functional Requirements

- **Cost control:** AI calls only for ambiguous/failed cases, not every test (target: AI invoked on <20% of test executions in a healthy suite).
- **Idempotency:** each run gets a unique `run_id`; re-runs don't corrupt history.
- **Extensibility:** new test types (e.g., GraphQL, gRPC) should be addable by extending the router without restructuring the whole flow.
- **Auditability:** every AI verdict stores its reasoning and the raw prompt/response for later review.
- **Security:** secrets (API keys, DB creds) live only in n8n Credentials, never in sheet data or logs.
- **Human-in-the-loop:** the agent never closes tickets automatically, and never modifies test expectations outside the narrow, audited auto-apply envelope defined in FR-16 — everything else requires human review via `qa_suggested_updates`.

## 8. Success Metrics

| Metric | Target |
|---|---|
| False-positive regression alerts (post AI-adjudication) | ↓ 70% vs. pure deterministic baseline |
| Mean time-to-triage a failed run | < 10 minutes (from "AI summary posted" to "root cause understood") |
| % of test executions that invoke the LLM | < 20% |
| Nightly run completion | 100% (with retry/backoff on transient HTTP errors) |
| Stale-expectation suggestions accepted by QA (precision proxy) | > 80% accepted-as-is or lightly edited |

## 9. Risks & Mitigations

| Risk | Mitigation |
|---|---|
| LLM hallucinates a verdict | Always show reasoning + raw diff in the ticket/report; require confidence ≥ 0.6 or route to `NEEDS_REVIEW`; never auto-close a ticket, and auto-apply a suggested fix only within the FR-16 envelope (single field, ≥0.95 confidence, non-critical, non-monetary) — everything else is logged for review, never silently accepted. |
| Cost overrun from over-calling AI | Deterministic pass-through skips AI entirely; batch + cap `max AI calls per run` config. |
| Flaky external environment causes false regressions | Root-cause category `ENV_ISSUE` + automatic retry (1x) before final verdict. |
| Sheet misconfiguration breaks a run | Schema validation step on load; malformed rows are skipped and reported, not fatal. |
| Ticket spam on a systemic outage | De-duplication: group failures by `root_cause_category` + endpoint before filing tickets; one ticket per cluster. |

## 10. Rollout Plan

1. Pilot on one service's API regression suite (10–20 test cases) — validate deterministic + AI paths.
2. Tune the "needs AI review" trigger condition based on false-positive rate observed.
3. Add Jira/Slack integration for the pilot team only.
4. Expand to additional services; add DB/UI-hook test types.
5. Build a lightweight dashboard (or reuse Airtable/Sheets views) over the run-history table for trend reporting.
