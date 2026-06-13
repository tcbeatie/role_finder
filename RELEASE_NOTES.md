# RoleFinder Release Notes

## Latest Release: v1.2.0

**Release Date:** June 12, 2026
**Branch:** `main`
**Status:** Production-Ready

### Overview

Version 1.2.0 is primarily a **maintenance release** recovering from breaking changes to the Apify `career-site-job-listing-api` actor, which renamed several output fields and removed `remote_derived` (legacy names are removed by the actor on **2026-06-22**). All field reads in the four production workflows were migrated to the current schema. The release also adds a profile activation toggle and carries over a few reliability/quality improvements from the live workflows.

### Key Changes

#### Apify breaking-change recovery (Loop Companies v5.3 + Loop Jobs v5.3)

The actor deprecated its legacy output field names. Both workflows now read the current names:

| Old (removed 2026-06-22) | New |
|--------------------------|-----|
| `ai_salary_minvalue` | `ai_salary_min_value` |
| `ai_salary_maxvalue` | `ai_salary_max_value` |
| `ai_salary_unittext` | `ai_salary_unit_text` |
| `locations_raw` | `locations` |
| `remote_derived` (field removed) | derived from `ai_work_arrangement` |

- ✅ **Loop Companies** "Save Apify data" — salary fields, `locations`, and remote status derived from `ai_work_arrangement`
- ✅ **Loop Jobs** "Message a model" prompt, "Format Job Card", and output mapping — same field migration

> ⚠️ `Validate_Apify_Data.json` is a non-production validation utility and still reads the deprecated fields — tracked in `TODO.md` with the 2026-06-22 deadline (patch or retire).

#### Main v6.2 (was v6.1) — profile activation toggle

- ✅ **Load Profiles** now filters out profiles with `status = Disabled` — disable a user without deleting their row
- ✅ Sub-workflow calls updated to Loop Companies v5.3 and Send Email v5.2

#### Loop Companies v5.3 — broadened discovery

- ✅ Removed the hardcoded `aiWorkArrangementFilter` from the Apify request; all work arrangements are now returned and judged downstream by AI `location_fit` scoring (re-introducing it as a per-user filter is tracked in TODO 3b)

#### Loop Jobs v5.3 — reliability

- ✅ AI call sets `maxTokens: 4096` (prevents truncated evaluation JSON on long job descriptions)
- ✅ Merge node uses `fuzzyCompare` for robust `job_id`↔`id` matching across string/number types

#### Send Email v5.2 — categorization tolerance

- ✅ The "strong" grouping in `Aggregate and Sort Jobs` now accepts `STRONG_MATCH` in addition to `GOOD_MATCH`

#### candidate_profile table

- ✅ New optional `status` column — set to `Disabled` to skip a profile; empty/any other value = active

---

## Previous Release: v1.1.0

**Release Date:** May 8, 2026
**Branch:** `3-send-no-jobs-today-email-if-no-jobs-found`
**Status:** Production-Ready

### Overview

Version 1.1.0 ships two improvements: per-job AI error handling in Loop Jobs (so a single Claude API failure no longer silently drops a job from the digest), and per-user opt-in "no matches today" email notifications in Send Email (so users know when nothing was found, rather than receiving no email at all).

### Key Changes

#### Loop Jobs v5.2 (11 nodes, was 9)

- ✅ **Model update** — upgraded from `claude-sonnet-4-5-20250929` to `claude-sonnet-4-6`
- ✅ **Log AI error** — captures the failed job's context and error message when `Message a model` or `Parse AI Response` raises an error
- ✅ **Insert error row** — persists the error to the `errors` table; loop then continues to the next job rather than halting

Error flow: `Message a model` (error output) or `Parse AI Response` (error output) → `Log AI error` → `Insert error row` → `Loop Over Jobs` (continue)

#### Send Email v5.1 (12 nodes, was 8)

Four new nodes added to create a no-matches notification path:

