# Job Monitoring System - Complete Documentation

## 📋 Table of Contents
1. [Overview](#overview)
2. [System Architecture](#system-architecture)
3. [Workflow 1: Loop Companies](#workflow-1-loop-companies)
4. [Workflow 2: Loop Jobs](#workflow-2-loop-jobs)
5. [Workflow 3: Send Email](#workflow-3-send-email)
6. [Key Concepts](#key-concepts)
7. [Database Schema](#database-schema)
8. [Data Flow & Traceability](#data-flow--traceability)
9. [Cost Analysis](#cost-analysis)
10. [Troubleshooting](#troubleshooting)
11. [Production Checklist](#production-checklist)
12. [FAQ](#faq)

---

## 🎯 Overview

### Purpose
Automated job monitoring system that discovers, evaluates, and delivers personalized product management job recommendations via daily email digest.

### Business Problem
As a senior technical product manager, you need to:
- Monitor 40+ target companies for relevant PM positions
- Evaluate job fit across 5 dimensions (seniority, domain, technical depth, location, compensation)
- Receive daily digest of best matches without manual searching
- Track historical job postings for market analysis

### Solution Architecture
**Three-workflow pipeline:**
1. **Loop Companies**: Discovers jobs from target companies via Apify API
2. **Loop Jobs**: Evaluates each job with AI (Claude Sonnet 4.5)
3. **Send Email**: Aggregates results and delivers professional digest via Gmail

### Key Metrics
- **Processing Time**: ~7-8 minutes for 40 companies
- **Cost per Day**: ~$0.84 (120 jobs × $0.003 AI + $0.48 Apify)
- **Deliverables**: Single daily email with all jobs sorted by AI score
- **Success Rate**: >99% with fault-tolerant architecture

---

## 🏗️ System Architecture

### High-Level Pipeline
```
┌─────────────────────────────────────────────────────────────────┐
│                    WORKFLOW 1: LOOP COMPANIES                    │
│  Purpose: Discover jobs from target companies                   │
│  Input: companies table (40 companies)                          │
│  Output: Raw jobs → jobs table, context → email_queue          │
└────────────────────┬────────────────────────────────────────────┘
                     │ For each company, calls ↓
┌─────────────────────────────────────────────────────────────────┐
│                     WORKFLOW 2: LOOP JOBS                        │
│  Purpose: AI evaluation of each job                             │
│  Input: Jobs with context from parent workflow                  │
│  Output: Formatted job cards → email_queue table                │
└────────────────────┬────────────────────────────────────────────┘
                     │ When all companies complete, calls ↓
┌─────────────────────────────────────────────────────────────────┐
│                    WORKFLOW 3: SEND EMAIL                        │
│  Purpose: Aggregate and deliver daily digest                    │
│  Input: workflow_run_id from parent                             │
│  Output: Professional HTML email via Gmail                      │
└─────────────────────────────────────────────────────────────────┘
```

### Workflow Interactions
```
Loop Companies (Parent)
├─ Loads: companies table
├─ Iterates: One company at a time
├─ For each company:
│  ├─ Calls: Apify API (get jobs)
│  ├─ Saves: Raw jobs → jobs table
│  ├─ Calls: Loop Jobs sub-workflow
│  │  ├─ Receives: Jobs with _context_ fields
│  │  ├─ Iterates: One job at a time
│  │  ├─ Evaluates: Claude AI scoring
│  │  ├─ Formats: HTML job cards
│  │  └─ Saves: Formatted cards → email_queue table
│  └─ Loop back: Next company
├─ When all complete:
│  └─ Calls: Send Email sub-workflow
│     ├─ Queries: email_queue WHERE workflow_run_id = X
│     ├─ Aggregates: All jobs, sort by score
│     ├─ Generates: Professional HTML email
│     └─ Sends: Via Gmail API
└─ Complete: User receives daily digest
```

### Key Design Principles

**1. Separation of Concerns**
- **Workflow 1**: Company orchestration and job discovery
- **Workflow 2**: Individual job evaluation and formatting
- **Workflow 3**: Email aggregation and delivery

**2. Fault Tolerance**
- Companies processed sequentially (one failure doesn't stop others)
- Jobs processed sequentially (AI rate limiting, error isolation)
- Error logging at each level (companies, jobs, email)

**3. Clean Data Flow**
- Context fields (_context_*) pass data across workflow boundaries
- workflow_run_id links all jobs from single execution
- Database tables accumulate data for email generation

**4. Independent Testing**
- Each workflow can be tested separately
- Pin data allows manual testing without dependencies
- Email workflow can regenerate from existing queue data

---

## 🔄 Workflow 1: Loop Companies

### Purpose
Orchestrates company-by-company job discovery and coordinates sub-workflow execution.

### Node Structure (14 nodes)
```
Manual Trigger
  ↓
Load Companies (40 companies from table)
  ↓
Build Apify Requests (format API parameters)
  ↓
Loop Over Companies ────┐
  ↓                      │
  Run Apify API          │
    ↓ Success  ↓ Error   │
    IF        Log Error  │
    ↓True ↓False  ↓      │
    Add    Log   Insert  │
   Context  No   Bad Row │
    ↓↓      Res    ↓     │
    ││      ↓      │     │
    │└─Save Apify──┘     │
    │   Data →           │
    │   Insert Raw Jobs  │
    │                    │
    └─Call Loop Jobs ────┤
      (sub-workflow)     │
      ↓                  │
      Loop back ─────────┘
      
When all companies complete:
  ↓
Workflow Summary
  ↓
[Ready to call Send Email workflow]
```

### Critical Nodes Explained

**NODE 4: Loop Over Companies**
- **Type**: Split In Batches (batch size: 1)
- **Purpose**: Process companies one at a time
- **Why**: Fault tolerance - if company #25 fails, #26-40 still process
- **Two outputs**:
  - Loop branch (bottom): Fires once per company
  - Done branch (top): Fires once when all complete

**NODE 5: Run Apify**
- **API**: Career Site Job Listing API
- **Returns**: 0-10 jobs per company
- **Settings**:
  - `alwaysOutputData: true` - Output even if 0 jobs
  - `onError: continueErrorOutput` - Don't crash on errors
- **Two outputs**:
  - Success: API succeeded (0+ jobs)
  - Error: API failed (timeout, auth, network)

**NODE 7: Add Context**
- **Purpose**: Enrich jobs with parent workflow context
- **Adds**:
  - `_context_workflow_run_id`: Parent execution ID
  - `_context_company_id`: Company identifier
  - `_context_company_name`: Company display name
  - `_context_domain`: Company domain
- **Why**: Sub-workflow needs parent context to track jobs

**NODE 10: Call 'Loop Jobs'**
- **Type**: Execute Workflow
- **Calls**: Loop Jobs sub-workflow
- **Passes**: All jobs with _context_ fields
- **Waits**: For sub-workflow to complete all jobs
- **Returns**: Control to company loop when done

**NODE 8: Save Apify data → Insert row**
- **Parallel path**: Runs alongside sub-workflow call
- **Purpose**: Immediate raw job backup
- **Fast**: ~100ms per job
- **Does NOT loop back**: Ends after insert

### Key Behaviors

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
Add Context → Call 'Loop Jobs' → [Loops back when all jobs complete]
```
- Calls sub-workflow for AI evaluation
- Takes 2-5 seconds per job
- Loops back to process next company

**Both paths must complete before company loop advances.**

### Database Tables Used
1. **companies** (input): Target companies to monitor
2. **jobs** (output): Raw job data with searchable fields
3. **errors** (output): API failures and no-results tracking

### Error Handling
- **API Error**: Logged, workflow continues to next company
- **No Results**: Logged separately, workflow continues
- **All errors**: Feed back to Loop Over Companies for next iteration

---

## 🤖 Workflow 2: Loop Jobs

### Purpose
Evaluates individual jobs with AI, formats HTML cards, saves to email queue.

### Node Structure (9 nodes)
```
Manual Trigger (or called from parent)
  ↓
Loop Over Jobs ────────┐
  ↓                     │
Add Resume Context      │
  ↓↓ (parallel split)   │
  ││                    │
  │└──────────┐         │
  ↓           ↓         │
Message a  [Raw job    │
model (AI)  passthrough]│
  ↓           ↓         │
Parse AI     ↓          │
Response     ↓          │
  ↓           ↓         │
  └────Merge──┘         │
       ↓                │
  Format Job Card       │
       ↓                │
  Save to Email Queue   │
       ↓                │
  Loop back ────────────┘
  
When all jobs complete:
  ↓
Workflow Complete
  ↓
[Returns to parent workflow]
```

### Critical Nodes Explained

**NODE 2: Loop Over Jobs**
- **Type**: Split In Batches (batch size: 1)
- **Purpose**: Process jobs one at a time
- **Why**: AI API rate limiting, cost tracking, error isolation
- **Input**: All jobs for current company (e.g., 8 jobs from Anthropic)
- **Two outputs**:
  - Loop branch: Process each job
  - Done branch: Signal completion to parent

**NODE 3: Add Resume Context**
- **Purpose**: Attach candidate profile to each job
- **Adds**: Complete resume, experience, target criteria
- **Why here**: Profile only needed for AI, not raw storage
- **Output splits to TWO parallel paths**:

**Path A (AI Evaluation - Slow):**
```
Add Resume Context → Message a model → Parse AI Response → Merge
```
- **Duration**: 2-5 seconds per job
- **Cost**: ~$0.003 per job
- **Returns**: AI evaluation with scores

**Path B (Raw Data - Fast):**
```
Add Resume Context → Merge (direct connection)
```
- **Duration**: Instant
- **Cost**: Free
- **Preserves**: Complete job data for merging

**NODE 4: Message a model**
- **Model**: Claude Sonnet 4.5 (claude-sonnet-4-5-20250929)
- **Input**: Job details + candidate profile
- **Output**: JSON evaluation with:
  - `overall_score` (1-10)
  - `recommendation` (EXCELLENT_MATCH | GOOD_MATCH | CONSIDER | POOR_FIT | REJECT)
  - `detailed_scores` (5 dimensions: seniority, domain, technical, location, compensation)
  - `reasoning` (strengths, weaknesses, key_mismatches arrays)
  - `match_explanation` (2-4 sentence summary)

**NODE 5: Parse AI Response**
- **Purpose**: Extract and structure JSON from Claude
- **Process**:
  1. Remove markdown code fences (```json)
  2. Parse JSON string
  3. Flatten structure for database
- **Critical**: Extracts `job_id` for merge matching

**NODE 6: Merge**
- **Type**: Merge By Fields
- **Matches**: AI evaluation (`job_id`) with raw job (`id`)
- **Input 1**: AI evaluation from Parse AI Response
- **Input 2**: Complete job data from Add Resume Context
- **Output**: Single item with BOTH AI scores AND all job fields

**NODE 7: Format Job Card**
- **Purpose**: Generate professional HTML card
- **Features**:
  - Color-coded badges by recommendation
  - Compact 2-column detail layout
  - Inline AI scores
  - Expandable strengths/weaknesses/mismatches
  - Responsive design (email-safe)
- **Output**: Complete HTML card (~2KB)

**NODE 8: Save to Email Queue**
- **Table**: email_queue
- **Stores**:
  - `workflow_run_id`: Links to parent execution
  - `overall_score`: For sorting
  - `recommendation`: For filtering
  - `html_card`: Complete formatted card
  - `company_name`: For grouping
  - `job_id`, `job_title`: For reference
- **Purpose**: Accumulate all jobs for email generation

**NODE 9: Workflow Complete**
- **Purpose**: Signal completion to parent
- **Returns**:
  - `status: 'success'`
  - `jobs_processed`: Count of jobs evaluated
  - `timestamp`: Completion time
- **Parent receives**: Confirmation signal, advances to next company

### Parallel Processing Pattern

**Why Two Paths After Add Resume Context?**

Without parallel paths, we'd lose job data after AI evaluation:
```
❌ WRONG: Add Context → AI → Parse → Format → Save
   Problem: AI node only returns evaluation, not original job fields
```

With parallel paths, we preserve everything:
```
✅ CORRECT: Add Context ──┬─→ AI → Parse ──┐
                          │                ↓
                          └──→ (preserve) → Merge → Format → Save
```

The Merge node combines:
- AI evaluation (scores, recommendation, reasoning)
- Complete job data (title, company, salary, location, description)

Result: Format Job Card has everything needed for rich HTML cards.

### Database Table Used
- **email_queue** (output): Accumulates formatted job cards

---

## 📧 Workflow 3: Send Email

### Purpose
Queries email queue, aggregates jobs by score, generates and sends daily digest.

### Node Structure (5 nodes)
```
Manual Trigger (or called from parent)
  ↓
Load Email Queue
  (Query: WHERE workflow_run_id = X)
  ↓
  Returns 20 separate job items
  ↓
Aggregate and Sort Jobs
  (Consolidate to single item with stats)
  ↓
Build Email
  (Generate HTML with dynamic subject)
  ↓
Send Jobs Email
  (Deliver via Gmail API)
  ↓
[Complete - User receives email]
```

### Critical Nodes Explained

**NODE 2: Load Email Queue**
- **Table**: email_queue
- **Query**: `WHERE workflow_run_id = {{ $json.workflow_run_id }}`
- **Returns**: All jobs from current workflow run
- **Example**: 
  - 40 companies processed
  - 120 jobs found total
  - All 120 jobs returned in query
- **Automatic iteration**: n8n creates separate item per row

**NODE 3: Aggregate and Sort Jobs**
- **Input**: 120 separate job items
- **Process**:
  1. Group by recommendation (excellent, good, consider, poor)
  2. Sort by overall_score descending (best first)
  3. Concatenate HTML cards with spacers
  4. Calculate statistics
- **Output**: Single item with:
  - `total_jobs`: 120
  - `excellent_count`: 15
  - `strong_count`: 45
  - `consider_count`: 40
  - `poor_count`: 20
  - `job_cards_html`: All cards combined
  - `workflow_run_id`: For tracking

**NODE 4: Build Email**
- **Purpose**: Generate professional HTML email
- **Features**:
  - **Dynamic subject line**:
    - "🎯 15 Excellent Matches, 45 Strong • 120 Total Jobs"
    - "📊 5 New Jobs to Review" (if no excellent/strong)
  - **Summary statistics card**: 4-column grid with counts
  - **Conditional highlight**: Banner for excellent matches
  - **Job cards section**: All cards sorted by score
  - **Footer**: System info and workflow tracking
- **Design principles**:
  - Inline styles (email client compatibility)
  - Responsive layout (700px max width)
  - Color coding (green/blue/amber/gray)
  - Progressive enhancement
- **Output**: Complete HTML email (~15-20KB)

**NODE 5: Send Jobs Email**
- **Service**: Gmail API (OAuth2)
- **Recipient**: tcbeatie@gmail.com
- **Subject**: Dynamic from Build Email
- **Body**: HTML email from Build Email
- **Delivery**:
  - Appears in Gmail Sent folder
  - Threads with replies
  - High deliverability rate
- **Error handling**:
  - Authentication errors (re-auth)
  - Rate limiting (100/sec, 10k/day)
  - Network failures (retry)

### Email Structure

**Header Section:**
```
┌────────────────────────────────────┐
│  🎯 Your Daily Job Digest          │
│  Monday, December 29, 2025         │
└────────────────────────────────────┘
```

**Summary Statistics:**
```
┌─────────┬─────────┬──────────┬─────────┐
│ Excellent│  Strong │ Consider │  Total  │
│    15    │   45    │    40    │   120   │
└─────────┴─────────┴──────────┴─────────┘
```

**Job Cards (sorted by score):**
```
[Job 1 - Score 10 - Excellent Match]
[Job 2 - Score 9 - Excellent Match]
[Job 3 - Score 9 - Good Match]
...
[Job 120 - Score 3 - Poor Fit]
```

**Footer:**
```
📬 Automated Job Monitoring System
Workflow Run ID: #67890
```

### Database Table Used
- **email_queue** (input): Source of all job data

---

## 🔑 Key Concepts

### 1. Workflow Orchestration

**Sub-Workflow Pattern:**
```
Parent Workflow (Loop Companies)
  ├─ Manages: Company iteration
  ├─ Calls: Loop Jobs for each company (Execute Workflow node)
  └─ Calls: Send Email when all complete (Execute Workflow node)

Sub-Workflows (Loop Jobs, Send Email)
  ├─ Independent: Can be tested separately
  ├─ Receive: Data from parent via node input
  └─ Return: Control to parent when complete
```

**Benefits:**
- Clean separation of concerns
- Independent testing and debugging
- Reusable components
- Easier maintenance

### 2. Context Preservation

**The _context_ Pattern:**
```javascript
// Added in parent workflow (Loop Companies)
_context_workflow_run_id: $execution.id     // Parent's execution ID
_context_company_id: "anthropic"            // Company identifier
_context_company_name: "Anthropic"          // Display name
_context_domain: "anthropic.com"            // Company domain
```

**Purpose:**
- Pass parent context to sub-workflows
- Link jobs back to parent execution
- Enable cross-workflow tracking
- Group jobs for email generation

**Flow:**
```
1. Loop Companies sets: _context_workflow_run_id = 67890
2. Passed to Loop Jobs via job data
3. Loop Jobs saves to email_queue: workflow_run_id = 67890
4. Send Email queries: WHERE workflow_run_id = 67890
5. Gets all jobs from this workflow run
```

### 3. Fault Tolerance

**Company-Level:**
- Process one company at a time
- If company #25 fails, #26-40 continue
- Errors logged to database
- No results logged separately

**Job-Level:**
- Process one job at a time (AI rate limiting)
- If job #5 fails, jobs #6-8 continue
- Errors logged in execution
- Queue accumulates successful evaluations

**Email-Level:**
- Query from persistent database
- Can retry without re-running pipeline
- Gmail API retry logic
- Delivery confirmation

### 4. Parallel Processing

**Company Workflow (After Add Context):**
```
Path A: Save Apify data → Insert row → ENDS (fast, backup)
Path B: Call Loop Jobs → Loops back when complete (slow, AI eval)
```
- Path A: Immediate raw data storage
- Path B: Expensive AI processing
- Both complete before next company

**Job Workflow (After Add Resume Context):**
```
Path A: Message a model → Parse → Merge (slow, AI path)
Path B: Direct → Merge (fast, preserve job data)
```
- Path A: AI evaluation with scores
- Path B: Preserve complete job fields
- Merge combines both for formatting

### 5. Data Accumulation

**Email Queue Pattern:**
```
Company 1: 8 jobs  → email_queue (workflow_run_id: 67890)
Company 2: 5 jobs  → email_queue (workflow_run_id: 67890)
Company 3: 7 jobs  → email_queue (workflow_run_id: 67890)
...
Company 40: 3 jobs → email_queue (workflow_run_id: 67890)

Email workflow queries: WHERE workflow_run_id = 67890
Returns: All 120 jobs in single query
```

**Benefits:**
- Single daily email (not per-company)
- Sort all jobs by score
- Group by company
- Filter by recommendation
- Historical tracking

### 6. HTML Card Embedding

**Strategy:**
```
Format once in Loop Jobs → Store in email_queue → Use in Send Email
```

**Alternative (NOT used):**
```
Store job fields → Format in Send Email (requires duplicating logic)
```

**Benefits:**
- Format once, use everywhere
- Consistent styling guaranteed
- Simple email generation
- Can preview in database
- A/B test formats on same data

---

## 🗄️ Database Schema

### Table 1: companies (Input)
```
Purpose: Target companies to monitor
Used by: Loop Companies (input)

Columns:
- company_id: string (PK) - "anthropic", "openai"
- company_name: string - "Anthropic", "OpenAI"
- domain: string - "anthropic.com", "openai.com"
```

### Table 2: jobs (Output)
```
Purpose: Raw job data backup with searchable fields
Used by: Loop Companies (output)

Columns:
- company_id: string - Company identifier
- company_name: string - Company display name
- domain_derived: string - Actual company domain from job
- job_id: string (PK) - Apify job ID
- job_title: string - "Senior Technical Product Manager"
- date_posted: datetime - When job was posted
- locality: string - City
- region: string - State
- remote_derived: boolean - Is remote?
- salary_min: number - Minimum salary
- salary_max: number - Maximum salary
- job_url: string - Link to apply
- job_data: text - Complete JSON backup

Purpose:
- Backup all job data immediately
- Searchable by title, salary, location
- Historical tracking
- Independent of AI evaluation
```

### Table 3: errors (Output)
```
Purpose: Track API failures and no-results
Used by: Loop Companies (output)

Columns:
- company_id: string - Company identifier
- company_name: string - Company display name
- domain_guess: string - Domain searched
- status: string - "api_error" or "no_results"
- jobs_found: number - Always 0
- note: text - Error message or explanation
- verified_at: datetime - When error occurred

Purpose:
- Debug API failures
- Identify inactive companies
- Track monitoring coverage
- Maintenance alerts
```

### Table 4: email_queue (Output/Input)
```
Purpose: Accumulate formatted job cards for email
Used by: Loop Jobs (output), Send Email (input)

Columns:
- workflow_run_id: number (INDEX) - Parent execution ID
- recommendation: string - "EXCELLENT_MATCH", "GOOD_MATCH", etc.
- overall_score: number - 1-10 rating
- html_card: text - Complete formatted HTML card
- job_id: string - Job identifier
- company_name: string - Company posting job
- job_title: string - Job title

Purpose:
- Accumulate jobs across all companies
- Group by workflow_run_id for daily digest
- Pre-formatted for email insertion
- Sort by overall_score
- Filter by recommendation
- Historical email regeneration
```

### Indexes
```
CREATE INDEX idx_email_queue_workflow ON email_queue(workflow_run_id);
CREATE INDEX idx_email_queue_score ON email_queue(overall_score DESC);
CREATE INDEX idx_jobs_company ON jobs(company_id);
CREATE INDEX idx_jobs_posted ON jobs(date_posted DESC);
```

---

## 🔍 Data Flow & Traceability

### Complete Pipeline Flow

**Step 1: Company Discovery (Loop Companies)**
```
Input: companies table (40 companies)
Process:
  For each company:
    1. Call Apify API
    2. Get 0-10 jobs
    3. Add _context_ fields:
       - _context_workflow_run_id = 67890 (execution ID)
       - _context_company_id = "anthropic"
       - _context_company_name = "Anthropic"
       - _context_domain = "anthropic.com"
    4. Save raw jobs → jobs table (backup)
    5. Call Loop Jobs sub-workflow with enriched jobs

Output: 
  - jobs table: 120 raw jobs
  - errors table: 5 companies with no results
```

**Step 2: Job Evaluation (Loop Jobs - Called 40 times)**
```
Input: Jobs with _context_ fields from parent
Process:
  For each job:
    1. Add candidate profile
    2. AI evaluation (Claude Sonnet 4.5)
    3. Parse JSON response
    4. Merge AI scores with job data
    5. Format HTML card
    6. Save to email_queue:
       - workflow_run_id: 67890 (from _context_)
       - html_card: <formatted HTML>
       - overall_score: 9
       - recommendation: "EXCELLENT_MATCH"

Output:
  - email_queue table: 120 formatted job cards
  - All linked by workflow_run_id = 67890
```

**Step 3: Email Delivery (Send Email - Called once)**
```
Input: workflow_run_id = 67890 from parent
Process:
  1. Query email_queue: WHERE workflow_run_id = 67890
  2. Get all 120 jobs
  3. Sort by overall_score DESC
  4. Group by recommendation
  5. Generate dynamic subject line
  6. Build HTML email with all job cards
  7. Send via Gmail API

Output:
  - Email delivered to inbox
  - Subject: "🎯 15 Excellent Matches, 45 Strong • 120 Total Jobs"
  - Body: Professional HTML digest with all jobs
```

### Traceability Matrix

**How to trace a job through the entire system:**

```
1. Job discovered:
   - Loop Companies execution: 67890
   - Company: Anthropic (anthropic.com)
   - Apify job ID: 12345
   - Saved to jobs table with company_id = "anthropic"

2. Job evaluated:
   - Loop Jobs execution: 67891 (sub-workflow)
   - Received with _context_workflow_run_id = 67890
   - AI score: 9/10
   - Recommendation: EXCELLENT_MATCH
   - Saved to email_queue with workflow_run_id = 67890

3. Job emailed:
   - Send Email execution: 67892 (sub-workflow)
   - Queried with workflow_run_id = 67890
   - Sorted as #2 (score 9)
   - Included in email sent to user

4. User action:
   - Clicks job link in email
   - Can reference workflow_run_id = 67890 in footer
   - Can query database for other jobs from same run
```

### Database Queries for Traceability

**Find all jobs from specific workflow run:**
```sql
SELECT * FROM email_queue 
WHERE workflow_run_id = 67890 
ORDER BY overall_score DESC;
```

**Find raw data for job in email:**
```sql
SELECT j.* FROM jobs j
JOIN email_queue eq ON j.job_id = eq.job_id
WHERE eq.workflow_run_id = 67890;
```

**Track workflow history:**
```sql
SELECT 
  workflow_run_id,
  COUNT(*) as total_jobs,
  COUNT(CASE WHEN recommendation = 'EXCELLENT_MATCH' THEN 1 END) as excellent,
  AVG(overall_score) as avg_score,
  MIN(processed_at) as run_date
FROM email_queue
GROUP BY workflow_run_id
ORDER BY workflow_run_id DESC;
```

**Find errors for specific workflow run:**
```sql
SELECT * FROM errors
WHERE verified_at >= (
  SELECT MIN(processed_at) 
  FROM email_queue 
  WHERE workflow_run_id = 67890
);
```

---

## 💰 Cost Analysis

### Breakdown by Component

**1. Apify API Costs**
```
Cost: $4.00 per 1,000 jobs fetched
Rate: Varies by company

Daily scenario:
- 40 companies monitored
- Average 3 jobs per company
- Total: 120 jobs/day
- Cost: 120 × $0.004 = $0.48/day

Monthly: $0.48 × 30 = $14.40
Annual: $0.48 × 365 = $175.20
```

**2. Claude AI API Costs**
```
Model: Claude Sonnet 4.5
Cost: ~$0.003 per job evaluation

Input tokens: ~2,000 (profile + job description)
Output tokens: ~500 (structured evaluation)

Daily scenario:
- 120 jobs evaluated
- Cost: 120 × $0.003 = $0.36/day

Monthly: $0.36 × 30 = $10.80
Annual: $0.36 × 365 = $131.40
```

**3. Gmail API Costs**
```
Service: Free (included with Google Workspace)
Rate limits: 100 emails/second, 10,000/day per user
Daily usage: 1 email/day

Cost: $0.00/day
```

**4. n8n Hosting Costs**
```
Self-hosted: Server/VPS costs (varies)
Cloud (n8n.cloud): $20-50/month depending on plan
Assumed for this analysis: $30/month

Daily cost: $30 / 30 = $1.00/day
```

**Total Daily Cost:**
```
Apify:     $0.48
Claude AI: $0.36
Gmail:     $0.00
n8n:       $1.00
-----------------
Total:     $1.84/day

Monthly:  $55.20
Annual:   $671.80
```

### Cost Optimization Strategies

**1. Reduce Apify Calls**
```
Current: Query all 40 companies daily
Option A: Query high-priority companies daily, others weekly
  - Save: ~$10/month on Apify

Option B: Increase timeRange filter (e.g., 7d → 3d)
  - Same jobs, lower Apify processing
  - Save: ~$5/month

Option C: Cache results for 24 hours
  - Prevent duplicate API calls
  - Save: Varies by retry rate
```

**2. Reduce AI Evaluation Costs**
```
Current: Evaluate all jobs with AI

Option A: Pre-filter by title/salary before AI
  - Only evaluate jobs meeting minimum criteria
  - Potential savings: 30-50% of AI costs
  - Trade-off: Might miss edge cases

Option B: Use Claude Haiku for initial screening
  - Cheaper model ($0.001 vs $0.003)
  - Use Sonnet only for promising candidates
  - Save: 50-70% on AI costs

Option C: Batch evaluation
  - Evaluate multiple jobs in single API call
  - Save: 20-30% via reduced overhead
  - Trade-off: More complex prompt engineering
```

**3. Optimize Job Discovery**
```
Current: Search all companies daily
Option: Smart scheduling
  - Frequently hiring companies: Daily
  - Rarely hiring companies: Weekly
  - Inactive companies: Monthly check
  - Save: 40-60% on both Apify and AI costs
```

### ROI Analysis

**Value Delivered:**
- Time saved: 2-3 hours/day of manual job searching
- Your hourly rate: $100-150/hour (based on target compensation)
- Daily value: $200-450 in time saved

**Cost vs Value:**
```
Daily cost:  $1.84
Daily value: $200-450 (time saved)
ROI:         10,787% - 24,356%
```

**Payback period:** Less than 1 day

### Scaling Costs

**If monitoring 100 companies:**
```
Apify:     100 × 3 jobs × $0.004 = $1.20/day
Claude AI: 300 jobs × $0.003 = $0.90/day
Gmail:     Still $0.00
n8n:       Still $1.00
Total:     $3.10/day ($93/month)

Still highly cost-effective for time saved.
```

---

## 🛠️ Troubleshooting

### Common Issues & Solutions

#### Issue 1: No Email Received

**Symptoms:**
- Workflows complete successfully
- No email in inbox or spam

**Diagnosis:**
1. Check Send Email execution log
2. Verify Gmail API returned message ID
3. Check Gmail Sent folder
4. Verify recipient email address

**Solutions:**
- Re-authenticate Gmail OAuth2 credentials
- Check email in Sent folder (may not deliver to self)
- Test with different recipient address
- Verify Gmail API quota not exceeded

---

#### Issue 2: Email Queue Empty

**Symptoms:**
- Load Email Queue returns 0 rows
- Email shows "0 New Jobs to Review"

**Diagnosis:**
```sql
-- Check if jobs exist for workflow_run_id
SELECT COUNT(*) FROM email_queue 
WHERE workflow_run_id = 67890;

-- Check recent workflow runs
SELECT DISTINCT workflow_run_id 
FROM email_queue 
ORDER BY workflow_run_id DESC 
LIMIT 5;
```

**Possible Causes:**
1. **Wrong workflow_run_id**: Parent passed incorrect ID
2. **Loop Jobs failed**: Jobs not saved to queue
3. **No jobs discovered**: All companies returned 0 jobs
4. **Database error**: Queue inserts failed

**Solutions:**
- Check Loop Companies execution ID matches query
- Review Loop Jobs execution log for errors
- Check errors table for no_results entries
- Verify email_queue table permissions

---

#### Issue 3: AI Evaluation Fails

**Symptoms:**
- Loop Jobs execution shows errors
- Jobs saved to jobs table but not email_queue
- AI API timeout or authentication errors

**Diagnosis:**
1. Check Loop Jobs execution log
2. Look for "Message a model" node errors
3. Review Parse AI Response errors

**Possible Causes:**
1. **Authentication**: Claude API key expired/invalid
2. **Rate limiting**: Too many requests too fast
3. **Malformed prompt**: Job description too long
4. **Network**: Connectivity issues

**Solutions:**
- Re-authenticate Anthropic API credentials
- Add retry logic to AI node (3 retries, 1 min apart)
- Check job description length (trim if >10,000 chars)
- Verify network connectivity to api.anthropic.com
- Test with single job (pin data)

---

#### Issue 4: Duplicate Jobs in Email

**Symptoms:**
- Same job appears multiple times in email
- Email much longer than expected

**Diagnosis:**
```sql
-- Check for duplicates
SELECT job_id, COUNT(*) 
FROM email_queue 
WHERE workflow_run_id = 67890 
GROUP BY job_id 
HAVING COUNT(*) > 1;
```

**Possible Causes:**
1. **Workflow run twice**: Same workflow_run_id reused
2. **Loop not completing**: Partial run then retry
3. **Database constraint missing**: No unique key on job_id

**Solutions:**
- Clear email_queue before production runs
- Add unique constraint: `UNIQUE(workflow_run_id, job_id)`
- Use fresh execution for each daily run
- Check Loop Companies doesn't retry mid-run

---

#### Issue 5: Jobs Not Merging Correctly

**Symptoms:**
- Format Job Card errors "cannot read property 'title'"
- HTML cards missing job details
- Only AI scores visible, no job info

**Diagnosis:**
1. Check Merge node output
2. Verify both inputs reaching Merge
3. Check job_id matching

**Possible Causes:**
1. **job_id mismatch**: AI response job_id ≠ raw job id
2. **Merge timing**: One path completing before other
3. **Parse error**: job_id not extracted from AI response

**Solutions:**
- Verify AI prompt includes: `"job_id": "{{ $json.id }}"`
- Check Parse AI Response extracts job_id correctly
- Test Merge with pinned data from both inputs
- Ensure Add Resume Context preserves job.id field

---

#### Issue 6: Apify API Errors

**Symptoms:**
- Many companies show api_error status
- Loop Companies completes but few jobs found

**Diagnosis:**
```sql
-- Check error breakdown
SELECT status, COUNT(*) 
FROM errors 
GROUP BY status;

-- Check specific error messages
SELECT company_name, note 
FROM errors 
WHERE status = 'api_error';
```

**Possible Causes:**
1. **Authentication**: Apify API key invalid
2. **Rate limiting**: Exceeded Apify quota
3. **Network**: Connectivity to Apify
4. **Domain format**: Invalid domain in companies table

**Solutions:**
- Re-authenticate Apify credentials in n8n
- Check Apify dashboard for account status
- Verify domain format: "anthropic.com" (not "https://anthropic.com")
- Add delay between companies (rate limiting)
- Test with single company first

---

#### Issue 7: HTML Email Renders Poorly

**Symptoms:**
- Formatting broken in email client
- Cards don't display correctly
- Colors/styles missing

**Diagnosis:**
1. View HTML source in email
2. Test in different email clients
3. Check for inline styles

**Possible Causes:**
1. **Email client limitations**: Some clients strip CSS
2. **HTML structure**: Nested tables not supported
3. **Missing inline styles**: Styles in <style> tag stripped

**Solutions:**
- All styles must be inline: `style="color: red;"`
- Avoid CSS grid in older Outlook
- Test in Gmail, Outlook, Apple Mail
- Use email testing service (Litmus, Email on Acid)
- Keep HTML structure simple
- Provide plain text fallback

---

#### Issue 8: Workflow Stops Mid-Execution

**Symptoms:**
- Loop Companies processes 10/40 companies then stops
- No error message
- Execution shows "Running" but frozen

**Diagnosis:**
1. Check n8n execution log
2. Look for timeout errors
3. Check server resources (CPU, memory)

**Possible Causes:**
1. **Timeout**: Workflow exceeds n8n timeout
2. **Memory**: Server runs out of RAM
3. **Network**: Connection lost to APIs
4. **Loop structure**: Loop not properly connected

**Solutions:**
- Increase n8n execution timeout (Settings > Workflow)
- Add timeout to individual nodes (60s)
- Monitor server resources during execution
- Verify loop back connections
- Split into smaller batches if needed

---

### Debugging Best Practices

**1. Pin Data for Testing**
```
Test individual nodes without full workflow:
1. Right-click node → "Pin data"
2. Add sample data matching expected input
3. Execute node in isolation
4. Verify output
5. Unpin when done
```

**2. Check Execution Logs**
```
For each workflow execution:
1. Click execution in sidebar
2. Review each node's input/output
3. Look for red error indicators
4. Check node execution time
5. Review error messages
```

**3. Database Verification**
```
After each workflow run:
1. Query jobs table: COUNT(*)
2. Query email_queue: COUNT(*) WHERE workflow_run_id = X
3. Query errors: SELECT * WHERE status != 'success'
4. Verify counts match expectations
```

**4. Incremental Testing**
```
Test workflow components in order:
1. Test Loop Companies with 1 company
2. Test Loop Jobs with 1 job (pin data)
3. Test Send Email with test workflow_run_id
4. Test full pipeline with 3 companies
5. Scale to production (40 companies)
```

**5. Monitor Costs**
```
Track API usage:
- Apify dashboard: Check daily API calls
- Anthropic dashboard: Check token usage
- n8n: Check execution count
- Compare to expected (40 companies × 3 jobs = 120 calls)
```

---

## ✅ Production Checklist

### Pre-Deployment

**1. Database Setup**
- [ ] Create all required tables (companies, jobs, errors, email_queue)
- [ ] Add indexes: workflow_run_id, company_id, date_posted
- [ ] Set up unique constraints: UNIQUE(workflow_run_id, job_id)
- [ ] Configure backups (daily recommended)
- [ ] Test database connectivity from n8n

**2. API Credentials**
- [ ] Apify OAuth2: Connected and tested
- [ ] Anthropic API: Key valid, quota sufficient
- [ ] Gmail OAuth2: Connected, test email sent
- [ ] All credentials saved in n8n securely

**3. Configuration**
- [ ] Update recipient email in Send Email workflow
- [ ] Verify companies table has 40 companies
- [ ] Check domain format: "anthropic.com" (no https://)
- [ ] Set Apify timeRange: "7d" for production
- [ ] Review job limit: 10 per company

**4. Testing**
- [ ] Test Loop Companies with 3 companies
- [ ] Verify jobs saved to database
- [ ] Test Loop Jobs with sample job (pin data)
- [ ] Verify AI evaluation works
- [ ] Test Send Email with test workflow_run_id
- [ ] Confirm email received and formatted correctly

---

### Deployment

**1. Workflow Activation**
- [ ] Import all three workflow JSON files
- [ ] Connect workflows (Execute Workflow nodes)
- [ ] Verify all node credentials assigned
- [ ] Test each workflow independently
- [ ] Test full pipeline end-to-end

**2. Scheduling**
- [ ] Set up cron trigger for Loop Companies
- [ ] Recommended: Daily at 6 AM local time
- [ ] Format: `0 6 * * *` (6:00 AM daily)
- [ ] Test scheduled execution
- [ ] Verify email received

**3. Monitoring Setup**
- [ ] Set up execution alerts (email or Slack)
- [ ] Configure error notifications
- [ ] Dashboard for daily run status
- [ ] Cost tracking (Apify + Anthropic)
- [ ] Success rate monitoring

---

### Post-Deployment

**1. Daily Monitoring (First Week)**
- [ ] Check email received each morning
- [ ] Verify job count reasonable (80-150 typical)
- [ ] Review errors table for issues
- [ ] Monitor API costs
- [ ] Check execution time (should be 5-10 minutes)

**2. Weekly Review**
- [ ] Query email_queue for statistics
- [ ] Review company success rate
- [ ] Check for consistently failing companies
- [ ] Update companies table if needed
- [ ] Review AI evaluation quality

**3. Monthly Maintenance**
- [ ] Clean up old email_queue entries (>30 days)
- [ ] Archive historical job data
- [ ] Review and update target companies
- [ ] Check API quota usage
- [ ] Update candidate profile if needed
- [ ] Review cost efficiency

---

### Validation Queries

**After Each Run:**
```sql
-- 1. Verify jobs discovered
SELECT COUNT(*) as jobs_found,
       COUNT(DISTINCT company_id) as companies_with_jobs
FROM jobs
WHERE date_posted >= CURRENT_DATE;

-- 2. Check email queue populated
SELECT COUNT(*) as jobs_in_queue,
       MIN(overall_score) as min_score,
       MAX(overall_score) as max_score,
       AVG(overall_score) as avg_score
FROM email_queue
WHERE workflow_run_id = (SELECT MAX(workflow_run_id) FROM email_queue);

-- 3. Review errors
SELECT status, COUNT(*) as count
FROM errors
WHERE verified_at >= CURRENT_DATE
GROUP BY status;

-- 4. Top recommendations
SELECT recommendation, COUNT(*) as count
FROM email_queue
WHERE workflow_run_id = (SELECT MAX(workflow_run_id) FROM email_queue)
GROUP BY recommendation
ORDER BY 
  CASE recommendation
    WHEN 'EXCELLENT_MATCH' THEN 1
    WHEN 'GOOD_MATCH' THEN 2
    WHEN 'CONSIDER' THEN 3
    WHEN 'POOR_FIT' THEN 4
    ELSE 5
  END;
```

---

### Rollback Plan

**If Production Issues Occur:**

1. **Pause Scheduling**
   - Deactivate cron trigger
   - Prevent additional executions

2. **Assess Impact**
   - Check last successful run
   - Identify failed component
   - Review error logs

3. **Quick Fixes**
   - Re-authenticate API credentials
   - Restart n8n service
   - Clear database locks

4. **Rollback Steps**
   - Restore previous workflow version
   - Revert database to last backup
   - Test with single company
   - Resume scheduling when stable

5. **Manual Email (Emergency)**
   - Query email_queue manually
   - Copy HTML cards to email client
   - Send manually to maintain cadence

---

## ❓ FAQ

### General Questions

**Q: How long does a full workflow run take?**  
A: Approximately 7-8 minutes for 40 companies with average 3 jobs each. Breakdown:
- Company processing: ~2 seconds per company (Apify API calls)
- Job evaluation: ~3 seconds per job (AI evaluation)
- Email generation: ~2 seconds
- Total: 40×2s + 120×3s + 2s ≈ 442 seconds ≈ 7.4 minutes

**Q: What happens if I add more companies?**  
A: Linear scaling:
- 60 companies ≈ 10-12 minutes
- 80 companies ≈ 13-16 minutes
- 100 companies ≈ 16-20 minutes

Cost also scales linearly with job count.

**Q: Can I run the workflow multiple times per day?**  
A: Yes, but recommendations:
- Morning: Full scan with timeRange "7d"
- Evening: Quick check with timeRange "24h"
- Adjust email subject to indicate run type

**Q: Why three separate workflows instead of one?**  
A: Several benefits:
1. **Independent testing**: Test email formatting without running full scan
2. **Reusability**: Loop Jobs could be triggered from other sources
3. **Clearer debugging**: Isolate issues to specific workflow
4. **Separation of concerns**: Discovery, evaluation, delivery are distinct
5. **Easier maintenance**: Modify one without affecting others

---

### Technical Questions

**Q: What if Apify API is down?**  
A: Workflow handles this gracefully:
- All companies will return api_error status
- Errors logged to errors table
- Workflow completes without crashing
- Re-run when Apify is back up

**Q: Can I modify the AI evaluation criteria?**  
A: Yes, in Loop Jobs workflow:
1. Update candidate profile in Add Resume Context node
2. Modify scoring criteria in Message a model prompt
3. Adjust threshold in email filtering (optional)

**Q: How do I add a new company to monitor?**  
A: Simple:
1. Add row to companies table:
   - company_id: "newco"
   - company_name: "NewCo Inc"
   - domain: "newco.com"
2. No workflow changes needed
3. Next run will automatically process new company

**Q: Can I filter email to only show excellent matches?**  
A: Yes, modify Load Email Queue query:
```sql
WHERE workflow_run_id = X 
AND recommendation = 'EXCELLENT_MATCH'
```
Or filter in Aggregate and Sort Jobs node.

**Q: What if I want weekly digests instead of daily?**  
A: Two approaches:
1. **Schedule weekly**: Change cron to weekly, increase timeRange to "7d"
2. **Accumulate**: Run daily, modify email query to include last 7 days of workflow_run_ids

**Q: Can I send to multiple recipients?**  
A: Yes:
- Gmail node: Comma-separated in sendTo field
- Or: Add loop in Send Email to send individually
- Or: Use BCC for privacy

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
- jobs: Raw job listings
- email_queue: Formatted cards
- errors: Monitoring logs
- No data sent to third parties except:
  - Apify: Company domains for search
  - Anthropic: Job descriptions for evaluation (no personal data)
  - Gmail: Recipient email for delivery

**Q: Is my candidate profile secure?**  
A: Yes:
- Stored only in workflow code (not database)
- Sent to Anthropic AI (encrypted in transit)
- Not stored by Anthropic (per their privacy policy)
- Not included in database tables

**Q: Can I delete historical data?**  
A: Yes, safe to delete:
```sql
-- Delete old email queue (>30 days)
DELETE FROM email_queue 
WHERE processed_at < NOW() - INTERVAL '30 days';

-- Delete old jobs (>90 days)
DELETE FROM jobs 
WHERE date_posted < NOW() - INTERVAL '90 days';
```

**Q: What data is sent to AI?**  
A: Only what's needed for evaluation:
- Your resume/profile (for matching)
- Job description (from Apify)
- Job metadata (title, company, salary, location)
- No personal identifiers
- No application data

---

### Customization Questions

**Q: Can I add more job sources beyond Apify?**  
A: Yes:
1. Add new API node in Loop Companies
2. Format to match expected schema
3. Merge with Apify results
4. Rest of pipeline handles automatically

**Q: Can I modify the email design?**  
A: Yes, in Build Email node:
- Update HTML template
- Change colors/styling
- Add/remove sections
- Test by viewing node output

**Q: Can I add different scoring dimensions?**  
A: Yes, in Loop Jobs:
1. Update AI prompt with new dimensions
2. Update Parse AI Response to extract them
3. Update Format Job Card to display them

**Q: Can I save emails to database instead of sending?**  
A: Yes:
- Replace Send Jobs Email node with Insert node
- Save html_body to emails table
- Can send later or view in database

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

**Q: Can I speed up execution?**  
A: Some options:
1. Reduce job limit per company (10 → 5)
2. Reduce timeRange (7d → 3d, fewer jobs)
3. Parallel company processing (loses fault tolerance)
4. Use faster AI model (Haiku vs Sonnet)

Trade-offs: Speed vs completeness, cost vs quality

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

**API Issues:**
- Apify: Check [status.apify.com](https://status.apify.com)
- Anthropic: Check [status.anthropic.com](https://status.anthropic.com)
- Gmail: Check [Google Workspace Status](https://www.google.com/appsstatus)

**Database Issues:**
1. Verify table structure
2. Check indexes exist
3. Review query performance
4. Check disk space

---

## 📝 Version History

**v3.0** (Current) - Three-Workflow Architecture
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

**v1.0** - Domain Validation
- Single workflow
- Company domain verification
- Basic Apify integration

---

**End of Documentation**

*Last Updated: December 29, 2025*
*Workflow Version: 3.0*
*Documentation Version: 1.0*
