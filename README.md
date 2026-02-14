# RoleFinder

**An intelligent, AI-powered job monitoring system that discovers, evaluates, and delivers personalized job opportunities via email digest.**

## Who This Project Is For

This repository is intended for **developers and technical users** who want to run and self-host their own job-tracking solution using RoleFinder.

**If you'd rather not manage the technical setup and ongoing maintenance**, see the [Managed Service Option](#managed-service-option) below for a fully hosted solution.

---

RoleFinder is a production-grade automation system built on n8n that monitors target companies for relevant job postings, evaluates each position using AI against your specific criteria, and delivers a professional email digest sorted by match quality. Designed by a senior technical product manager for use by anyone, it surfaces high-signal opportunities with minimal noise through intelligent filtering and multi-dimensional scoring.

---

## Key Features

- 🎯 **Targeted Discovery**: Monitors companies via Apify Career Site API
- 🤖 **AI-Powered Evaluation**: AI scores jobs across 5 dimensions
- 📧 **Professional Digests**: HTML email with best matches first
- 🔄 **Fault Tolerant**: Company-by-company processing survives individual failures
- 💾 **Complete Tracking**: Raw job data, AI evaluations, and email history preserved
- 📊 **Cost Efficient**: ~$1.84/day
- 🏗️ **Production Ready**: Comprehensive documentation, error handling, monitoring queries

---

## Managed Service Option

**Don't want to self-host?** A fully managed RoleFinder service is available where all infrastructure, setup, and maintenance is handled for you.

### What's Included

- **Zero-Touch Setup**: No n8n installation, API configuration, or workflow imports
- **Infrastructure Management**: Hosting, monitoring, and uptime guaranteed
- **API Credential Handling**: Apify, Claude, and SMTP email integrations managed
- **Profile Customization**: Your resume, criteria, and target companies configured
- **Daily Monitoring**: Error resolution and workflow optimization
- **Ongoing Updates**: Automatic workflow improvements and feature additions
- **Dedicated Support**: Direct access for questions, adjustments, and troubleshooting

### Best For

- **Non-technical users** who want the benefits without the technical setup
- **Busy professionals** who value time over DIY cost savings (~4-6 hours setup + ongoing maintenance)
- **Reliability-focused users** who want guaranteed uptime and expert support
- **Anyone** who prefers hands-off automation that "just works"

### Self-Hosted vs. Managed

| Feature | Self-Hosted (This Repo) | Managed Service |
|---------|------------------------|-----------------|
| **Cost** | ~$55/month (API costs only) | Contact for pricing |
| **Setup Time** | 4-6 hours initial setup | 0 hours (done for you) |
| **Maintenance** | Your responsibility | Fully managed |
| **Support** | Best-effort via GitHub | Dedicated support channel |
| **Customization** | Full control over code | Configuration without code |
| **Updates** | Manual workflow updates | Automatic improvements |
| **Monitoring** | Self-service | Proactive error resolution |
| **Uptime** | Dependent on your setup | Guaranteed reliability |

### Get Started

Interested in the managed service? **Contact for pricing and availability:**