- ✅ **If no matches** — gates on `preferences.notifications.send_empty_digest`; fires when email queue is empty after the `If matches` false branch
- ✅ **Build NoMatch Email** — generates a minimal HTML digest with subject `📭 No new matches today — <date>`, a tip card, and workflow run ID in the footer
- ✅ **Send NoMatch Email** — SMTP delivery of the no-match notification
- ✅ **NoMatch Response** — returns `{ status: 'no_matches', email_sent: true, ... }` to parent workflow

Existing nodes renamed for clarity:
- `Build Email` → `Build Match Email`
- `Send email` → `Send Match Email`
- `If` → `If matches`

**Three execution paths (was two):**

| Path | Condition | Email sent | Status returned |
|------|-----------|-----------|----------------|
| Matches | Jobs found in queue | Match digest | `success` |
| No-matches-send | Empty queue + `send_empty_digest: true` | "No matches" notice | `no_matches` |
| No-matches-skip | Empty queue + `send_empty_digest: false` (or missing `workflow_run_id`) | None | `skipped` |

#### candidate_profile table

- ✅ New `preferences TEXT` column required — stores JSON with notification preferences
- Schema: `{ "notifications": { "send_empty_digest": true } }`

---

## Previous Release: v1.0.0

**Release Date:** February 20, 2026
**Branch:** `main`
**Status:** Production-Ready

### Overview

Version 1.0.0 (developed as v0.6.0) adds a post-run summary dashboard: after all profiles are processed each day, Main sends an admin email with grand totals across all profiles and persists a per-profile stats row to a new `run_reports` table. Send Email now returns full category counts to the parent, making historical run data complete and queryable.

### Key Changes

#### Main v6.1 (13 nodes, was 8)

Five new nodes added to the done-branch and per-profile completion path:

- ✅ **Set Start Time** — records per-profile iteration start timestamp; fan-out hub to three downstream nodes
- ✅ **Build Run Report** — assembles one stats row per profile from Send Email's return value and the start timestamp (computes `run_duration_ms`)
- ✅ **Save Run Report** — persists the row to the `run_reports` data table and triggers the next Loop Over Profiles iteration
- ✅ **Build Summary Email** — fires once after all profiles complete (done branch); aggregates grand totals across all profiles and renders an HTML admin digest with per-profile table, five stat cards, and run duration
- ✅ **Send an Email** — delivers the admin summary via SMTP

#### Send Email v4.1 (8 nodes, unchanged count)

- ✅ **Full category counts** — `Success Response` now returns `strong_count`, `consider_count`, and `poor_count` in addition to `total_jobs` and `excellent_count`; `Empty Response` (skip path) returns zeros for all five fields
- These counts flow to `Build Run Report` in Main, populating all columns in `run_reports`

#### New: `run_reports` data table

- ✅ Schema template at `tables/template_run_reports.csv`
- Columns: `main_execution_id`, `profile_id`, `full_name`, `status`, `email_sent`, `total_jobs`, `excellent_count`, `strong_count`, `consider_count`, `poor_count`, `workflow_run_id`, `message_id`, `run_duration_ms`, `completed_at`
- One row per profile per daily run; `main_execution_id` links rows from the same orchestrator execution

---

## Previous Release: v0.5.1

**Release Date:** February 9, 2026
**Status:** Production-Ready

### Overview

Version 0.5.1 adds dual-path execution to Send Email workflow, preventing workflow hangs in multi-profile scenarios.

### Key Changes

#### Send Email v3.2 (8 nodes)
- ✅ Email queue validation gate (If node) — skips email when no jobs evaluated this run
- ✅ Dual-path execution: success path + skip path
- ✅ Structured status responses prevent parent workflow hangs
- ✅ Dynamic recipient from profile data (Start[1])

---

## Previous Release: v0.5.0

**Release Date:** February 7, 2026
**Branch:** `35-remove-hard-coded-scoring-criteria`
**Status:** Production-Ready

### Overview

Version 0.5.0 completes the externalization of scoring criteria and job discovery filters to the database. The AI evaluation system now dynamically constructs prompts from user-defined scoring dimensions stored in `candidate_profiles.target_criteria`, enabling per-user customization without workflow modifications. Job discovery filters are also partially externalized; full externalization was deferred and ultimately superseded in v1.2.0 (`aiWorkArrangementFilter` was removed; `timeRange`/`locationSearch` refactor tracked in TODO 3b).

