# RoleFinder Release Notes

## Latest Release: v0.5.1

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

Version 0.5.0 completes the externalization of scoring criteria and job discovery filters to the database. The AI evaluation system now dynamically constructs prompts from user-defined scoring dimensions stored in `candidate_profiles.target_criteria`, enabling per-user customization without workflow modifications. Job discovery filters are also partially externalized, with full externalization planned for v0.5.1.

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
- ⚠️ `timeRange`, `locationSearch`, and `aiWorkArrangementFilter` still hardcoded (planned for v0.5.1)

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
- **Remaining Hardcoded Filters**: `timeRange`, `locationSearch`, `aiWorkArrangementFilter` need externalization
- **Error Handling**: Comprehensive try/catch blocks for JSON parsing and API failures
- **Retry Logic**: Automatic retry for transient API failures
- **Error Notifications**: Alert system for workflow failures

**Current Mitigation:** Monitor n8n execution logs; most API failures logged to `errors` table without halting workflow.

### Performance & Cost

- Execution time: 7-8 minutes (40 companies, 120 jobs)
- Cost per run: ~$1.84/day (Apify $0.48, Claude $0.36, n8n hosting $1.00)

---

## Version History

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

### v0.6.0 (Planned)
- Externalize remaining Apify filters (`timeRange`, `locationSearch`, `aiWorkArrangementFilter`)
- Comprehensive error handling (try/catch in all Code nodes)
- Retry logic for API failures
- Error notification system
- Multi-user production deployment hardening

### v0.7.0 (Planned)
- Profile management UI/API
- Historical job tracking and deduplication
- Advanced filtering and scoring customization
- Workflow scheduling per profile

---

## Support

For issues or questions:
- Review `WORKFLOW_DOCUMENTATION_README.md` for troubleshooting
- Check workflow execution logs in n8n UI
- Query `errors` table for logged failures
- Refer to `TODO.md` for planned enhancements