- **Email**: [finder@launchbridge.co](mailto:finder@launchbridge.co)
- **Website**: [https://www.launchbridge.co/rolefinder/](https://www.launchbridge.co/rolefinder/)
- **Response Time**: Within 24-48 hours

---

## Non-Goals

RoleFinder is purpose-built for a specific use case. To maintain focus and manage expectations, here's what it deliberately does **not** do:

- **Not a general-purpose job scraper** - Monitors specific target companies, not every job board or career site
- **Not real-time alerting** - Runs on scheduled intervals, not instant push notifications when jobs are posted
- **Not guaranteed to catch every posting** - Depends on data quality, company career page updates, and API timing
- **Not a replacement for human judgment** - AI provides scoring and recommendations, but final application decisions remain yours
- **Not an application tracker/CRM** - Doesn't manage applications, interview schedules, follow-ups, or offer negotiations
- **Not a resume builder or optimizer** - Doesn't generate, rewrite, or improve resumes (uses your existing profile for evaluation)
- **Not a LinkedIn/Indeed replacement** - Designed to complement, not replace, traditional job search methods
- **Not infinitely scalable without cost** - Each job evaluation incurs API costs (Apify + Claude); monitoring 500+ companies daily gets expensive

If you're looking for these features, this project may not be the right fit. Issues requesting non-goal features will be closed with a reference to this section.

---

## Architecture Overview

The following section describes the internal workflows for advanced users who want to understand or operate the system.

RoleFinder uses a four-workflow pipeline with a top-level orchestrator that separates concerns for clean architecture. The system features externalized profile management stored in the database, enabling multi-user support and eliminating hardcoded personal data from workflow files.

```
┌─────────────────────────────────────────────────────────────┐
│  MAIN v6.1 (Orchestrator)                                   │
│  Purpose: Load profiles and orchestrate multi-user pipeline │
│  Input:   Manual/scheduled trigger                          │
│  Actions: 1. Load Profiles from candidate_profile table     │
│           2. Loop Over Profiles (process each sequentially) │
│           3. For each profile:                              │
│              a. Load Companies (filtered by profile_id)     │
│              b. Merge Profile+Companies                     │
│              c. Call Loop Companies v5.1                    │
│              d. Merge Profile+Done                          │
│              e. Call Send Email v4.1                        │
│              f. Save run report row to run_reports table    │
│           4. After all profiles: send admin summary email   │
└────┬────────────────────────────────────────────────────┬───┘
     │ Passes profile+companies array ↓                   │
┌─────────────────────────────────────────────────────────────┐
│  WORKFLOW 1: Loop Companies v5.1                            │
│  Purpose: Discover jobs from target companies               │
│  Input:   Profile+companies array from Main (concatenated)  │
│  Output:  Raw jobs → database, enriched jobs → Loop Jobs    │
│  Key:     Dynamic Apify filters from target_criteria        │
└────────────────┬────────────────────────────────────────────┘
                 │ Calls for each company ↓
┌─────────────────────────────────────────────────────────────┐
│  WORKFLOW 2: Loop Jobs v5.1                                 │
│  Purpose: AI evaluation and HTML card generation            │
│  Input:   Jobs with _context_* fields (incl. profile data)  │
│  Output:  Formatted job cards → email_queue table           │
│  Key:     Dynamic AI scoring from ai_scoring_criteria       │
└─────────────────────────────────────────────────────────────┘
     │ When all companies complete, Main calls ↓          │
┌─────────────────────────────────────────────────────────────┐
│  WORKFLOW 3: Send Email v4.1                                │
│  Purpose: Aggregate and deliver email digest                │
│  Input:   workflow_run_id + email from Main                 │
│  Output:  Professional HTML email via SMTP + status         │
└─────────────────────────────────────────────────────────────┘
```

### Profile Externalization Architecture

Candidate profiles are stored in the `candidate_profile` database table, not hardcoded in workflows. This enables:

- **Multi-user support**: Multiple users can share the same workflows with different profiles
- **Clean workflow files**: No personal data (resume, criteria) in workflow JSON files
- **Easy updates**: Change profile in database without modifying workflows
- **Security**: Profile data can be encrypted at rest, separately from workflow definitions

**Data Flow**:
1. Main workflow loads profile from `candidate_profile` table at startup
2. Profile passed to Loop Companies via sub-workflow execution
3. Loop Companies merges profile with each company in "Merge Data" node
4. Profile passed to Loop Jobs via `_context_resume_text` and `_context_target_criteria` fields
5. Loop Jobs uses profile data for AI evaluation without storing it in workflow
6. Main workflow calls Send Email via sub-workflow execution

### Multi-Profile Architecture (Main v6.1)

Single workflow run can process multiple users sequentially.

**Architecture**:
1. **Load Profiles** node retrieves ALL profiles from `candidate_profile` table
2. **Loop Over Profiles** node iterates through each profile (batch size: 1)
3. For each profile iteration:
   - **Load Companies** filters companies WHERE profile_id = current profile
   - **Merge Profile+Companies** creates concatenated array [profile, ...companies]
   - **Call Loop Companies** processes all companies for this profile
   - **Merge Profile+Done** combines completion data with profile
   - **Call Send Email** delivers personalized digest to profile's email
   - **Save Run Report** persists per-profile stats to `run_reports` table
4. Loop advances to next profile after run report is saved
5. After all profiles complete: **Build Summary Email** sends an admin digest with grand totals across all profiles

**Benefits**:
- **No workflow duplication**: One set of workflows serves all users
- **Automatic scaling**: Add users by adding database rows
- **Profile isolation**: Each user's companies and criteria are independent
- **Personalized digests**: Each email uses that profile's resume and criteria
- **Simple maintenance**: Update workflows once, all users benefit

### Workflow Details

**Main v6.1 (13 nodes)** - Top-level orchestrator with multi-profile support and run reporting
- Entry point via manual trigger or cron schedule
- Loads ALL candidate profiles from `candidate_profile` database table
- Loops over each profile sequentially (enables multi-user support)
- For each profile:
  - Records iteration start time (`Set Start Time`)
  - Loads companies filtered by profile_id
  - Merges profile with companies into concatenated array
  - Calls Loop Companies v5.1 sub-workflow (passes profile+companies)
  - Waits for all companies to complete processing
  - Merges profile with completion data
  - Calls Send Email v4.1 sub-workflow (passes workflow_run_id and profile)
  - Builds a per-profile run report row (`Build Run Report`)
  - Saves run report to `run_reports` data table (`Save Run Report`)
- After all profiles complete: generates and sends an admin summary email (`Build Summary Email`) with grand totals across all profiles
- Single workflow run can process multiple users

**Loop Companies v5.1 (15 nodes)** - Job discovery
- Receives concatenated profile+companies array from Main workflow
- Companies are pre-filtered by profile_id in Main's Load Companies node
- Splits array: extracts profile (index 0) and companies (index 1+)
- Merges profile fields into each company object
- Iterates one company at a time (fault tolerance)
- For each company:
  - Parses `target_criteria.apify_filters` to build API request
  - Calls Apify API to discover jobs (with dynamic filters)
  - Saves raw job data immediately to `jobs` table (backup path)
  - Enriches jobs with `_context_*` fields including profile data
  - Calls Loop Jobs v5.1 sub-workflow with enriched job data
- Handles API errors and no-results gracefully
- Returns summary to Main when all companies complete

**Loop Jobs v5.1 (9 nodes)** - AI evaluation with dynamic scoring
- Receives jobs with `_context_*` fields from Loop Companies
- Uses `_context_resume_text` from parent (no hardcoded profile)
- Parses `_context_target_criteria.ai_scoring_criteria` for dynamic prompt construction
- Iterates one job at a time (AI rate limiting + error isolation)
- For each job:
  - Builds AI prompt dynamically from scoring dimensions
  - Sends job + profile + criteria to Claude Sonnet 4.5 for evaluation
  - Parses structured JSON response (scores, reasoning, recommendation)
  - Merges AI evaluation with complete job data (parallel paths)
  - Generates professional HTML job card
  - Saves to `email_queue` table with `workflow_run_id`
- Returns to Loop Companies when all jobs complete

**Send Email v4.1 (8 nodes)** - Email delivery with status responses
- Receives `workflow_run_id` and `email` from Main workflow
- Email address dynamically extracted from profile data
- **Validates email queue has results before processing (If node)**
- **Two execution paths for reliability:**
  - **Success path**: Jobs found in queue → aggregate → email sent → success status
  - **Skip path**: Empty queue (no jobs evaluated this run) → skip email → skip status
- Queries `email_queue` for all jobs from this workflow run
- Aggregates and sorts by AI score (best first)
- Groups by recommendation category (excellent/good/consider)
- Generates dynamic subject line based on quality
- Builds professional HTML email with statistics
- Sends via SMTP (migrated from Gmail API for improved reliability)
- **Returns structured status object to parent workflow**
- **Ensures Main workflow loop continues even with empty results**

### Status Response Pattern (v4.1)

Send Email now implements dual-path execution with consistent status responses, ensuring the Main workflow's Loop Over Profiles never hangs:

**Success Path** (when jobs are found):
```json
{
  "status": "success",
  "email_sent": true,
  "total_jobs": 20,
  "excellent_count": 3,
  "strong_count": 8,
  "consider_count": 6,
  "poor_count": 3,
  "workflow_run_id": 67890,
  "timestamp": "2025-02-09T14:30:00.000Z",
  "message_id": "<abc123@mail.gmail.com>"
}
```

**Skip Path** (when email queue is empty for this workflow run):
```json
{
  "status": "skipped",
  "reason": "no_workflow_run_id",
  "email_sent": false,
  "total_jobs": 0,
  "excellent_count": 0,
  "strong_count": 0,
  "consider_count": 0,
  "poor_count": 0,
  "workflow_run_id": null,
  "timestamp": "2025-02-09T14:30:00.000Z"
}
```

**Architecture Benefits:**
- **Loop continuation**: Both paths return output, preventing workflow hangs
- **Consistent schema**: Parent workflow can safely handle either response
- **Debugging clarity**: Status field makes execution outcome explicit
- **Multi-profile support**: Essential for Loop Over Profiles to process multiple users

---

## Technology Stack

- **Orchestration**: n8n (workflow automation platform)
- **Job Discovery**: Apify Career Site Job Listing API
- **AI Evaluation**: Anthropic Claude Sonnet 4.5
- **Email Delivery**: SMTP (standard email protocol)
- **Data Storage**: n8n Data Tables (built-in)
- **Deployment**: n8n Cloud or self-hosted (Docker)

---

## Data Flow & Traceability

Every job is traceable through the entire pipeline via workflow_run_id:

1. **Discovery**: Loop Companies discovers job
2. **Context**: Enriched with `_context_workflow_run_id`
3. **Evaluation**: Loop Jobs evaluates with AI, saves to email_queue
4. **Delivery**: Send Email queries by workflow_run_id, includes in digest
5. **Tracking**: Email footer shows workflow_run_id for reference

Database tables:
- `candidate_profile` - User profiles with resume and target criteria
- `companies` - Target companies to monitor
- `jobs` - Raw job data with searchable fields
- `email_queue` - Formatted job cards linked by workflow_run_id
- `errors` - API failures and no-results tracking
- `run_reports` - Per-profile run stats (counts, timing, email status) saved by Main after each Send Email call; see `tables/template_run_reports.csv` for schema

---

## AI Evaluation Criteria

RoleFinder uses **dynamic, profile-specific AI scoring** powered by the `target_criteria` JSON field in the `candidate_profile` table. The system automatically constructs evaluation prompts from your custom dimensions.

### Scoring Architecture

**Two-Section Configuration:**

1. **Apify Filters** (`apify_filters`): Job discovery parameters
   - `titleSearch`: Required words in job title (e.g., `["product manager"]`)
   - `titleExclusionSearch`: Words that disqualify jobs (e.g., `["marketing"]`)
   - `timeRange`: How far back to search (`"7d"`, `"30d"`, `"6m"`)
   - `locationSearch`: Geographic filters (e.g., `["United States", "Canada"]`)
   - `aiWorkArrangementFilter`: Work arrangement types (e.g., `["Remote OK", "Hybrid"]`)

2. **AI Scoring Criteria** (`ai_scoring_criteria`): Dynamic evaluation dimensions
   - `version`: Schema version for future evolution (currently `"1.0"`)
   - `scoring_model`: Calculation method (currently `"weighted_average"`)
   - `dimensions`: Array of scoring dimensions with:
     - `name`: Dimension identifier (used in AI response)
     - `description`: Human-readable criteria sent to AI
     - `weight`: Importance multiplier (all `1.0` for equal weighting)

### Example Configuration (Senior Technical Product Manager)

```json
{
  "apify_filters": {
    "titleSearch": ["product"],
    "titleExclusionSearch": ["marketing"],
    "timeRange": "7d",
    "locationSearch": ["United States", "Canada", "United Kingdom"],
    "aiWorkArrangementFilter": ["Hybrid", "Remote OK", "Remote Solely"]
  },
  "ai_scoring_criteria": {
    "version": "1.0",
    "scoring_model": "weighted_average",
    "dimensions": [
      {
        "name": "seniority_match",
        "description": "Senior/Lead/Principal/Director level roles (not Associate/Junior)",
        "weight": 1.0
      },
      {
        "name": "domain_match",
        "description": "Infrastructure, Platform Engineering, Developer Experience, API platforms, Cloud systems",
        "weight": 1.0
      },
      {
        "name": "technical_depth",
        "description": "Backend systems, integrations, developer tools expertise required",
        "weight": 1.0
      },
      {
        "name": "location_fit",
        "description": "Remote, Hybrid NYC Metro, or Hybrid Bay Area (10=perfect, 0=dealbreaker)",
        "weight": 1.0
      },
      {
        "name": "compensation",
        "description": "$190k+ is threshold (10 if well above, 0 if below)",
        "weight": 1.0
      }
    ]
  }
}
```

### How It Works

1. **Loop Companies** workflow parses `apify_filters` to build job discovery API requests
2. **Loop Jobs** workflow parses `ai_scoring_criteria` to dynamically construct AI evaluation prompts
3. Claude AI scores each job 1-10 across your custom dimensions
4. Overall score calculated using `scoring_model` (weighted average of dimension scores)
5. Recommendation category assigned based on overall score

### Recommendation Categories

- **EXCELLENT_MATCH**: 8-10 score, strong fit across dimensions
- **GOOD_MATCH**: 6-8 score, solid candidate with minor gaps
- **CONSIDER**: 4-6 score, worth reviewing despite gaps
- **POOR_FIT**: 1-4 score, significant mismatches

### Customization Benefits

✅ **Per-User Criteria**: Different users can have completely different scoring dimensions
✅ **No Workflow Changes**: Update criteria in database without modifying workflows
✅ **A/B Testing**: Test different dimension weights or descriptions
✅ **Version Tracking**: `version` field enables methodology tracking
✅ **Future Flexibility**: Can add new scoring models (threshold-based, knockout criteria, etc.)

---

## Sample Email Output

```
┌──────────────────────────────────────────────────────────┐
│ 🎯 Your Daily Job Digest                                 │
│ Monday, December 29, 2025                                │
└──────────────────────────────────────────────────────────┘

Subject: 🎯 15 Excellent Matches, 45 Strong • 120 Total Jobs

┌──────────┬──────────┬───────────┬─────────┐
│ Excellent│  Strong  │  Consider │  Total  │
│    15    │    45    │     40    │   120   │
└──────────┴──────────┴───────────┴─────────┘

[Job Card 1 - Score 10/10 - Excellent Match]
Senior Technical Product Manager - Anthropic
$200,000 - $250,000 • San Francisco, CA • 🏠 Remote
AI Assessment: Perfect match - infrastructure focus...
✅ Strengths: Senior level, API platform expertise...

[Job Card 2 - Score 9/10 - Excellent Match]
...
```

---

## Cost Analysis (Example)

**Daily Breakdown (monitoring 40 companies, ~120 jobs/day):**
- Apify API: ~$0.48 (120 jobs × $0.004)
- Claude AI: ~$0.36 (120 jobs × $0.003)
- SMTP: $0.00 (typically free tier for most providers)
- n8n hosting: ~$1.00/day (varies by provider)
- **Total: ~$1.84/day** (~$55/month, ~$671/year)

**Note**: Your actual costs will vary based on the number of companies monitored and their job posting frequency.

**ROI:**
- Time saved: 2-3 hours/day of manual searching
- Your hourly value: $100-150/hour (based on target comp)
- Daily value: $200-450 in time saved

---

## Key Design Principles

**1. Separation of Concerns**
- Discovery (Loop Companies) ≠ Evaluation (Loop Jobs) ≠ Delivery (Send Email)
- Each workflow testable independently
- Clear workflow boundaries and responsibilities

**2. Fault Tolerance**
- Company-by-company processing (one failure doesn't stop others)
- Job-by-job evaluation (AI rate limiting, error isolation)
- Error logging at every level with continuation

**3. Data Preservation**
- Raw jobs saved immediately (before AI evaluation)
- Complete job data preserved through processing
- Email queue accumulates formatted cards
- Historical tracking via workflow_run_id

**4. Clean Architecture**
- Sub-workflow pattern for modularity
- Context fields pass data across workflow boundaries
- Database-driven accumulation (not in-memory state)
- Parallel processing where beneficial (backup + evaluation)

---

## Documentation

📚 **Complete documentation available in repository:**

- **README.md** - You're looking at it!
- **Main.json** - Main orchestrator v6.1 (13 nodes, multi-profile support + run reporting + admin summary email)
- **Loop_Companies.json** - Workflow 1 v5.1 (15 nodes, dynamic Apify filters)
- **Loop_Jobs.json** - Workflow 2 v5.1 (9 nodes, dynamic AI scoring)
- **Send_Email.json** - Workflow 3 v4.1 (8 nodes, dual-path status responses with full category counts)
- **tables/template_run_reports.csv** - Schema template for the `run_reports` data table

Each workflow JSON includes comprehensive inline comments suitable for junior developer handoff.

**Key Architecture Innovations (v5.1):**
- **Dynamic Apify Filters**: Job discovery parameters externalized to `target_criteria.apify_filters`
- **Dynamic AI Scoring**: Evaluation dimensions externalized to `target_criteria.ai_scoring_criteria`
- **Per-User Customization**: Different users can have completely different filters and scoring criteria
- **No Workflow Changes**: Update criteria in database without modifying workflows

---

## Quick Start

### Prerequisites
- n8n instance (cloud or self-hosted)
- Apify account with Career Site API access
- Anthropic API key (Claude access)
- SMTP email account (any provider)

### Setup Steps

1. **Import Workflows**
   ```bash
   # Import in this order (sub-workflows must exist before their parent):
   # 1. Loop_Jobs.json (v5.1) - AI evaluation with dynamic scoring
   # 2. Loop_Companies.json (v5.1) - Job discovery with dynamic filters
   # 3. Send_Email.json (v4.1) - Email delivery with full category counts
   # 4. Main.json (v6.1) - Multi-profile orchestrator with run reporting
   ```

2. **Configure Credentials**
   - Apify OAuth2: Connect account
   - Anthropic API: Add API key
   - SMTP: Configure email credentials (username/password)

3. **Setup Database Tables**
   ```sql
   -- Create required tables
   CREATE TABLE candidate_profile (
     profile_id VARCHAR(50) PRIMARY KEY,
     full_name VARCHAR(100),
     email VARCHAR(255),
     resume_text TEXT,
     target_criteria TEXT,
     notes TEXT
   );

   CREATE TABLE companies (
     profile_id VARCHAR(50),           -- links to candidate_profile
     company_id VARCHAR(50),
     company_name VARCHAR(100),
     domain VARCHAR(100),
     PRIMARY KEY (profile_id, company_id)
   );

   CREATE TABLE jobs (...);  -- See documentation
   CREATE TABLE email_queue (...);
   CREATE TABLE errors (...);
   CREATE TABLE run_reports (...);  -- See tables/template_run_reports.csv for schema

   -- Add indexes
   CREATE INDEX idx_email_queue_workflow ON email_queue(workflow_run_id);
   ```

4. **Populate Candidate Profile**
   ```sql
   -- Add your profile data with complete target_criteria
   INSERT INTO candidate_profile VALUES (
     'default',                    -- profile_id
     'Your Name',                  -- full_name
     'your.email@example.com',     -- email (used for digest delivery)
     'Your complete resume/experience text here...',  -- resume_text
     '{
       "apify_filters": {
         "titleSearch": ["product"],
         "titleExclusionSearch": ["marketing"],
         "timeRange": "7d",
         "locationSearch": ["United States", "Canada", "United Kingdom"],
         "aiWorkArrangementFilter": ["Hybrid", "Remote OK", "Remote Solely"]
       },
       "ai_scoring_criteria": {
         "version": "1.0",
         "scoring_model": "weighted_average",
         "dimensions": [
           { "name": "seniority_match", "description": "Senior/Lead/Principal/Director level roles (not Associate/Junior)", "weight": 1.0 },
           { "name": "domain_match", "description": "Infrastructure, Platform Engineering, Developer Experience, API platforms, Cloud systems", "weight": 1.0 },
           { "name": "technical_depth", "description": "Backend systems, integrations, developer tools expertise required", "weight": 1.0 },
           { "name": "location_fit", "description": "Remote, Hybrid NYC Metro, or Hybrid Bay Area (10=perfect, 0=dealbreaker)", "weight": 1.0 },
           { "name": "compensation", "description": "$190k+ is threshold (10 if well above, 0 if below)", "weight": 1.0 }
         ]
       }
     }',  -- target_criteria (JSON with job discovery filters and dynamic AI scoring)
     'Optional notes'              -- notes
   );
   ```

   **Note**: The `target_criteria` field contains two sections:
   - **apify_filters**: Job discovery parameters (what jobs to find)
   - **ai_scoring_criteria**: Dynamic evaluation dimensions (how to score jobs)

   All scoring dimensions are customizable per profile. The Loop Jobs workflow dynamically constructs the AI prompt from these criteria, enabling per-user customization without workflow changes.

5. **Populate Companies**
   ```sql
   -- Add target companies (profile_id must match a row in candidate_profile)
   INSERT INTO companies (profile_id, company_id, company_name, domain) VALUES
     ('default', 'anthropic', 'Anthropic', 'anthropic.com'),
     ('default', 'openai', 'OpenAI', 'openai.com'),
     -- ... add more companies using the same profile_id
   ```

6. **Update Configuration**
   - Update `fromEmail` in two places: the "Send an Email" node in Main.json and the "Send email" node in Send_Email.json — change `sender@example.com` to your actual sending address
   - Recipient email is automatically sourced from candidate_profile table
   - Main v6.1 automatically processes all profiles (no filter needed)
   - To limit to specific profiles, add filter in Load Profiles node
   - Adjust scoring criteria in Loop Jobs v5.1 prompt if needed

7. **Test Execution**
   ```bash
   # Test with 3 companies first
   # 1. Run Main v6.1 manually
   # 2. Verify jobs in database
   # 3. Check email received
   # 4. Review formatting
   ```

8. **Schedule Daily Run**
   - Add cron trigger to Main v6.1: `0 6 * * *` (6 AM daily)
   - Monitor first week for issues
   - Review cost and performance

---

## Monitoring & Maintenance

### Daily Checks
- Verify email received each morning
- Check job count reasonable (80-150 typical)
- Review errors table for failures

### Weekly Review
```sql
-- Weekly statistics
SELECT 
  COUNT(*) as total_jobs,
  COUNT(DISTINCT company_name) as companies_with_jobs,
  AVG(overall_score) as avg_score
FROM email_queue
WHERE processed_at >= CURRENT_DATE - INTERVAL '7 days';
```

### Monthly Maintenance
- Clean old email_queue entries (>30 days)
- Archive historical job data
- Review and update target companies
- Check API quota usage
- Update candidate profile if needed

---

## Troubleshooting

### Common Issues

**No Email Received**
- Check Send Email execution log
- Verify SMTP credentials and server settings
- Check spam/junk folder
- Verify email address in candidate_profile table is correct

**Email Queue Empty**
- Check workflow_run_id matches
- Review Loop Jobs execution log
- Verify jobs discovered in jobs table

**AI Evaluation Fails**
- Re-authenticate Anthropic API
- Check job description length (<10k chars)
- Verify network connectivity

**Duplicate Jobs**
- Clear email_queue before runs
- Add unique constraint on (workflow_run_id, job_id)

---

## Extending RoleFinder

### Adding More Companies
Simply add rows to companies table - no workflow changes needed.

### Updating Your Profile
Update the `candidate_profile` table with new resume or criteria - no workflow changes needed.
```sql
UPDATE candidate_profile
SET resume_text = 'Updated resume text...',
    target_criteria = '{
      "apify_filters": {
        "titleSearch": ["updated search terms"],
        "titleExclusionSearch": ["exclusions"],
        "timeRange": "7d",
        "locationSearch": ["your locations"],
        "aiWorkArrangementFilter": ["Remote OK", "Hybrid"]
      },
      "ai_scoring_criteria": {
        "version": "1.0",
        "scoring_model": "weighted_average",
        "dimensions": [
          { "name": "dimension_1", "description": "Your criteria", "weight": 1.0 }
        ]
      }
    }'
WHERE profile_id = 'default';
```
Both job discovery filters and AI scoring dimensions are dynamically read from this field.

### Supporting Multiple Users
Add multiple profiles to `candidate_profile` table (each with unique email). Main v6.1 automatically processes all profiles in a single run via the Loop Over Profiles node. No workflow modifications needed.

### Customizing AI Criteria
Update the `ai_scoring_criteria` section in `target_criteria` - no workflow changes needed:
```sql
UPDATE candidate_profile
SET target_criteria = '{
  "apify_filters": { ... },
  "ai_scoring_criteria": {
    "version": "1.0",
    "scoring_model": "weighted_average",
    "dimensions": [
      { "name": "custom_dimension", "description": "Your criteria here", "weight": 1.0 }
    ]
  }
}'
WHERE profile_id = 'default';
```
The Loop Jobs workflow automatically uses your custom dimensions in AI evaluation prompts.

### Different Email Formats
Modify HTML template in Send Email v4.1 Build Email node - test with pin data.

### Multiple Recipients
Add multiple profiles to candidate_profile table (Main v6.1 automatically processes all). Each profile receives their own personalized digest.

### Alternative Delivery
Replace SMTP node with Slack, Discord, or database save.

### Weekly Digests
Change cron schedule in Main v6.1 and modify email query to include last 7 days.

---

## Performance Metrics

**Typical Execution:**
- Companies processed: 40
- Jobs discovered: 100-150
- AI evaluations: 100-150
- Execution time: 7-8 minutes
- Email delivery: <1 second

**Success Rates:**
- Job discovery: >95% (API availability)
- AI evaluation: >99% (retry logic)
- Email delivery: >99% (SMTP)

---

## Project Status

**Current State:**
- ✅ Production-ready four-workflow architecture with orchestrator (Main v6.1)
- ✅ Multi-profile support (process multiple users in single run)
- ✅ Profile externalization (clean workflow files, no PII in JSON)
- ✅ Comprehensive documentation (116KB)
- ✅ Fault-tolerant design with error handling
- ✅ **Dual-path status responses (Send Email v4.1) prevent workflow hangs**
- ✅ **Per-run summary email with full category counts saved to `run_reports` table**
- ✅ Cost-efficient ($1.84/day)
- ✅ Complete traceability and monitoring

**Future Enhancements:**
- [ ] Web dashboard for historical analysis
- [ ] Mobile notifications for excellent matches
- [ ] ML-powered company prioritization
- [ ] Integration with ATS APIs directly (bypass Apify)
- [ ] Real-time job alerts (webhook-based)
- [ ] Per-profile scheduling (different run times for each user)

---

## Support & Maintenance

This project is provided **as-is** with no guarantees or warranties:

- ✅ **Issues and pull requests are welcome** - Bug reports, feature requests, and contributions are appreciated
- ⏱️ **Response time is not guaranteed** - This is maintained alongside other commitments; replies may take days or weeks
- 🎯 **Prioritization follows managed service roadmap** - Features aligned with the managed offering may be prioritized over community requests
- 🔧 **Best-effort support only** - No SLAs, uptime guarantees, or dedicated support channels

If you need reliable support, guaranteed response times, or hands-off maintenance, consider the managed service option.

---

## Contributing

This project is open source and contributions are welcome. The architecture is designed to be generalizable beyond the original use case.

**How to adapt RoleFinder for your own use:**
1. Fork the repository
2. Update the `candidate_profile` table with your resume and criteria
3. Modify scoring criteria in Loop Jobs v5.1 to match your targets
4. Populate `companies` table with your target list
5. Configure your own API credentials (Apify, Anthropic, SMTP)

**Pull requests are welcome for:**
- Bug fixes and error handling improvements
- Performance optimizations
- Architectural enhancements
- Documentation improvements
- New features that align with the project's goals (see Non-Goals section)

Please open an issue first for major changes to discuss the approach.

---

## License

This project is licensed under the **GNU General Public License v3.0 (GPL-3.0)**.

In short:
- ✅ You are free to use, modify, and self-host RoleFinder
- ✅ If you redistribute modified versions, you must also provide source code under the same license
- ✅ No warranty is provided - use at your own risk

See the [LICENSE](LICENSE) file for full legal terms.

---

## Acknowledgments

Built with:
- [n8n](https://n8n.io/) - Workflow automation platform
- [Apify](https://apify.com/) - Career site job listing API
- [Anthropic Claude](https://www.anthropic.com/) - AI evaluation engine
- SMTP - Email delivery (standard protocol)

Inspired by the need for intelligent, low-noise job discovery that respects your time and expertise.

---

## About the Creator

RoleFinder was created by **[Ted Beatie](https://tedbeatie.com/)**, a Senior Technical Product Manager with over 20 years of experience in developer tools, infrastructure, and platform engineering.

### Why I Built This

After years of relying on traditional job boards, I needed a system that:

- **Monitored specific companies** where I wanted to work, not every company on the internet
- **Used AI to evaluate fit** based on my actual experience and criteria, not keyword matching
- **Delivered high-signal alerts** instead of overwhelming me with noise
- **Saved hours of manual searching** every single day

RoleFinder is that system. I built it for myself, and now I use it daily to monitor over 50 target companies and receive personalized job digests every morning.

### From Personal Tool to Open Source

What started as a personal automation project evolved into a production-grade system with:
- Clean architecture separating discovery, evaluation, and delivery
- Externalized profile management for multi-user support
- Error handling and fault tolerance
- Comprehensive documentation 

I'm open-sourcing RoleFinder because I believe others face the same frustrations with traditional job search. Whether you're a senior engineer, product manager, designer, or any other role - if you know *where* you want to work but struggle to track opportunities, this system can help.

### Managed Service

For those who want the benefits without the technical setup, I now offer a [fully managed service](#managed-service-option) where I handle all infrastructure, monitoring, and maintenance.

---

## Contact

### For Self-Hosted Users

For questions about architecture, implementation, or technical issues:
- **GitHub Issues**: Open an issue in this repository for bugs or feature requests
- **Support**: Best-effort community support via GitHub

### For Managed Service Inquiries

Interested in having RoleFinder fully managed for you?
- **Email**: [finder@launchbridge.co](mailto:finder@launchbridge.co)
- **Website**: [https://www.launchbridge.co/rolefinder/](https://www.launchbridge.co/rolefinder/)
- **Response Time**: Within 24-48 hours
- **Pricing**: Contact for personalized quote based on your needs

---

**RoleFinder** - Intelligent role monitoring for active job-seekers.

*Last Updated: February 12, 2026*