### Key Changes

#### Dynamic AI Scoring Criteria (NEW in v0.5.0)
- ✅ AI scoring dimensions externalized to `target_criteria.ai_scoring_criteria`
- ✅ Loop_Jobs.json (v5.1) dynamically constructs AI prompts from scoring dimensions
- ✅ Per-user scoring criteria without workflow modifications
- ✅ "Add Scoring Criteria" node parses dimensions and builds evaluation prompt
- ✅ Supports custom dimension names, descriptions, and weights

#### Dynamic Job Discovery Filters (PARTIAL in v0.5.0)
- ✅ `titleSearch` and `titleExclusionSearch` externalized to `target_criteria.apify_filters`
- ✅ Loop_Companies.json (v5.1) dynamically builds Apify API requests
- ⚠️ `timeRange`, `locationSearch`, and `aiWorkArrangementFilter` still hardcoded (later superseded in v1.2.0 — see TODO 3b)

#### Profile Externalization (from v0.4.1)
- ✅ Candidate profile data moved from workflow nodes to `candidate_profiles` database table
- ✅ Profile loaded once at workflow start, passed via `_context_*` fields
- ✅ Workflow JSON files no longer contain sensitive personal information
- ✅ Foundation for multi-user support

#### Company Management (from v0.4.1)
- ✅ Company list moved to database with profile-based filtering
- ✅ Dynamic company-to-profile relationships via foreign keys
- ✅ Add/remove companies without workflow edits

#### Workflow Validation
All four workflows validated using n8n MCP tools (0 critical errors):
- Main.json (v4.2) - Production-ready (8 nodes)
- Loop_Companies.json (v5.1) - Production-ready (15 nodes)
- Loop_Jobs.json (v5.1) - Production-ready (9 nodes)
- Send_Email.json (v3.2) - Production-ready (8 nodes)

### Database Schema

**Updated Table: `candidate_profiles`**
```sql
- profile_id (string, PK)      -- Unique identifier
- full_name (string)            -- Display name for email
- email (string)                -- Email address for job digest
- resume_text (text)            -- Resume for AI evaluation
- target_criteria (text/JSON)   -- Job search criteria + AI scoring (NEW structure in v0.5.0)
- notes (text)                  -- Additional notes
```

**NEW in v0.5.0: `target_criteria` JSON Structure**
```json
{
  "apify_filters": {
    "titleSearch": ["product manager"],           // Job title filters
    "titleExclusionSearch": ["marketing"],        // Title exclusions
    "timeRange": "7d",                            // Search window (to be externalized)
    "locationSearch": ["United States"],          // Geographic filters (to be externalized)
    "aiWorkArrangementFilter": ["Remote OK"]      // Work arrangement (to be externalized)
  },
  "ai_scoring_criteria": {
    "version": "1.0",                             // Schema version
    "scoring_model": "weighted_average",          // Calculation method
    "dimensions": [                               // Dynamic scoring dimensions
      {
        "name": "seniority_match",
        "description": "Senior/Lead/Principal level roles",
        "weight": 1.0
      }
      // ... additional dimensions
    ]
  }
}
```

**Table: `companies`** (from v0.4.1)
```sql
- profile_id (string, FK)       -- Links companies to profiles
- company_id (string, PK)
- company_name (string)
- domain (string)
```

### Known Limitations

**Completed in v0.5.0:**
- ✅ **Scoring Criteria**: Now externalized to `target_criteria.ai_scoring_criteria`
- ✅ **Dynamic AI Prompts**: Automatically constructed from scoring dimensions
- ✅ **Title Filters**: `titleSearch` and `titleExclusionSearch` externalized

**Deferred to v0.6.0:**
- **Remaining Hardcoded Filters**: `timeRange`, `locationSearch`, `aiWorkArrangementFilter` need externalization (superseded in v1.2.0 — `aiWorkArrangementFilter` removed; the rest tracked in TODO 3b)
- **Error Handling**: Comprehensive try/catch blocks for JSON parsing and API failures
- **Retry Logic**: Automatic retry for transient API failures
- **Error Notifications**: Alert system for workflow failures

