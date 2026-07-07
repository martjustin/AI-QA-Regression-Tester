# AI QA Regression Testing Agent — n8n

A hybrid **deterministic + AI-adjudicated** regression testing agent built as a single n8n workflow. It runs your regression suite (API / DB / UI-hook), checks results with fast exact-match assertions first, and only escalates ambiguous or failing cases to Claude for semantic judgment — then produces an executive summary, alerts, and tickets, with a full audit trail. On top of the v1 core it now includes six differentiating capabilities: cross-run pattern intelligence, tiered-autonomy stale-expectation fixes, deterministic risk scoring, multimodal visual diffing, a test-as-code source option, and Jira ticket de-duplication across runs — see [Feature differentiators](#feature-differentiators) below.

## Files 

| File | What it is |
|---|---|
| `qa-regression-agent.n8n.json` | The n8n workflow — import this directly into n8n. |
| `PRD.md` | Product requirements: problem, goals, personas, functional/non-functional requirements, success metrics, risks. |
| `ARCHITECTURE.md` | Node-by-node design, data contracts, AI prompt contracts, error handling, security, extensibility. |
| `USER_STORIES.md` | Full backlog of user stories with acceptance criteria, grouped by epic. |
| `sample_test_suite.json` / `sample_test_suite.csv` | 12 sample regression test cases (API, DB, UI) — including deliberately ambiguous/failing/malformed rows to exercise every path of the agent. Import the CSV into the Google Sheet the workflow reads from. |

## What it does, in one paragraph

On a nightly schedule (or on-demand via webhook from CI/CD), the agent loads active test cases from a spreadsheet (or a git-backed JSON file), executes them in controlled batches, and checks each result deterministically (status code, JSON schema, exact field match, `_min`/`_max` threshold assertions — skipping fields you've marked as expected-to-vary). Anything that isn't a clean deterministic PASS — including tests that only have semantic assertions — is sent to Claude with the expected value, actual value, and context (screenshots too, for UI tests), and comes back with a structured verdict (`PASS/FAIL/FLAKY/NEEDS_REVIEW`), a root-cause category, and reasoning. After the full run, a deterministically computed risk score decides `LOW/MEDIUM/HIGH/CRITICAL`, and Claude writes the narrative around it — factoring in chronically flaky tests and root causes recurring across builds, not just this run. If risk is high or a critical test failed, the agent pages Slack and files (or comments on an existing) grouped Jira ticket per failure cluster; otherwise it posts a routine summary. High-confidence, non-critical, non-monetary stale-expectation fixes are applied to the live suite automatically with a full audit trail; everything else is queued for human review. Every run is persisted for trend analysis.

## Prerequisites

- n8n instance (self-hosted or cloud), version supporting the node types used below (n8n ≥ 1.4x recommended; if your instance has older node versions, adjust `typeVersion` per node — see "Version notes" below).
- Anthropic API key (Claude access, including vision/multimodal support for the visual-diff feature).
- A Google Sheet (or swap for Airtable/Notion) with a "Test Suite" tab matching the schema in `sample_test_suite.csv`. **Or**, use the git-backed test source instead/alongside it — see [Test-as-code source](#6-test-as-code-alternative-source).
- A Postgres database (or swap for Airtable) for run history storage — schema below (now three tables: run history, per-test history, suggested updates).
- Slack workspace with a bot token (`chat:write` scope) and two channels: e.g. `#qa-regression` (routine) and `#qa-critical-alerts` (paging).
- Jira Cloud project + API token (or swap for Linear/GitHub Issues).
- (Optional) An external UI test runner (Playwright/Selenium microservice) exposing a webhook if you use `test_type: ui` cases. For visual diffing, extend its response contract to also return `baseline_screenshot_base64` / `current_screenshot_base64` (PNG, base64) alongside `screenshot_match_score`.

## Setup

1. **Import the workflow**: n8n → Workflows → Import from File → `qa-regression-agent.n8n.json`.
2. **Create the Test Suite sheet**: create a Google Sheet, add a tab named `Test Suite`, and import `sample_test_suite.csv` as its content (or start from it and add your own tests). Open the `Load Test Suite` node **and** the `Update Test Suite (Auto-Apply)` node, and set `documentId` on both to your sheet, replacing the `REPLACE_WITH_SHEET_ID` placeholder.
3. **Set up credentials** in n8n → Credentials:
   - **Anthropic API Key** — HTTP Header Auth credential, header name `x-api-key`, value = your Anthropic key. Attach it to `Claude - Semantic Diff`, `Claude - Visual Diff`, and `Claude - Executive Summary`.
   - **Google Sheets** OAuth2 — attach to `Load Test Suite` and `Update Test Suite (Auto-Apply)`.
   - **Postgres** — attach to `Execute DB Query`, `Load Recent Test History`, `Store Run History`, `Insert Test History`, and `Insert Suggested Updates`.
   - **Slack** — attach to `Slack - Critical Alert` and `Slack - Routine Summary`; update `channelId` values to your real channel names/IDs.
   - **Jira** (Basic Auth: email + API token) — attach to `Search Existing Jira Ticket`, `Comment on Existing Ticket`, and `Create Jira Tickets`; replace `YOUR_DOMAIN` and the `project.key` (`QA`) in each node with your real values.
   - **Webhook Header Auth** (recommended) — protect `CI/CD Webhook Trigger` with a shared secret so only your pipeline can trigger runs.
   - (Optional, git-backed source) **Header Auth** on `Load Test Suite (Git)` if your repo is private (e.g. a GitHub PAT as a Bearer/Authorization header).
4. **Create the database tables** in Postgres:
   ```sql
   CREATE TABLE qa_run_history (
     id SERIAL PRIMARY KEY,
     run_id TEXT NOT NULL,
     started_at TIMESTAMPTZ NOT NULL DEFAULT now(),
     pass_count INT, fail_count INT, flaky_count INT, needs_review_count INT,
     risk_rating TEXT,
     risk_score NUMERIC,
     narrative TEXT
   );

   -- Per-test history: powers cross-run pattern intelligence (chronic flaky / recurring root cause detection).
   CREATE TABLE qa_test_run_history (
     id SERIAL PRIMARY KEY,
     run_id TEXT NOT NULL,
     test_id TEXT NOT NULL,
     build_id TEXT,
     verdict TEXT NOT NULL,
     root_cause_category TEXT,
     priority TEXT,
     created_at TIMESTAMPTZ NOT NULL DEFAULT now()
   );
   CREATE INDEX idx_qa_test_run_history_test_id ON qa_test_run_history (test_id, created_at DESC);

   -- Full audit trail for stale-expectation suggestions, auto-applied or not.
   CREATE TABLE qa_suggested_updates (
     id SERIAL PRIMARY KEY,
     run_id TEXT NOT NULL,
     test_id TEXT NOT NULL,
     field TEXT,
     old_expected JSONB,
     suggested_expected JSONB,
     reasoning TEXT,
     confidence NUMERIC,
     auto_applied BOOLEAN NOT NULL DEFAULT false,
     created_at TIMESTAMPTZ DEFAULT now(),
     reviewed BOOLEAN NOT NULL DEFAULT false
   );
   ```
5. **Adjust the schedule** in `Schedule Trigger` (default: nightly at 02:00, cron `0 2 * * *`).
6. **Test it**: run the workflow manually in the n8n editor (Execute Workflow), or send a test POST to the webhook:
   ```bash
   curl -X POST https://your-n8n-host/webhook/qa-regression-trigger \
     -H "Content-Type: application/json" \
     -H "X-Auth: your-shared-secret" \
     -d '{"tags": ["smoke"], "build_id": "1234"}'
   ```
   Pass `"test_source": "git"` in the body to load tests from the git-backed JSON source for that run instead of the Sheet (see below); omit it (or use `"sheet"`) for the default spreadsheet source.

## Feature differentiators

### 1. Cross-run pattern intelligence
`Load Recent Test History` pulls each test's last 5 runs from `qa_test_run_history` before `Aggregate Run Results` runs. Two things fall out of that: a test that's been `FLAKY` in ≥3 of its last 5 runs (this run included) is flagged as **chronically flaky**, and a `FAIL` whose `root_cause_category` matches ≥2 prior runs is flagged as a **recurring issue** with the build it first appeared in. Both surface in the Slack message and are handed to the executive-summary prompt so the narrative calls them out explicitly instead of treating every run as a clean slate. `Prepare Test History Rows` → `Insert Test History` writes this run's results back so the next run has history to compare against.

### 2. Tiered autonomy for stale-expectation fixes
Every `STALE_EXPECTATION` verdict is still logged to `qa_suggested_updates` for human review (via `Flatten Suggested Updates` → `Insert Suggested Updates`) — nothing is silent. But a narrow slice — single unambiguous mismatched field, confidence ≥ 0.95, non-`critical` priority, and not a monetary/protected field (`price|total|subtotal|discount|amount|cost|balance|currency`, checked in `Aggregate Run Results`) — is additionally auto-applied: `Prepare Auto-Apply Sheet Rows` patches just that one field and `Update Test Suite (Auto-Apply)` writes it to the live sheet, with `auto_applied = true` recorded in the audit row. Financial fields, critical-priority tests, and anything with more than one mismatched field always fall through to manual review, no matter the confidence score.

### 3. Deterministic risk scoring
`Compute Risk Score` calculates `risk_score` (0–100) and `risk_rating` from hard counts — fail/needs-review/flaky counts, a critical-path failure override, and weight for recurring/chronic issues — **before** Claude ever sees the run. The executive-summary prompt is told the rating is already final and must only write `headline`/`top_risks`/`narrative` around it; `Parse Summary JSON` merges the AI's narrative onto the deterministic fields (spread first, so a malformed AI response can never corrupt the rating, only lose the prose). This makes the go/no-go signal reproducible and auditable instead of an LLM's per-run subjective pick.

### 4. Multimodal visual diffing
For `test_type: ui` escalations, `Is UI Test?` routes to `Claude - Visual Diff` instead of the text-only `Claude - Semantic Diff`. It sends Claude the baseline and current screenshots as images (base64) alongside the test metadata and asks it to describe concretely what changed before judging pass/fail — turning a bare `screenshot_match_score` float into an actual explanation ("nav shifted right 12px", "promo banner text differs, expected per dynamic_fields"). Requires extending your external UI runner to return `baseline_screenshot_base64` / `current_screenshot_base64`.

### 5. Test-as-code alternative source
`Route Test Source` checks `test_source` (`"sheet"` by default, or `"git"` from a webhook body) and can load tests from `Load Test Suite (Git)` — a JSON file (same `test_cases[]` shape as `sample_test_suite.json`) fetched from a git raw URL — instead of the Google Sheet, via `Parse Git Test Suite`. Both paths converge on the same `Validate & Filter Tests` node and contract, so engineering teams that want PR-reviewed, version-controlled test definitions can use git while less technical QA authors keep using the Sheet — pick per run, or standardize on one.

### 6. Jira ticket de-duplication across runs
`Split Failure Clusters` computes a stable `cluster_label` (a hash of root-cause category + sorted test IDs). `Search Existing Jira Ticket` checks Jira for an open issue already carrying that label before `Ticket Exists?` decides: if one exists, `Comment on Existing Ticket` appends "still failing as of run X" instead of opening a duplicate; only a genuinely new cluster reaches `Create Jira Tickets`, which tags the new issue with the label so future runs recognize it.

## How the sample data exercises the agent

`sample_test_suite.json` / `.csv` is deliberately built to walk through every code path:

- **TC-001, TC-005, TC-007** — clean passes with dynamic fields (tokens, timestamps, IDs) correctly excluded → deterministic `PASS`, **no AI call**.
- **TC-003** — exercises the `_min` threshold assertion: `count_min: 5` is checked with `>=` against the actual `count` field (not exact-match, and not silently skipped) → deterministic `PASS`/`AMBIGUOUS` without an AI call needed for this field.
- **TC-002** — status/error-code match but the human-readable `message` wording changed → escalated to Claude to judge semantic equivalence (`STALE_EXPECTATION`/`PASS` if truly equivalent, `REGRESSION` if meaning changed). A `STALE_EXPECTATION` verdict here (single mismatched `message` field, non-critical, non-monetary) is a candidate for **auto-apply** if confidence ≥ 0.95.
- **TC-004** — seeded to demonstrate a **real regression** (discount math wrong) → Claude should classify `REGRESSION`, not a stale expectation. Even if Claude misjudged this as stale, `total` matches the protected-field pattern, so it can never auto-apply — it always lands in manual review.
- **TC-006** — marked `known_flaky: true`, models a test that intermittently fails on environment lag → exercises `FLAKY` classification; repeated across nightly runs it will also trip the chronic-flaky detector once `qa_test_run_history` has enough rows.
- **TC-008** — has **no exact expected value at all**, only a `message_semantic` description → always routes to AI, demonstrating pure semantic assertion support.
- **TC-009** — `test_type: db`, showing the DB execution branch.
- **TC-010** — `test_type: ui`, showing the external UI-runner delegation branch, a `_min` threshold assertion (`screenshot_match_score_min: 0.97`) against the runner's `screenshot_match_score` field, and (once the runner returns screenshots) the visual-diff path.
- **TC-011** — `active: false`, showing that disabled tests are correctly excluded from a run.
- **TC-012** — intentionally missing `endpoint`, showing the row-validation/skip path that keeps a malformed sheet row from crashing the run.

## Interpreting output

- **Slack** message: headline, deterministic risk rating and score, pass/fail/flaky/needs-review counts, run ID, short narrative, plus a line for chronically flaky tests and recurring cross-run issues when present. Critical alerts additionally `@here`-mention the channel.
- **Jira tickets**: one per failure *cluster* (grouped by root cause), containing the list of failing tests, their mismatches, and Claude's reasoning per test. A cluster that's still open from a prior run gets a comment, not a duplicate ticket — see `cluster_label`.
- **Webhook response** (CI/CD-triggered runs only): `{ run_id, risk_rating, risk_score, pass_count, fail_count, headline }` — use this to gate a deploy pipeline.
- **`qa_run_history` table**: query this for pass-rate and risk-score trends over time.
- **`qa_test_run_history` table**: per-test verdict history; what chronic-flaky and recurring-issue detection reads from.
- **`qa_suggested_updates` table**: every stale-expectation suggestion, `auto_applied = true` for the ones already written to the live sheet, `false` for ones awaiting a QA lead's review.

## Version notes

Node `typeVersion`s in the JSON reflect a recent n8n release. If your instance is older and refuses to import a node version, open that node, and n8n will typically offer to auto-downgrade it, or you can recreate the node manually using the `parameters` object as a reference — the logic and field names will still apply.

## Extending

See **"Extensibility"** in `ARCHITECTURE.md` for how to add new test types, swap the LLM provider, swap the ticketing system, or build a trend dashboard on top of the history tables.

## Safety notes

- The AI never decides *whether* to mutate state — every node that mutates state (Jira, Slack, Postgres, the Sheet) is a plain n8n node acting on **parsed, validated** AI output, never the AI itself calling a tool.
- Auto-apply is intentionally narrow: only single-field, ≥0.95-confidence, non-critical-priority, non-monetary stale-expectation fixes are written to the live sheet, and every one (auto-applied or not) is logged to `qa_suggested_updates` with `auto_applied` set accordingly — nothing is silently changed without a corresponding audit row. Tune or disable this entirely by editing the `AUTO_APPLY_CONFIDENCE` / `PROTECTED_FIELD_PATTERN` constants in `Aggregate Run Results`.
- Use synthetic QA accounts/data in your test suite (see `qa.regression@example.com` in the sample data) — avoid sending real user PII to the LLM (now including screenshots sent to `Claude - Visual Diff`; make sure UI test fixtures don't expose real user data on-screen).
