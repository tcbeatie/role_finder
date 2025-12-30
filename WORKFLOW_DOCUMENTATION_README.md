# RoleRadar Job Monitoring System - Complete Documentation

## 📋 Table of Contents
1. [Overview](#overview)
2. [System Architecture](#system-architecture)
3. [Workflow 1: Loop Companies v2.1](#workflow-1-loop-companies-v21)
4. [Workflow 2: Loop Jobs v3.1](#workflow-2-loop-jobs-v31)
5. [Workflow 3: Send Email v1.1](#workflow-3-send-email-v11)
6. [Key Concepts](#key-concepts)
7. [Database Schema](#database-schema)
8. [Data Flow & Traceability](#data-flow--traceability)
9. [Profile Management](#profile-management)
10. [Cost Analysis](#cost-analysis)
11. [Troubleshooting](#troubleshooting)
12. [Production Checklist](#production-checklist)
13. [FAQ](#faq)

---

## 🎯 Overview

### Purpose
RoleRadar is an automated job monitoring system that discovers, evaluates, and delivers personalized product management job recommendations via daily email digest.

### Business Problem
As a senior technical product manager, you need to:
- Monitor 40+ target companies for relevant PM positions
- Evaluate job fit across 5 dimensions (seniority, domain, technical depth, location, compensation)
- Receive daily digest of best matches without manual searching
- Track historical job postings for market analysis
- Maintain privacy and security of personal profile data

### Solution Architecture
**Three-workflow pipeline with externalized profile management:**
1. **Loop Companies v2.1**: Discovers jobs from target companies via Apify API, loads profile from database
2. **Loop Jobs v3.1**: Evaluates each job with AI (Claude Sonnet 4.5) using profile from context
3. **Send Email v1.1**: Aggregates results and delivers professional digest via Gmail

### Key Metrics
- **Processing Time**: ~7-8 minutes for 40 companies
- **Cost per Day**: ~$1.84 (120 jobs × $0.003 AI + $1.48 Apify)
- **Deliverables**: Single daily email with all jobs sorted by AI score
- **Success Rate**: >99% with fault-tolerant architecture

### What's New in v2.1/v3.1
**Major Architectural Improvement: Profile Externalization**
- Profile data moved from hardcoded workflow JSON to database table
- Eliminates personal data from workflow files
- Enables multi-user support (different profiles per candidate)
- Simplifies profile updates (no workflow editing required)
- Improves security (profile can be encrypted in database)
- Reduces workflow complexity (fewer nodes, cleaner data flow)

---

## 🗂️ System Architecture

### High-Level Pipeline
```
┌─────────────────────────────────────────────────────────────────────────┐
│                 WORKFLOW 1: LOOP COMPANIES v2.1                         │
│  Purpose: Discover jobs + load profile from database                   │
│  Input: companies table (40 companies) + candidate_profile table       │
│  Output: Raw jobs → jobs table, enriched jobs → Loop Jobs workflow     │
│  NEW: Profile loaded once and passed to all jobs via context           │
└──────────────────────┬──────────────────────────────────────────────────┘
                       │ For each company, calls ↓
┌─────────────────────────────────────────────────────────────────────────┐
│                    WORKFLOW 2: LOOP JOBS v3.1                           │
│  Purpose: AI evaluation using profile from _context_resume_text        │
│  Input: Jobs with _context_ fields (including profile data)            │
│  Output: Formatted job cards → email_queue table                       │
│  NEW: Profile used directly from context, no hardcoded data            │
└──────────────────────┬──────────────────────────────────────────────────┘
                       │ When all companies complete, calls ↓
┌─────────────────────────────────────────────────────────────────────────┐
│                   WORKFLOW 3: SEND EMAIL v1.1                           │
│  Purpose: Aggregate and deliver daily digest                           │
│  Input: workflow_run_id from parent                                    │
│  Output: Professional HTML email via Gmail                             │
└─────────────────────────────────────────────────────────────────────────┘
```

### Workflow Interactions
```
Loop Companies v2.1 (Parent)
├─ Loads: companies table
├─ Loads: candidate_profile table (NEW in v2.1)
├─ Merges: company + profile data (NEW: Merge Data node)
├─ Iterates: One company at a time
├─ For each company:
│  ├─ Calls: Apify API (get jobs)
│  ├─ Saves: Raw jobs → jobs table
│  ├─ Add Context: Enriches jobs with company + profile data
│  │  ├─ _context_workflow_run_id
│  │  ├─ _context_company_id, _context_company_name, _context_domain
│  │  ├─ _context_profile_id (NEW in v2.1)
│  │  ├─ _context_resume_text (NEW in v2.1)
│  │  └─ _context_target_criteria (NEW in v2.1)
│  ├─ Calls: Loop Jobs v3.1 sub-workflow
│  │  ├─ Receives: Jobs with _context_ fields (including profile)
│  │  ├─ Iterates: One job at a time
│  │  ├─ Evaluates: Claude AI uses _context_resume_text directly
│  │  ├─ Formats: HTML job cards
│  │  └─ Saves: Formatted cards → email_queue table
│  └─ Loop back: Next company
├─ When all complete:
│  └─ Calls: Send Email v1.1 sub-workflow
│     ├─ Queries: email_queue WHERE workflow_run_id = X
│     ├─ Aggregates: All jobs, sort by score
│     ├─ Generates: Professional HTML email
│     └─ Sends: Via Gmail API
└─ Complete: User receives daily digest
```

### Key Design Principles

**1. Separation of Concerns**
- **Workflow 1**: Company orchestration, job discovery, profile loading
- **Workflow 2**: Individual job evaluation with AI
- **Workflow 3**: Email aggregation and delivery

**2. Profile Externalization (v2.1/v3.1)**
- Profile stored in database, not hardcoded in workflows
- Loaded once in parent workflow, passed via context
- Eliminates personal data from workflow JSON
- Enables multi-user support and easier updates

**3. Fault Tolerance**
- Companies processed sequentially (one failure doesn't stop others)
- Jobs processed sequentially (AI rate limiting, error isolation)
- Error logging at each level (companies, jobs, email)

**4. Clean Data Flow**
- Context fields (_context_*) pass data across workflow boundaries
- workflow_run_id links all jobs from single execution
- Database tables accumulate data for email generation

**5. Independent Testing**
- Each workflow can be tested separately
- Pin data allows manual testing without dependencies
- Email workflow can regenerate from existing queue data

---

## 📄 Workflow 1: Loop Companies v2.1

### Purpose
Orchestrates company-by-company job discovery, loads candidate profile from database, and coordinates sub-workflow execution.

### Key Architectural Change (v2.1)
**Profile Externalization**: Profile data is now loaded from the `candidate_profile` database table instead of being hardcoded in the workflow. This enables:
- Multi-user support (different profiles for different candidates)
- Easy profile updates (no workflow editing)
- Security (no personal data in workflow JSON)
- Version control friendly (profile changes don't affect workflow)

### Node Structure (15 nodes)
```
Start (Manual Trigger)
  ↓
Load Companies (40 companies from database)
  ↓
Merge Data (NEW v2.1: Combine companies with profile)
  ↓
Build Apify Request (Extract criteria from profile.target_criteria)
  ↓
Loop Over Companies ─────┐
  ↓                      │
  Run Apify API          │
    ↓ Success  ↓ Error   │
    IF        Log Error  │
    ↓True ↓False  ↓      │
    Add    Log   Insert  │
   Context  No   Bad Row │
    ↓↓      Res    ↓     │
    │└─Save Apify──┘     │
    │   Data ↓           │
    │   Insert Raw Jobs  │
    │                    │
    └─Call Loop Jobs v3.1 ─┤
      (passes profile)   │
      ↓                  │
      Loop back ─────────┘
      
When all companies complete:
  ↓
Workflow Summary
  ↓
Call 'Send Email v1.1'
```

### Critical Nodes Explained

**NODE 1: Start**
- **Type**: Manual Trigger
- **Purpose**: Entry point for workflow
- **Production**: Can be triggered by schedule or webhook
- **Testing**: Manual execution with pin data

**NODE 2: Load Companies**
- **Type**: Data Table (Get)
- **Query**: All active companies from `companies` table
- **Returns**: ~40 company records with domain, name, id
- **Note**: Runs in parallel with profile loading

**NODE 3: Merge Data (NEW in v2.1)**
- **Type**: Merge node
- **Purpose**: Combine companies with profile data
- **Inputs**:
  - Input 1: Companies from Load Companies
  - Input 2: Profile from Start trigger (or database load)
- **Output**: Each company record now includes profile fields
- **Architectural benefit**: Single profile load propagates to all companies

**NODE 4: Build Apify Request**
- **Type**: Code node
- **Purpose**: Transform company + profile into Apify API format
- **NEW**: Extracts `titleSearch` and `titleExclusionSearch` from `profile.target_criteria` JSON
- **Output**: Properly formatted API request per company
- **Dynamic**: Job search criteria now configurable per profile

**NODE 5: Loop Over Companies**
- **Type**: Split In Batches (batch size: 1)
- **Purpose**: Process companies one at a time
- **Why**: Fault tolerance - if company #25 fails, #26-40 still process
- **Two outputs**:
  - Loop branch (bottom): Fires once per company
  - Done branch (top): Fires once when all complete

**NODE 6: Run Apify**
- **API**: Career Site Job Listing API
- **Returns**: 0-10 jobs per company
- **Settings**:
  - `alwaysOutputData: true` - Output even if 0 jobs
  - `onError: continueErrorOutput` - Don't crash on errors
- **Two outputs**:
  - Success: API succeeded (0+ jobs)
  - Error: API failed (timeout, auth, network)

**NODE 8: Add Context (ENHANCED in v2.1)**
- **Purpose**: Enrich jobs with parent workflow context + profile
- **Adds company context**:
  - `_context_workflow_run_id`: Parent execution ID
  - `_context_company_id`: Company identifier
  - `_context_company_name`: Company display name
  - `_context_domain`: Company domain
- **Adds profile context (NEW v2.1)**:
  - `_context_profile_id`: Profile identifier
  - `_context_resume_text`: Complete resume for AI evaluation
  - `_context_target_criteria`: Job search criteria JSON
- **Why**: Sub-workflow needs both company AND profile context
- **Note**: References Merge Data node for original merged data with profile

**NODE 11: Call 'Loop Jobs v3.1'**
- **Type**: Execute Workflow
- **Calls**: Loop Jobs v3.1 sub-workflow
- **Passes**: All jobs with _context_ fields (including profile data)
- **NEW**: Profile data passed via context, not hardcoded
- **Waits**: For sub-workflow to complete all jobs
- **Returns**: Control to company loop when done

**NODE 9: Save Apify data → Insert row**
- **Parallel path**: Runs alongside sub-workflow call
- **Purpose**: Immediate raw job backup
- **Fast**: ~100ms per job
- **Does NOT loop back**: Ends after insert

### Key Behaviors

**Profile Flow (NEW in v2.1):**
```
Start → Profile loaded from trigger or database
  ↓
Load Companies → Companies from database (parallel)
  ↓
Merge Data → Combines profile with each company
  ↓
Build Request → Extracts criteria from profile.target_criteria
  ↓
Loop Over Companies → Each iteration has company+profile
  ↓
Add Context → Passes profile fields to sub-workflow
  ↓
Loop Jobs → Uses _context_resume_text for AI evaluation
```

**Parallel Processing After Add Context:**
After jobs are enriched with context, TWO paths execute:

**Path A (Fast - Database Backup):**
```
Add Context → Save Apify data → Insert row → [ENDS]
```
- Immediate raw job storage
- Completes in milliseconds
- Backup before AI evaluation

**Path B (Slow - AI Evaluation):**
```
Add Context → Call 'Loop Jobs v3.1' → [Loops back when all jobs complete]
```
- Calls sub-workflow for AI evaluation
- Takes 2-5 seconds per job
- Loops back to process next company

**Both paths must complete before company loop advances.**

### Database Tables Used
1. **companies** (input): Target companies to monitor
2. **candidate_profile** (input, NEW v2.1): Candidate resume and criteria
3. **jobs_test_1** (output): Raw job data with searchable fields
4. **errors** (output): API failures and no-results tracking

### Error Handling
- **API Error**: Logged, workflow continues to next company
- **No Results**: Logged separately, workflow continues
- **All errors**: Feed back to Loop Over Companies for next iteration

### Benefits of v2.1 Profile Architecture

**Before (v5):**
- Profile hardcoded in Loop Jobs workflow
- 60+ lines of resume text in workflow JSON
- Multi-user support impossible
- Profile updates require editing multiple workflows
- Personal data exposed in version control

**After (v2.1):**
- Profile loaded from database once
- No personal data in workflow JSON
- Multi-user ready (different profiles per execution)
- Profile updates only need database change
- Version control friendly
- More secure (can encrypt profile in database)

---

## 🤖 Workflow 2: Loop Jobs v3.1

### Purpose
Evaluates individual jobs with AI using profile from context, formats HTML cards, saves to email queue.

### Key Architectural Change (v3.1)
**Profile Context Usage**: The "Add Resume Context" node has been **eliminated**. Profile data now arrives via `_context_resume_text` from the parent workflow and is used directly in the AI evaluation node. This simplifies the architecture and eliminates hardcoded profile data.

### Node Structure (8 nodes - simplified from 9)
```
Start (receives jobs with _context_ fields including profile)
  ↓
Loop Over Jobs ─────────┐
  ↓                     │
  │ ← PARALLEL SPLIT → │
  │                     │
  Message a model       │
  (uses _context_      │
   resume_text)        │
  ↓                     │
  Parse AI Response    Merge (Input 1: Raw job data)
  ↓                     │
  Merge (Input 0: AI eval) ←┘
  ↓
  Format Job Card
  ↓
  Save to Email Queue
  ↓
  Loop back ────────────┘

When all jobs complete:
  ↓
Workflow Complete
```

### Critical Nodes Explained

**NODE 1: Start**
- **Type**: Manual Trigger  
- **Purpose**: Sub-workflow entry point
- **Input**: Jobs with _context_ fields from parent workflow
- **NEW v3.1**: Receives `_context_resume_text` (no separate profile loading needed)

**NODE 2: Loop Over Jobs**
- **Type**: Split In Batches (batch size: 1)
- **Purpose**: Process jobs one at a time
- **Why**: AI rate limiting, cost tracking, error isolation
- **NEW v3.1**: Job data already includes profile via `_context_resume_text`
- **Two outputs**:
  - Done branch (top): All jobs complete → Workflow Complete
  - Loop branch (bottom): SPLITS to two parallel paths:
    - Path A: Message a model (AI evaluation)
    - Path B: Direct to Merge (raw data preservation)

**NODE 3: Message a model (UPDATED v3.1)**
- **Type**: Anthropic Claude API
- **Model**: claude-sonnet-4-5-20250929
- **NEW**: Uses `{{ $json._context_resume_text }}` directly in prompt
- **NO LONGER NEEDED**: Separate "Add Resume Context" node eliminated
- **Input**: Job data with profile already in `_context_resume_text`
- **Prompt**:
  - Candidate Profile: `{{ $json._context_resume_text }}`
  - Job Details: All Apify fields
  - Evaluation structure: 5 scoring dimensions + reasoning
- **Output**: AI evaluation JSON with scores and recommendations
- **Cost**: ~$0.003 per job

**NODE 4: Parse AI Response**
- **Type**: Code node
- **Purpose**: Extract and structure AI evaluation
- **Input**: Claude API response with JSON in `content[0].text`
- **Processing**:
  - Clean markdown fences (```json)
  - Parse JSON string
  - Flatten structure (extract scores, arrays)
  - Preserve full evaluation in `ai_evaluation` field
- **Output**: Structured evaluation ready for merging

**NODE 5: Merge**
- **Type**: Merge node (Combine by Fields)
- **Purpose**: Combine AI evaluation with raw job data
- **Merge key**: `job_id` (AI eval) == `id` (raw job)
- **NEW v3.1**: Input 1 comes DIRECTLY from Loop Over Jobs
  - Old: From "Add Resume Context" node
  - New: Direct from loop (no intermediate transformation)
- **Inputs**:
  - Input 0 (AI path): Evaluation scores and reasoning
  - Input 1 (Raw path): Complete job details + _context_ fields
- **Output**: Single object with both AI evaluation AND complete job data

**NODE 6: Format Job Card**
- **Type**: Code node
- **Purpose**: Generate HTML email card for job
- **Input**: Merged job data (AI + raw + context)
- **Processing**:
  - Format salary, location with helper functions
  - Generate color-coded badges
  - Build HTML card with inline styles
  - Include AI assessment sections
- **Output**: Complete HTML card + metadata for database

**NODE 7: Save to Email Queue**
- **Type**: Data Table (Insert)
- **Table**: email_queue
- **Inserts**:
  - workflow_run_id (for grouping)
  - recommendation, overall_score (for sorting)
  - html_card (complete formatted card)
  - job_id, company_name, job_title (for reference)
- **Purpose**: Accumulate all jobs for email workflow
- **Loops back**: To Loop Over Jobs for next job

**NODE 8: Workflow Complete**
- **Type**: Code node
- **Purpose**: Signal completion to parent workflow
- **Output**: Status and job count
- **Returns**: Control to parent Loop Companies workflow

### Parallel Processing Pattern

**Why Two Paths After Loop Over Jobs?**

Loop Over Jobs outputs to TWO destinations simultaneously:

**Path A (AI Evaluation - Slow):**
```
Loop Over Jobs → Message a model → Parse AI Response → Merge (Input 0)
```
- Takes 2-5 seconds per job
- Costs $0.003 per job
- Provides intelligence and scoring

**Path B (Raw Data Preservation - Fast):**
```
Loop Over Jobs → Merge (Input 1)
```
- Instant (no processing)
- Free (no API calls)
- Preserves complete job data

**Merge Node Receives Both:**
- Input 0: AI evaluation (from Path A)
- Input 1: Complete job data (from Path B)
- Result: Job with BOTH AI scores AND all original fields

**Why This Pattern?**
- AI node only returns evaluation, not original fields
- Merge combines AI intelligence with complete job details
- Fault tolerant: if AI fails, raw data preserved
- Clear separation: AI processing vs. data preservation

### Profile Usage (v3.1)

**Before (v5):**
```
Loop Over Jobs
  ↓
Add Resume Context (60+ lines hardcoded profile)
  ↓ adds candidate_profile field
Parallel split to AI + Raw paths
```

**After (v3.1):**
```
Loop Over Jobs (already has _context_resume_text from parent)
  ↓
Direct parallel split to AI + Raw paths
  - AI uses _context_resume_text directly
  - No intermediate transformation needed
```

**Benefits:**
- One less node to maintain
- No hardcoded personal data
- Simpler data flow
- Profile updates don't require workflow changes
- Multi-user ready

### Database Tables Used
1. **email_queue** (output): Formatted job cards for email

### Cost Analysis (per job)
- Apify API: Included in company-level cost
- Claude AI: $0.003 per job
- Database: Negligible
- **Total**: ~$0.003 per job

---

## 📧 Workflow 3: Send Email v1.1

### Purpose
Aggregates all evaluated jobs from the workflow run and delivers professional HTML email digest via Gmail.

### Node Structure (5 nodes)
```
Start (receives workflow_run_id from parent)
  ↓
Load Email Queue (query by workflow_run_id)
  ↓
Aggregate and Sort Jobs (many → one item)
  ↓
Build Email (generate HTML template)
  ↓
Send Jobs Email (Gmail delivery)
```

### Critical Nodes Explained

**NODE 1: Start**
- **Type**: Manual Trigger
- **Purpose**: Sub-workflow entry point
- **Input**: `workflow_run_id` from parent workflow
- **Critical**: This ID links all jobs from current run

**NODE 2: Load Email Queue**
- **Type**: Data Table (Get with Filters)
- **Query**: `WHERE workflow_run_id = {{ $json.workflow_run_id }}`
- **Returns**: ALL jobs across ALL companies from this run
- **Example**: 20 job items (8 from Anthropic + 5 from OpenAI + 7 from Google)
- **Each item contains**:
  - workflow_run_id
  - recommendation (EXCELLENT_MATCH, GOOD_MATCH, CONSIDER, POOR_FIT, REJECT)
  - overall_score (1-10)
  - html_card (complete formatted HTML)
  - job_id, company_name, job_title

**NODE 3: Aggregate and Sort Jobs**
- **Type**: Code node
- **Purpose**: Transform many items → one aggregated summary
- **Processing**:
  - Categorize by recommendation
  - Sort by overall_score DESC (best first)
  - Concatenate HTML cards with spacers
  - Calculate summary statistics
- **Input**: 20 separate job items
- **Output**: Single item with:
  - total_jobs: 20
  - excellent_count: 3
  - strong_count: 8
  - consider_count: 6
  - poor_count: 3
  - job_cards_html: Combined HTML string
  - workflow_run_id: For footer

**NODE 4: Build Email**
- **Type**: Code node
- **Purpose**: Generate professional HTML email
- **Creates**:
  - **Dynamic subject line**:
    - "🎯 3 Excellent Matches, 8 Strong • 20 Total Jobs"
    - Adapts based on job quality distribution
  - **HTML email body**:
    - Header with date and branding
    - Summary statistics (color-coded grid)
    - Highlight banner if excellent matches exist
    - All job cards (sorted by score)
    - Footer with workflow_run_id
- **Output**: Complete email ready to send
  - subject: Dynamic subject line
  - html_body: Full HTML email
  - Statistics for logging

**NODE 5: Send Jobs Email**
- **Type**: Gmail node
- **API**: Gmail API with OAuth2
- **Configuration**:
  - sendTo: tcbeatie@gmail.com
  - subject: `{{ $json.subject }}`
  - message: `{{ $json.html_body }}`
- **Delivery**: < 1 second
- **Result**: Email in inbox, appears in Sent folder
- **This is the FINAL node in the entire pipeline**

### Email Design Features

**Professional Layout:**
- Responsive design (mobile + desktop)
- Inline styles (email client compatible)
- 700px max width for readability
- Color-coded statistics (green/blue/amber/gray)

**Dynamic Content:**
- Subject line adapts to job quality
- Highlight banner for excellent matches
- Conditional formatting based on job counts
- Summary statistics with visual feedback

**Email Client Compatibility:**
- Tested with Gmail, Outlook, Apple Mail
- Graceful degradation of advanced features
- No external CSS or scripts
- Table-free layout (modern approach)

### Data Transformation Pattern

This workflow demonstrates the **many → one → delivered** pattern:

1. **MANY items** (Load Email Queue): 20 individual jobs
2. **ONE item** (Aggregate and Sort): Single summary object
3. **DELIVERED** (Send Email): User receives digest

**Why consolidate?**
- Simplifies email generation
- Pre-calculated statistics
- Single template to fill
- Cleaner node logic

---

## 🔑 Key Concepts

### workflow_run_id - The Master Key

The `workflow_run_id` is **THE KEY** that ties the entire three-workflow system together.

**Created by**: Loop Companies workflow (`$execution.id`)
**Value example**: 67890

**Propagation path**:
```
Loop Companies ($execution.id = 67890)
  ↓ passes via _context_workflow_run_id
Loop Jobs (saves to email_queue with workflow_run_id = 67890)
  ↓ all jobs from all companies tagged with 67890
Send Email (queries WHERE workflow_run_id = 67890)
  ↓ retrieves all jobs from that run
Single email with all 20+ jobs
```

**Why critical?**
- Groups all jobs from single daily run
- Enables email to query all jobs at once
- Allows historical email regeneration
- Facilitates debugging and tracing
- Prevents mixing jobs from different days

**Without workflow_run_id:**
- No way to know which jobs belong together
- Email would mix jobs from different days
- Impossible to regenerate specific day's email
- No audit trail for workflow executions

### Context Fields Pattern

**_context_* fields** pass data across workflow boundaries.

**Company context** (added by Loop Companies):
- `_context_workflow_run_id`: Links to parent execution
- `_context_company_id`: Company identifier
- `_context_company_name`: Company display name
- `_context_domain`: Company domain

**Profile context** (NEW in v2.1, added by Loop Companies):
- `_context_profile_id`: Profile identifier
- `_context_resume_text`: Complete resume for AI
- `_context_target_criteria`: Job search criteria JSON

**Why prefix with _context_?**
- Clear distinction from job data fields
- Prevents name collisions
- Self-documenting (clearly parent workflow data)
- Easy to filter out when saving raw jobs

### Sub-Workflow Pattern

**Execute Workflow** nodes enable modular architecture.

**Benefits:**
1. **Separation of concerns**: Each workflow has single responsibility
2. **Independent testing**: Test workflows in isolation
3. **Reusability**: Loop Jobs can be called for different use cases
4. **Scalability**: Can parallelize if needed
5. **Maintainability**: Changes isolated to specific workflow

**Parent-Child Relationship:**
```
Loop Companies (parent)
  ├─ Calls Loop Jobs (child) multiple times
  │  └─ Returns when all jobs processed
  └─ Calls Send Email (child) once
     └─ Returns when email sent
```

**Data passing:**
- Parent passes data via workflow input
- Child processes and returns result
- Parent continues with returned data

### Parallel Paths Pattern

**Parallel processing** enables backup + intelligence.

**Used in**:
1. **Loop Companies**: Raw job storage + AI evaluation
2. **Loop Jobs**: Raw data preservation + AI scoring

**Benefits:**
- **Fast path**: Immediate backup (milliseconds)
- **Slow path**: Intelligence/processing (seconds)
- **Fault tolerant**: If slow path fails, fast path preserved
- **Clean separation**: Storage vs. processing logic

**n8n behavior:**
- Both paths execute simultaneously
- Downstream nodes wait for both to complete
- Merge nodes require both inputs before firing

### Database as Accumulator

**Tables accumulate data** across iterations for batch processing.

**Pattern:**
```
Loop (many iterations)
  ↓ each saves one row
Database table accumulates all rows
  ↓ query all rows
Process batch (email, analysis, etc.)
```

**Used for:**
- email_queue: Accumulates all job cards
- jobs: Accumulates raw job data
- errors: Accumulates error logs

**Benefits:**
- Decouple processing from accumulation
- Enable batch operations
- Historical data preservation
- Easy querying and aggregation

---

## 💾 Database Schema

### Table: companies
**Purpose**: Target companies to monitor for job postings

| Column | Type | Description |
|--------|------|-------------|
| id | Integer | Auto-increment primary key |
| company_id | String | Unique company identifier (e.g., "anthropic") |
| company_name | String | Display name (e.g., "Anthropic") |
| domain | String | Company domain (e.g., "anthropic.com") |
| status | String | "active" or "inactive" |
| notes | Text | Optional notes about company |
| createdAt | DateTime | When record was created |
| updatedAt | DateTime | When record was last modified |

**Indexes**: 
- PRIMARY KEY (id)
- UNIQUE (company_id)

**Usage**: Load Companies node queries WHERE status = 'active'

---

### Table: candidate_profile (NEW in v2.1)
**Purpose**: Store candidate resume and job search criteria

| Column | Type | Description |
|--------|------|-------------|
| id | Integer | Auto-increment primary key |
| profile_id | String | Unique profile identifier (e.g., "ted_beatie_pm") |
| profile_name | String | Display name (e.g., "Ted Beatie - Technical PM") |
| resume_text | Text | Complete resume/experience for AI evaluation |
| target_criteria | JSON/Text | Job search criteria (titles, location, salary, etc.) |
| notes | Text | Optional profile notes |
| createdAt | DateTime | When profile was created |
| updatedAt | DateTime | When profile was last modified |

**Indexes**:
- PRIMARY KEY (id)
- UNIQUE (profile_id)

**Usage**: 
- Loop Companies loads profile at workflow start
- Profile passed via _context_* fields to sub-workflows
- Loop Jobs uses resume_text for AI evaluation

**Example target_criteria JSON**:
```json
{
  "titleSearch": "\"Product Manager\" OR \"Technical Program Manager\" OR \"Technical Product Manager\"",
  "titleExclusionSearch": "\"Associate Product Manager\" OR \"Junior\" OR \"Entry\" OR \"Intern\"",
  "minSalary": 190000,
  "locations": ["Remote", "Hybrid NYC Metro", "Hybrid Bay Area"],
  "requiredSkills": ["Infrastructure", "Platform", "API", "Cloud", "Developer Tools"]
}
```

**Security note**: 
- Consider encrypting resume_text in production
- Access controls on this table
- Regular backups with encryption
- No exposure in workflow JSON (security improvement over v5)

---

### Table: jobs_test_1
**Purpose**: Store raw job data from Apify API

| Column | Type | Description |
|--------|------|-------------|
| id | Integer | Auto-increment primary key |
| job_id | String | Unique job identifier from Apify |
| workflow_run_id | Integer | Links to workflow execution |
| company_id | String | Company that posted job |
| company_name | String | Company display name |
| domain | String | Company domain |
| title | String | Job title |
| location | String | Job location |
| salary_min | Integer | Minimum salary |
| salary_max | Integer | Maximum salary |
| description | Text | Full job description |
| url | String | Link to job posting |
| date_posted | DateTime | When job was posted |
| job_data_full | JSON | Complete Apify response (all fields) |
| createdAt | DateTime | When record was created |

**Indexes**:
- PRIMARY KEY (id)
- INDEX (workflow_run_id) - for querying by run
- INDEX (company_id) - for company-specific queries
- UNIQUE (job_id) - prevent duplicates

**Usage**: Save Apify data → Insert row saves each job immediately

---

### Table: email_queue
**Purpose**: Store formatted job cards for email aggregation

| Column | Type | Description |
|--------|------|-------------|
| id | Integer | Auto-increment primary key |
| workflow_run_id | Integer | Links to workflow execution |
| job_id | String | Links to jobs_test_1 table |
| company_name | String | Company that posted job |
| job_title | String | Job title for reference |
| recommendation | String | EXCELLENT_MATCH, GOOD_MATCH, CONSIDER, POOR_FIT, REJECT |
| overall_score | Integer | 1-10 AI scoring |
| html_card | Text | Complete formatted HTML card |
| processed_at | DateTime | When AI evaluation completed |
| createdAt | DateTime | When record was created |

**Indexes**:
- PRIMARY KEY (id)
- INDEX (workflow_run_id) - CRITICAL for email query performance
- INDEX (recommendation, overall_score) - for filtering/sorting
- INDEX (processed_at) - for cleanup operations

**Usage**:
- Loop Jobs saves one row per evaluated job
- Send Email queries WHERE workflow_run_id = X
- All jobs from workflow run retrieved in single query

**Cleanup**:
```sql
-- Delete old entries (e.g., >30 days)
DELETE FROM email_queue 
WHERE processed_at < NOW() - INTERVAL '30 days';
```

---

### Table: errors
**Purpose**: Log workflow errors and monitoring data

| Column | Type | Description |
|--------|------|-------------|
| id | Integer | Auto-increment primary key |
| workflow_run_id | Integer | Links to workflow execution |
| company_id | String | Company that had error |
| company_name | String | Company display name |
| domain | String | Company domain |
| error_type | String | "api_error", "no_results", "validation_error" |
| error_message | Text | Error details |
| timestamp | DateTime | When error occurred |
| createdAt | DateTime | When record was created |

**Usage**:
- Log Error node saves API failures
- Log no results node saves zero-job scenarios
- Enables monitoring and alerting

---

## 🔄 Data Flow & Traceability

### Complete Data Flow

```
1. WORKFLOW START
   Loop Companies v2.1 starts
   Execution ID: 67890 (becomes workflow_run_id)
   
2. PROFILE LOADING (NEW in v2.1)
   Load from candidate_profile table
   profile_id: "ted_beatie_pm"
   resume_text: "TED BEATIE - PRODUCT LEADER..."
   target_criteria: { titleSearch: "...", ... }
   
3. COMPANY LOADING
   Load from companies table
   40 active companies
   
4. MERGE PROFILE + COMPANIES
   Merge Data node combines
   Each company now has profile fields attached
   
5. BUILD API REQUESTS
   Extract criteria from profile.target_criteria
   titleSearch dynamically set from profile
   
6. COMPANY LOOP
   For each company (e.g., Anthropic):
   
   7. APIFY API CALL
      domain: "anthropic.com"
      titleSearch: from profile
      Returns: 8 jobs
      
   8. RAW JOB STORAGE (Fast Path)
      jobs_test_1 table
      8 rows inserted
      Fields: job_id, title, company, salary, etc.
      workflow_run_id: 67890
      
   9. ADD CONTEXT
      Attach to each job:
      - _context_workflow_run_id: 67890
      - _context_company_id: "anthropic"
      - _context_company_name: "Anthropic"
      - _context_domain: "anthropic.com"
      - _context_profile_id: "ted_beatie_pm" (NEW)
      - _context_resume_text: "TED BEATIE..." (NEW)
      - _context_target_criteria: {...} (NEW)
      
   10. CALL LOOP JOBS v3.1 (Slow Path)
       Receives 8 jobs with all context fields
       
       11. JOB LOOP
           For each job (e.g., "Senior PM"):
           
           12. PARALLEL SPLIT
               Path A: AI evaluation
               Path B: Raw data to merge
               
           13. AI EVALUATION (Path A)
               Message a model node
               Uses _context_resume_text directly (NEW)
               Prompt: Profile + Job → Evaluation
               Returns: Scores, recommendation, reasoning
               Cost: $0.003
               
           14. PARSE RESPONSE
               Extract scores, flatten structure
               job_id, overall_score, recommendation, etc.
               
           15. MERGE (Both Paths)
               Input 0: AI evaluation (from Path A)
               Input 1: Raw job + context (from Path B)
               Combined: AI scores + Complete job details
               
           16. FORMAT HTML CARD
               Generate professional card
               Include: Title, company, salary, location
               Include: AI scores, strengths, weaknesses
               Include: Color-coded badge
               
           17. SAVE TO EMAIL QUEUE
               email_queue table
               workflow_run_id: 67890
               html_card: complete HTML
               recommendation: EXCELLENT_MATCH
               overall_score: 9
               
           18. LOOP BACK
               Process next job
               
       19. ALL JOBS COMPLETE
           Return to Loop Companies
           
   20. LOOP BACK
       Process next company
       
21. ALL COMPANIES COMPLETE
    40 companies × ~3 jobs = 120 jobs in email_queue
    All tagged with workflow_run_id: 67890
    
22. WORKFLOW SUMMARY
    Consolidate stats
    Prepare to call Send Email
    
23. CALL SEND EMAIL v1.1
    Pass workflow_run_id: 67890
    
    24. LOAD EMAIL QUEUE
        Query: WHERE workflow_run_id = 67890
        Returns: 120 job items
        
    25. AGGREGATE AND SORT
        Categorize by recommendation:
        - excellent_count: 12
        - strong_count: 35
        - consider_count: 48
        - poor_count: 25
        Sort by overall_score DESC
        Concatenate HTML cards with spacers
        
    26. BUILD EMAIL
        Subject: "🎯 12 Excellent Matches, 35 Strong • 120 Total Jobs"
        HTML body: Header + Stats + Cards + Footer
        
    27. SEND VIA GMAIL
        API call to Gmail
        Email delivered to inbox
        
28. WORKFLOW COMPLETE
    User receives email
    System ready for tomorrow
```

### Tracing a Specific Job

**Example**: Trace "Senior Technical PM" at Anthropic

1. **Company Loop** (Loop Companies):
   - Company: Anthropic (company_id: "anthropic")
   - Apify returns 8 jobs
   - Job #3: "Senior Technical Product Manager"
   - job_id: "fantastic-jobs-12345"

2. **Raw Storage** (Fast Path):
   - Saved to jobs_test_1
   - Record #8542
   - workflow_run_id: 67890
   - Searchable: title, company, salary

3. **Context Addition**:
   - _context_workflow_run_id: 67890
   - _context_company_id: "anthropic"
   - _context_resume_text: Full profile (NEW)
   - Job + context → Loop Jobs

4. **Job Loop** (Loop Jobs):
   - Job iteration #3
   - Parallel paths split

5. **AI Evaluation** (Path A):
   - Claude evaluates using _context_resume_text
   - overall_score: 9
   - recommendation: EXCELLENT_MATCH
   - Reasoning: Strong infrastructure focus, senior level, remote

6. **Merge**:
   - AI evaluation (Input 0) + Raw job (Input 1)
   - Combined object with scores + details

7. **HTML Generation**:
   - Format Job Card creates HTML
   - Green badge (excellent match)
   - Scores displayed
   - Strengths/weaknesses sections

8. **Email Queue**:
   - Saved to email_queue
   - Record #1205
   - workflow_run_id: 67890
   - html_card: Complete formatted HTML

9. **Email Aggregation** (Send Email):
   - Queried from email_queue
   - Sorted by score (9 → high priority)
   - Appears near top of email

10. **Email Delivery**:
    - Card #3 in digest
    - User sees and applies
    - Success!

### Debugging With IDs

**To trace any job**:

```sql
-- 1. Find job in raw storage
SELECT * FROM jobs_test_1 
WHERE workflow_run_id = 67890 
AND job_id = 'fantastic-jobs-12345';

-- 2. Find in email queue
SELECT * FROM email_queue 
WHERE workflow_run_id = 67890 
AND job_id = 'fantastic-jobs-12345';

-- 3. Find all jobs from workflow run
SELECT COUNT(*) FROM email_queue 
WHERE workflow_run_id = 67890;

-- 4. Find all excellent matches from run
SELECT * FROM email_queue 
WHERE workflow_run_id = 67890 
AND recommendation = 'EXCELLENT_MATCH'
ORDER BY overall_score DESC;
```

---

## 👤 Profile Management

### Overview (NEW in v2.1)

Profile management has been completely redesigned in v2.1/v3.1 to externalize candidate data from workflow code into a database table.

### Architecture Improvements

**Before (v5 and earlier):**
```
- Profile hardcoded in Loop Jobs workflow
- 60+ lines of resume text in workflow JSON
- Profile updates require workflow editing
- Personal data exposed in version control
- Multi-user support impossible
- Security concerns (plaintext in workflows)
```

**After (v2.1/v3.1):**
```
- Profile stored in candidate_profile database table
- Loaded once in Loop Companies workflow
- Passed via _context_* fields to sub-workflows
- Profile updates via database only (no workflow changes)
- Multi-user ready (multiple profiles supported)
- More secure (can encrypt profile data)
- Version control friendly (no personal data in workflow JSON)
```

### Profile Data Structure

**candidate_profile table:**
```json
{
  "profile_id": "ted_beatie_pm",
  "profile_name": "Ted Beatie - Technical PM",
  "resume_text": "TED BEATIE - PRODUCT LEADER\n\nSUMMARY:\n- 15+ years...",
  "target_criteria": {
    "titleSearch": "\"Product Manager\" OR \"Technical Program Manager\"",
    "titleExclusionSearch": "\"Associate\" OR \"Junior\" OR \"Intern\"",
    "minSalary": 190000,
    "locations": ["Remote", "Hybrid NYC Metro", "Hybrid Bay Area"],
    "requiredSkills": ["Infrastructure", "Platform", "API"]
  },
  "notes": "Senior technical PM with infrastructure focus"
}
```

### Profile Flow

```
Loop Companies Start
  ↓
Load profile from candidate_profile table
  ↓
Merge Data: Combine profile with companies
  ↓
Build Request: Extract criteria from profile.target_criteria
  ↓
Loop Over Companies: Each company has profile attached
  ↓
Add Context: Pass profile to jobs via:
  - _context_profile_id
  - _context_resume_text
  - _context_target_criteria
  ↓
Loop Jobs: AI uses _context_resume_text directly
  ↓
No hardcoded profile data anywhere in workflow
```

### Managing Profiles

**Create New Profile:**
```sql
INSERT INTO candidate_profile (
  profile_id,
  profile_name,
  resume_text,
  target_criteria,
  notes
) VALUES (
  'john_doe_pm',
  'John Doe - Senior PM',
  'JOHN DOE - PRODUCT MANAGER...',
  '{"titleSearch": "Product Manager", "minSalary": 150000}',
  'Consumer product focus'
);
```

**Update Profile:**
```sql
-- Update resume
UPDATE candidate_profile
SET resume_text = 'Updated resume text...',
    updatedAt = NOW()
WHERE profile_id = 'ted_beatie_pm';

-- Update target criteria
UPDATE candidate_profile
SET target_criteria = '{"titleSearch": "Director OR VP", "minSalary": 250000}',
    updatedAt = NOW()
WHERE profile_id = 'ted_beatie_pm';
```

**Switch Active Profile:**
In Loop Companies workflow, modify the Start node or database query to load different profile_id.

**Delete Profile:**
```sql
DELETE FROM candidate_profile 
WHERE profile_id = 'old_profile';
```

### Multi-User Support

With profile externalization, the system now supports multiple users:

**Option 1: Multiple Profiles, Single Workflow**
- Store multiple profiles in candidate_profile table
- Pass desired profile_id to workflow execution
- Workflow loads specified profile dynamically

**Option 2: Separate Workflows per User**
- Clone workflows for each user
- Each workflow hardcoded to specific profile_id
- Independent execution schedules

**Option 3: Dynamic Profile Selection**
- Add profile_id parameter to workflow trigger
- Schedule different executions for different profiles
- Shared workflow infrastructure

### Security Best Practices

1. **Encryption at Rest**
   ```sql
   -- Encrypt sensitive columns (PostgreSQL example)
   ALTER TABLE candidate_profile 
   ADD COLUMN resume_text_encrypted BYTEA;
   
   -- Use pgcrypto for encryption
   UPDATE candidate_profile
   SET resume_text_encrypted = pgp_sym_encrypt(resume_text, 'encryption-key');
   ```

2. **Access Controls**
   - Limit database access to workflow user
   - Separate read/write permissions
   - Audit log for profile access

3. **Data Retention**
   - Clear policy for profile data retention
   - Regular cleanup of old profiles
   - Secure deletion (not just logical delete)

4. **API Security**
   - Profile data encrypted in transit to Anthropic API
   - No profile data stored by Anthropic (per privacy policy)
   - Review Anthropic's data handling policies

5. **Version Control**
   - Never commit profile data to git
   - Workflow JSON contains no personal information
   - Safe to share workflows publicly

### Benefits Summary

**Security:**
- No personal data in workflow JSON
- Can encrypt profile in database
- Centralized access control
- Audit trail capability

**Flexibility:**
- Update profile without workflow changes
- A/B test different profiles
- Support multiple users
- Easy to backup/restore

**Maintainability:**
- Single source of truth for profile data
- Database-managed updates
- No workflow editing for profile changes
- Version control friendly

**Scalability:**
- Support unlimited profiles
- Profile-specific workflows possible
- Easy to add new users
- Shared infrastructure

---

## 💰 Cost Analysis

### Daily Costs

**Apify API** (~$1.48/day):
- 40 companies × ~3 jobs average = 120 jobs
- Cost: 120 jobs × $0.012 per job = $1.44
- Platform fee: ~$0.04
- **Total Apify**: ~$1.48/day

**Anthropic Claude API** (~$0.36/day):
- 120 jobs × 1 evaluation each = 120 API calls
- Input: ~2,500 tokens per job (profile + job description)
- Output: ~500 tokens per job (structured evaluation)
- Cost per job: ~$0.003
- **Total AI**: 120 × $0.003 = $0.36/day

**Gmail API**: Free
- Within generous free tier limits
- 1 email/day well under quotas

**n8n Hosting**: Variable
- Self-hosted: Infrastructure costs only
- Cloud: Depends on plan
- Database: Minimal storage requirements

**Total Daily Cost**: ~$1.84/day
**Monthly Cost**: ~$55/month
**Annual Cost**: ~$672/year

### Cost Optimization Strategies

**Reduce Apify Costs:**
1. Monitor fewer companies (40 → 20) = 50% savings
2. Increase timeRange (7d → 14d) = run less frequently
3. Self-host career site scraping (eliminates Apify)
4. Batch company requests (if Apify supports)

**Reduce AI Costs:**
1. Pre-filter jobs before AI evaluation:
   ```javascript
   // Filter out obviously bad matches
   const eligible = jobs.filter(job => {
     const title = job.title.toLowerCase();
     return !title.includes('associate') && 
            !title.includes('junior') &&
            job.salary_min >= 150000;
   });
   ```
   Savings: ~30% if filtering removes 30% of jobs

2. Use cheaper AI model for pre-screening:
   - Claude Haiku for initial filter (cheaper)
   - Claude Sonnet only for promising jobs
   - Savings: ~50% on low-quality jobs

3. Cache evaluations for recurring jobs:
   - Check if job_id already evaluated
   - Skip AI if evaluated recently
   - Savings: ~20% for recurring postings

4. Reduce profile size:
   - Shorter resume in profile
   - Fewer input tokens
   - Savings: ~10-15%

**Alternative Free/Cheaper Options:**
1. Use open-source models (Llama, Mistral)
   - Self-host on GPU server
   - One-time cost vs. per-use
2. Use job board APIs (some free)
   - Indeed, LinkedIn (limited)
   - Greenhouse, Lever (if available)
3. Web scraping (DIY)
   - Free but requires maintenance
   - Legal considerations

### ROI Calculation

**Time Savings:**
- Manual job search: 2-3 hours/day
- Hourly value: $100-150/hour (for senior PM)
- Daily value: $200-450 saved

**Cost:**
- Daily: $1.84
- Monthly: $55

**ROI:**
- Daily: $200 saved / $1.84 cost = 109x return
- Monthly: $6,000 saved / $55 cost = 109x return
- **Annual ROI: >10,000%**

**Intangible Benefits:**
- Never miss opportunities
- Better market awareness
- Data-driven job search
- Reduced stress and cognitive load
- Historical data for market analysis

**Break-even:**
- Saves 30-60 seconds per day
- Breaks even immediately
- Pays for itself many times over

---

## 🔧 Troubleshooting

### Common Issues

**1. No Email Received**

**Symptoms**: Workflow completes but no email in inbox

**Diagnosis**:
```sql
-- Check if jobs in email queue
SELECT COUNT(*) FROM email_queue 
WHERE workflow_run_id = [latest_run_id];

-- Check Gmail node execution
-- Review n8n execution log for Send Email workflow
```

**Solutions**:
- Verify Gmail API credentials not expired
- Check Gmail API quotas not exceeded
- Check spam folder
- Verify recipient email address
- Review Send Email workflow execution log

---

**2. Apify API Errors**

**Symptoms**: All companies return 0 jobs or api_error

**Diagnosis**:
```sql
-- Check error logs
SELECT * FROM errors 
WHERE workflow_run_id = [latest_run_id]
ORDER BY timestamp DESC;
```

**Solutions**:
- Verify Apify API key valid
- Check Apify credit balance
- Review Apify platform status
- Check rate limiting
- Verify domain format in companies table

---

**3. Profile Not Loading (NEW in v2.1)**

**Symptoms**: AI evaluation fails or uses wrong profile

**Diagnosis**:
```sql
-- Check if profile exists
SELECT * FROM candidate_profile 
WHERE profile_id = 'ted_beatie_pm';

-- Check if profile data complete
SELECT 
  LENGTH(resume_text) as resume_length,
  target_criteria,
  updatedAt
FROM candidate_profile;
```

**Solutions**:
- Verify profile_id matches workflow configuration
- Check resume_text field not empty
- Validate target_criteria JSON format
- Ensure profile loaded in Loop Companies Start node
- Review Merge Data node execution

---

**4. AI Evaluation Failures**

**Symptoms**: Jobs not appearing in email queue

**Diagnosis**:
```sql
-- Check raw jobs vs email queue counts
SELECT 
  (SELECT COUNT(*) FROM jobs_test_1 WHERE workflow_run_id = X) as raw_jobs,
  (SELECT COUNT(*) FROM email_queue WHERE workflow_run_id = X) as evaluated_jobs;
```

**Solutions**:
- Verify Anthropic API key valid
- Check API rate limits
- Review Parse AI Response node errors
- Verify _context_resume_text field present (v3.1)
- Check AI response format

---

**5. Workflow Hangs/Doesn't Complete**

**Symptoms**: Workflow running for >15 minutes

**Diagnosis**:
- Check n8n execution log for stuck node
- Review Loop Over Companies progress
- Check for infinite loop scenarios

**Solutions**:
- Cancel and restart workflow
- Check database connections
- Verify loop nodes configured correctly
- Review error handling paths
- Check if sub-workflow hung

---

**6. Wrong Profile Used in Evaluation (v3.1)**

**Symptoms**: AI evaluation doesn't match expected profile

**Diagnosis**:
```javascript
// Check context fields in Loop Jobs workflow
console.log($json._context_profile_id);
console.log($json._context_resume_text.substring(0, 100));
```

**Solutions**:
- Verify correct profile loaded in Loop Companies
- Check Merge Data node configuration
- Verify Add Context node passes profile fields
- Review _context_resume_text in job data

---

### Debugging Workflow Issues

**Step 1: Identify Failed Node**
- Open workflow execution in n8n
- Look for red (error) or yellow (warning) nodes
- Click node to view error details

**Step 2: Check Input Data**
- Review input tab on failed node
- Verify expected fields present
- Check data types and formats

**Step 3: Test Node Independently**
- Use "Execute node" feature
- Pin test data to node input
- Verify node logic works

**Step 4: Review Database**
```sql
-- Check data made it to database
SELECT * FROM jobs_test_1 
WHERE workflow_run_id = [latest_run_id]
LIMIT 10;

SELECT * FROM email_queue 
WHERE workflow_run_id = [latest_run_id]
LIMIT 10;

SELECT * FROM errors 
WHERE workflow_run_id = [latest_run_id];
```

**Step 5: Check API Status**
- [Apify Status](https://status.apify.com)
- [Anthropic Status](https://status.anthropic.com)
- [Google Workspace Status](https://www.google.com/appsstatus)

---

### Profile Troubleshooting (v2.1/v3.1)

**Profile Not Loading:**
```sql
-- Verify profile exists
SELECT profile_id, profile_name, 
       LENGTH(resume_text) as resume_chars,
       LENGTH(target_criteria) as criteria_chars
FROM candidate_profile;
```

**Profile Data Incomplete:**
```sql
-- Check for NULL fields
SELECT profile_id,
       CASE WHEN resume_text IS NULL THEN 'MISSING' ELSE 'OK' END as resume,
       CASE WHEN target_criteria IS NULL THEN 'MISSING' ELSE 'OK' END as criteria
FROM candidate_profile;
```

**Wrong Profile Used:**
- Check Loop Companies Start node configuration
- Verify profile_id being loaded
- Check Merge Data node receives correct profile

---

## ✅ Production Checklist

### Pre-Launch

**Profile Setup (v2.1):**
- [ ] Create candidate_profile table
- [ ] Insert your profile with resume_text
- [ ] Format target_criteria as valid JSON
- [ ] Test profile loading in Loop Companies
- [ ] Verify _context_resume_text appears in jobs

**Database Setup:**
- [ ] All tables created (companies, candidate_profile, jobs_test_1, email_queue, errors)
- [ ] Indexes added (workflow_run_id, company_id, etc.)
- [ ] Test data inserted (at least 5 companies)
- [ ] Backup strategy configured

**API Credentials:**
- [ ] Apify API key configured and tested
- [ ] Anthropic API key configured and tested
- [ ] Gmail OAuth2 configured and authorized
- [ ] All credentials stored in n8n securely

**Workflows Configured:**
- [ ] Loop Companies v2.1: All nodes tested
  - [ ] Profile loading works
  - [ ] Merge Data combines profile + companies
  - [ ] Build Request extracts criteria from profile
- [ ] Loop Jobs v3.1: All nodes tested
  - [ ] Profile arrives via _context_resume_text
  - [ ] AI evaluation uses profile correctly
- [ ] Send Email v1.1: All nodes tested

**Testing:**
- [ ] Run full pipeline end-to-end
- [ ] Verify email received
- [ ] Check all database tables populated
- [ ] Test error scenarios (API down, no results)
- [ ] Verify costs match estimates
- [ ] Test with different profile (if multi-user)

### Post-Launch

**Monitoring:**
- [ ] Set up daily execution schedule
- [ ] Configure email alerts for failures
- [ ] Monitor API costs daily
- [ ] Review email quality weekly
- [ ] Check database growth monthly

**Maintenance:**
- [ ] Clean up old email_queue entries (30+ days)
- [ ] Clean up old jobs_test_1 entries (90+ days)
- [ ] Review and update company list monthly
- [ ] Update profile as needed (no workflow changes!)
- [ ] Backup database weekly

**Optimization:**
- [ ] Review AI evaluation accuracy
- [ ] Adjust scoring criteria if needed
- [ ] Update target_criteria in profile as job market changes
- [ ] Monitor for new companies to add
- [ ] Track time/cost savings

---

## ❓ FAQ

### General Questions

**Q: How does the system work?**  
A: Three-workflow pipeline:
1. Loop Companies discovers jobs from 40 target companies
2. Loop Jobs evaluates each job with AI (Claude Sonnet 4.5)
3. Send Email delivers professional daily digest

Total time: ~8 minutes. Total cost: ~$1.84/day.

---

**Q: What's new in v2.1/v3.1?**  
A: Major profile externalization update:
- Profile moved from hardcoded workflow code to database table
- Loop Companies loads profile and passes via _context_* fields
- Loop Jobs uses profile directly from context (no separate node)
- Eliminates personal data from workflow JSON
- Enables multi-user support
- Simplifies profile updates

**Architecture changes:**
- Loop Companies v2.1: +Merge Data node, profile loading, dynamic criteria
- Loop Jobs v3.1: -Add Resume Context node (eliminated), 8 nodes (down from 9)
- Send Email v1.1: No changes (same architecture)

---

**Q: Why use three separate workflows?**  
A: Benefits:
1. **Separation of concerns**: Discovery, evaluation, delivery are distinct
2. **Independent testing**: Test each workflow separately
3. **Reusability**: Loop Jobs can be called for different use cases
4. **Fault tolerance**: Error in one doesn't crash others
5. **Easier maintenance**: Modify one without affecting others

---

### Technical Questions

**Q: What if Apify API is down?**  
A: Workflow handles this gracefully:
- All companies will return api_error status
- Errors logged to errors table
- Workflow completes without crashing
- Re-run when Apify is back up

---

**Q: Can I modify the AI evaluation criteria?**  
A: Yes, in v2.1 it's easier than ever:
1. Update target_criteria in candidate_profile table (no workflow changes!)
2. Optionally modify scoring criteria in Loop Jobs → Message a model prompt
3. Optionally adjust threshold in email filtering

**Example:**
```sql
UPDATE candidate_profile
SET target_criteria = '{
  "titleSearch": "Director OR VP OR \"Head of Product\"",
  "titleExclusionSearch": "Associate OR Junior",
  "minSalary": 250000
}'
WHERE profile_id = 'ted_beatie_pm';
```

---

**Q: How do I add a new company to monitor?**  
A: Simple:
1. Add row to companies table:
   ```sql
   INSERT INTO companies (company_id, company_name, domain, status)
   VALUES ('newco', 'NewCo Inc', 'newco.com', 'active');
   ```
2. No workflow changes needed
3. Next run will automatically process new company

---

**Q: Can I use this for multiple people?**  
A: Yes! v2.1 enables multi-user support:

**Option 1: Multiple Profiles, Single Workflow**
```sql
-- Add profile for each person
INSERT INTO candidate_profile (profile_id, profile_name, resume_text, target_criteria)
VALUES ('jane_doe_pm', 'Jane Doe', 'Resume text...', '{"titleSearch": "..."}');
```
- Modify Loop Companies to load specific profile_id
- Or pass profile_id as workflow parameter

**Option 2: Separate Workflows**
- Clone workflows for each person
- Each hardcoded to specific profile_id
- Independent schedules

---

**Q: How do I update my resume or job criteria?**  
A: Much easier in v2.1 - just update the database:

```sql
-- Update resume
UPDATE candidate_profile
SET resume_text = 'Updated resume text...',
    updatedAt = NOW()
WHERE profile_id = 'ted_beatie_pm';

-- Update job criteria
UPDATE candidate_profile
SET target_criteria = '{"titleSearch": "VP OR Director", "minSalary": 300000}',
    updatedAt = NOW()
WHERE profile_id = 'ted_beatie_pm';
```

**No workflow editing required!** Changes take effect on next run.

---

**Q: Can I filter email to only show excellent matches?**  
A: Yes, modify Load Email Queue query:
```sql
WHERE workflow_run_id = X 
AND recommendation = 'EXCELLENT_MATCH'
```
Or filter in Aggregate and Sort Jobs node.

---

**Q: What if I want weekly digests instead of daily?**  
A: Two approaches:
1. **Schedule weekly**: Change cron to weekly, increase timeRange to "7d"
2. **Accumulate**: Run daily, modify email query to include last 7 days of workflow_run_ids

---

**Q: Can I send to multiple recipients?**  
A: Yes:
- Gmail node: Comma-separated in sendTo field
- Or: Add loop in Send Email to send individually
- Or: Use BCC for privacy

---

**Q: How do I test without sending email?**  
A: Several options:
1. Disable Send Email workflow
2. Change recipient to test email
3. Use n8n's "Execute workflow" test mode
4. Check email_queue table directly

---

### Data & Privacy Questions

**Q: Where is my data stored?**  
A: All data stored in your n8n database:
- companies: Target companies list
- candidate_profile: Your resume and criteria (NEW v2.1)
- jobs_test_1: Raw job listings
- email_queue: Formatted cards
- errors: Monitoring logs

**Data sent to third parties:**
- Apify: Company domains for search (no personal data)
- Anthropic: Job descriptions + profile for evaluation (encrypted in transit)
- Gmail: Recipient email for delivery

---

**Q: Is my profile secure?**  
A: Much more secure in v2.1:
- Stored in database (not hardcoded in workflows)
- Can encrypt profile data in database
- Access controlled via database permissions
- No profile data in workflow JSON (safe for version control)
- Not stored by Anthropic (per their privacy policy)
- Encrypted in transit to Anthropic API

**Before v2.1**: Profile hardcoded in workflow JSON, exposed in version control
**After v2.1**: Profile in database, can be encrypted, access controlled

---

**Q: Can I delete historical data?**  
A: Yes, safe to delete:
```sql
-- Delete old email queue (>30 days)
DELETE FROM email_queue 
WHERE processed_at < NOW() - INTERVAL '30 days';

-- Delete old jobs (>90 days)
DELETE FROM jobs_test_1 
WHERE date_posted < NOW() - INTERVAL '90 days';
```

---

**Q: What data is sent to AI?**  
A: Only what's needed for evaluation:
- Your resume/profile (from candidate_profile table)
- Job description (from Apify)
- Job metadata (title, company, salary, location)
- No personal identifiers
- No application data

**v2.1 improvement**: Profile loaded from secure database, not hardcoded

---

### Customization Questions

**Q: Can I add more job sources beyond Apify?**  
A: Yes:
1. Add new API node in Loop Companies
2. Format to match expected schema
3. Merge with Apify results
4. Rest of pipeline handles automatically

---

**Q: Can I modify the email design?**  
A: Yes, in Build Email node:
- Update HTML template
- Change colors/styling
- Add/remove sections
- Test by viewing node output

---

**Q: Can I add different scoring dimensions?**  
A: Yes, in Loop Jobs:
1. Update AI prompt with new dimensions
2. Update Parse AI Response to extract them
3. Update Format Job Card to display them

---

**Q: Can I save emails to database instead of sending?**  
A: Yes:
- Replace Send Jobs Email node with Insert node
- Save html_body to emails table
- Can send later or view in database

---

**Q: Can I export data to CSV?**  
A: Yes, use n8n export features or SQL:
```sql
COPY (
  SELECT job_title, company_name, overall_score, recommendation, job_url
  FROM email_queue
  WHERE workflow_run_id = 67890
) TO '/path/to/jobs.csv' WITH CSV HEADER;
```

---

### Cost & Performance Questions

**Q: What if costs increase?**  
A: Optimization strategies:
1. Reduce monitored companies (40 → 20 best)
2. Increase timeRange (7d → 14d, run less frequently)
3. Pre-filter jobs before AI (by title/salary)
4. Use cheaper AI model for initial screening

---

**Q: Can I speed up execution?**  
A: Some options:
1. Reduce job limit per company (10 → 5)
2. Reduce timeRange (7d → 3d, fewer jobs)
3. Parallel company processing (loses fault tolerance)
4. Use faster AI model (Haiku vs Sonnet)

Trade-offs: Speed vs completeness, cost vs quality

---

**Q: What's the ROI of this system?**  
A: Significant:
- Time saved: 2-3 hours/day (manual searching)
- Your hourly value: $100-150/hour
- Daily value: $200-450
- Daily cost: $1.84
- ROI: >10,000%

Plus intangible benefits:
- Never miss opportunities
- Better job market awareness
- Data-driven decision making

---

## 📚 Additional Resources

### n8n Documentation
- [Workflows](https://docs.n8n.io/workflows/)
- [Sub-workflows](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.executeworkflow/)
- [Loop nodes](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.splitinbatches/)
- [Code node](https://docs.n8n.io/code-examples/)
- [Error handling](https://docs.n8n.io/workflows/error-handling/)

### API Documentation
- [Apify Career Site API](https://apify.com/fantastic-jobs/career-site-job-listing-api)
- [Anthropic Claude API](https://docs.anthropic.com/claude/reference/getting-started-with-the-api)
- [Gmail API](https://developers.google.com/gmail/api/guides)

### Community
- [n8n Community Forum](https://community.n8n.io/)
- [n8n Discord](https://discord.gg/n8n)

---

## 🙋 Support

### Getting Help

**Workflow Issues:**
1. Check execution logs in n8n
2. Review this documentation
3. Test components in isolation
4. Check database for data issues
5. Check profile data if evaluation issues (v2.1)

**API Issues:**
- Apify: Check [status.apify.com](https://status.apify.com)
- Anthropic: Check [status.anthropic.com](https://status.anthropic.com)
- Gmail: Check [Google Workspace Status](https://www.google.com/appsstatus)

**Database Issues:**
1. Verify table structure
2. Check indexes exist
3. Review query performance
4. Check disk space
5. Verify profile data integrity (v2.1)

**Profile Issues (v2.1):**
1. Check candidate_profile table exists
2. Verify profile_id matches workflow
3. Validate resume_text not empty
4. Check target_criteria JSON format
5. Review _context_ fields in job data

---

## 📝 Version History

**v3.1** (Current) - Profile Externalization
- Loop Companies v2.1: Profile loaded from database
- Loop Jobs v3.1: Profile used from context (Add Resume Context node eliminated)
- Send Email v1.1: No changes
- Added candidate_profile database table
- Profile passed via _context_* fields
- Eliminated personal data from workflow JSON
- Multi-user support enabled
- Simplified architecture (fewer nodes)
- Enhanced security (profile can be encrypted)
- Improved documentation with profile management section

**v3.0** - Three-Workflow Architecture
- Split into Loop Companies, Loop Jobs, Send Email
- Added sub-workflow pattern
- Improved fault tolerance
- Enhanced email formatting
- Added comprehensive documentation

**v2.0** - Dual-Loop Architecture (v5)
- Nested company and job loops
- AI evaluation integration
- HTML card generation
- Email queue pattern
- Profile hardcoded in workflow (security concern)

**v1.0** - Domain Validation
- Single workflow
- Company domain verification
- Basic Apify integration

---

**End of Documentation**

*Last Updated: December 30, 2025*
*Workflow Version: v2.1/v3.1 (Profile Externalization)*
*Documentation Version: 2.0*
*System Name: RoleRadar*