**Current Mitigation:** Monitor n8n execution logs; most API failures logged to `errors` table without halting workflow.

### Performance & Cost

- Execution time: 7-8 minutes (40 companies, 120 jobs)
- Cost per run: ~$1.84/day (Apify $0.48, Claude $0.36, n8n hosting $1.00)

---

## Version History

### v1.2.0 (2026-06-12)

**Changed:**
- Main.json: v6-1 → v6-2 — Load Profiles filters out `status = Disabled` profiles; calls Loop Companies v5.3 + Send Email v5.2
- Loop_Companies.json: v5.2 → v5.3 — Apify field migration (`ai_salary_*_value`/`_unit_text`, `locations`, remote via `ai_work_arrangement`); removed the hardcoded `aiWorkArrangementFilter` (broadened discovery)
- Loop_Jobs.json: v5.2 → v5.3 — Apify field migration (AI prompt + Format Job Card + output); added `maxTokens: 4096`; Merge node `fuzzyCompare: true`
- Send_Email.json: v5.1 → v5.2 — "strong" grouping accepts `STRONG_MATCH` || `GOOD_MATCH`
- `candidate_profile` table: new optional `status` column (`Disabled` skips a profile)
- README.md: version references (v6.2 / v5.3 / v5.2), profile `status` filter docs + schema, Apify recovery + broadened-discovery + reliability notes
- TODO.md: reframed item 3b (`aiWorkArrangementFilter` is now a refactor against the new array input, not a simple externalize); added dated High-Priority item to migrate or retire `Validate_Apify_Data.json` before 2026-06-22

**Fixed:**
- Apify `career-site-job-listing-api` breaking changes — renamed salary/location output fields and the removed `remote_derived` would return null/undefined after 2026-06-22; the four production workflows now read the current field names

**Known issues:**
- `Validate_Apify_Data.json` (non-production utility) still reads deprecated Apify fields — tracked in `TODO.md`

---

### v1.1.0 (2026-05-08)

**Added:**
- `Log AI error` node in Loop Jobs — captures error context when AI call or JSON parse fails
- `Insert error row` node in Loop Jobs — persists per-job AI failures to `errors` table; loop continues
- `If no matches` node in Send Email — gates no-match email on `preferences.notifications.send_empty_digest`
- `Build NoMatch Email` node in Send Email — "no new matches today" HTML digest with tip card
- `Send NoMatch Email` node in Send Email — SMTP delivery of no-match notification
- `NoMatch Response` node in Send Email — returns `{ status: 'no_matches', email_sent: true }` to parent
- `preferences TEXT` column in `candidate_profile` table — JSON field for per-user notification settings

**Changed:**
- Loop_Jobs.json: Upgraded from v5.1 to v5.2 (9 → 11 nodes)
- Loop Jobs AI model: `claude-sonnet-4-5-20250929` → `claude-sonnet-4-6`
- Send_Email.json: Upgraded from v4.1 to v5.1 (8 → 12 nodes)
- `Build Email` → `Build Match Email` (renamed for clarity)
- `Send email` → `Send Match Email` (renamed for clarity)
- `If` → `If matches` (renamed for clarity)
- Send Email now has three execution paths (was two); false branch of `If matches` routes to `If no matches` instead of directly to `Empty Response`
- README.md: Updated all version references, workflow descriptions, status response schema, `candidate_profile` table schema, setup example

**Fixed:**
- AI evaluation failures (Claude API error or malformed JSON response) previously silently dropped a job from the digest with no record; now logged to `errors` table

---

### v1.0.0 (2026-02-20)

**Added:**
- `run_reports` data table for per-profile run history (schema: `tables/template_run_reports.csv`)
- `Set Start Time` node in Main — per-profile start timestamp and fan-out hub
- `Build Run Report` node in Main — assembles stats row from Send Email return + start timestamp
- `Save Run Report` node in Main — persists row to `run_reports`, advances loop
- `Build Summary Email` node in Main — post-loop admin HTML digest with grand totals across all profiles
- `Send an Email` node in Main — SMTP delivery of admin summary

**Changed:**
- Main.json: Upgraded from v5-1 to v6-1 (8 → 13 nodes)
- Send_Email.json: Upgraded from v3.2 to v4.1 (8 nodes; node count unchanged)
- `Success Response` in Send Email now returns `strong_count`, `consider_count`, `poor_count`
- `Empty Response` in Send Email now returns zeros for all five category count fields
- README.md: Updated all version references, architecture diagram, node counts, database tables list, status response schemas, documentation section, and project status

**Fixed:**
- `strong_count`, `consider_count`, `poor_count` previously missing from Send Email's return value, leaving them unavailable to parent workflow for reporting

### v0.5.1 (2026-02-09)

**Added:**
- Email queue result validation (If node) — skips send when queue is empty for this run
- Success Response and Empty Response nodes for structured status
- Dual-path execution architecture (prevents workflow hangs)

**Changed:**
- Send_Email.json: Upgraded from v2.3 to v3.2 (5→8 nodes)
- Dynamic recipient addressing from Start node item [1]
- README.md: Documented dual-path status response pattern
- README.md: Updated Send Email to 8 nodes throughout

**Fixed:**
- Multi-profile workflow hangs when no jobs found for a profile
- Loop Over Profiles continuation in Main workflow

### v0.5.0 (2026-02-07)

**Added:**
- `ai_scoring_criteria` section in `target_criteria` JSON field
- "Add Scoring Criteria" node in Loop_Jobs.json (v5.1) for dynamic prompt construction
- Dynamic AI prompt generation from user-defined scoring dimensions
- Partial `apify_filters` externalization (`titleSearch`, `titleExclusionSearch`)
- Comprehensive README.md documentation of dynamic scoring system
- Updated CLAUDE.md with complete target_criteria schema

**Changed:**
- Loop_Jobs.json: Added node to parse scoring criteria and build dynamic prompts
- Loop_Companies.json: Build Request node now reads `titleSearch`/`titleExclusionSearch` from target_criteria
- README.md: Expanded AI Evaluation Criteria section with full JSON examples
- README.md: Updated workflow versions to v5.1 throughout documentation

**Improved:**
- Per-user scoring criteria customization without workflow modifications
- Multi-user support with different evaluation dimensions per profile
- A/B testing capability for scoring methodologies
- Version tracking for scoring criteria evolution

**Documentation:**
- Complete target_criteria JSON structure with both sections documented
- Setup examples updated with full schema

### v0.4.1 (2026-02-05)

**Added:**
- `candidate_profiles` database table for profile externalization
- Profile loading in Main.json workflow
- Profile context propagation via `_context_*` fields
- Per-profile company filtering
- Comprehensive workflow validation using n8n MCP tools

**Changed:**
- Main.json: Added Load Profiles node and profile merging logic
- Loop_Companies.json: Updated to receive and propagate profile data
- Loop_Jobs.json: Changed to use `_context_resume_text` instead of hardcoded profile
- Send_Email.json: Updated recipient extraction to use profile data

**Removed:**
- Hardcoded resume text from Loop_Jobs.json
- Hardcoded email addresses from workflows
- Hardcoded job search criteria from workflows

**Fixed:**
- Profile context not available in AI evaluation
- Email delivery to incorrect/hardcoded addresses
- Company data management requiring workflow edits

**Security:**
- Removed sensitive personal data from workflow JSON files
- Enabled safe version control for workflows

---

## Roadmap

### v1.0.1 (Skipped)
Planned but never shipped — v1.1.0 shipped instead. Remaining items tracked in `TODO.md`.

### v2.0.0 (Planned)
- Resume uploading
- Advanced filtering and scoring customization
- Workflow scheduling per profile

---

## Support

For issues or questions:
- Check workflow execution logs in n8n UI
- Query `errors` table for logged failures
- Refer to `TODO.md` for planned enhancements
