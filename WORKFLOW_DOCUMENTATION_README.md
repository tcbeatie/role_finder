# Validate Apify Data v3 - Complete Documentation

## 📋 Table of Contents
1. [Overview](#overview)
2. [Workflow Architecture](#workflow-architecture)
3. [Node-by-Node Guide](#node-by-node-guide)
4. [Key Concepts](#key-concepts)
5. [Status Taxonomy](#status-taxonomy)
6. [Troubleshooting](#troubleshooting)
7. [Production Checklist](#production-checklist)

---

## 🎯 Overview

### Purpose
This workflow validates company domain guesses against actual job listing data from Apify's Career Site Job Listing API.

### Business Problem
We have a list of companies with domain guesses (e.g., "simspace.com"). We need to verify if these guesses are correct by comparing them against real-world job postings.

### Solution
1. Search Apify for each company's job listings
2. Extract the actual domain from those jobs
3. Compare against our guess
4. Save validation results + raw job data

### Outputs
**Two database tables:**
1. **`domain_verification`** - Validation summary (one row per company)
2. **`apify_jobs_raw`** - Raw job data (one row per job, with extracted fields)

---

## 🏗️ Workflow Architecture

### High-Level Flow
```
Manual Trigger
  ↓
Load Companies from Table
  ↓
Build Apify Requests
  ↓
LOOP (one company at a time) ──┐
  ↓                              |
  Run Apify API                  |
    ↓ Success        ↓ Error     |
    ├─ IF           Log Error   |
    |  ↓ True  ↓ False   ↓       |
    |  Valid  No Res  Insert bad |
    |  ↓        ↓         ↓      |
    |  Convert Insert     |      |
    |  ↓        bad       |      |
    |  Insert  row        |      |
    |  good    ↓          |      |
    |  row     └──────────┴──────┘
    |  ↓
    └─ Save Apify Data
       ↓
       Insert Raw Jobs
```

### Parallel Processing
From **Run Apify Success branch**, two paths run **simultaneously**:
- **Path 1 (Main)**: Validation → Database
- **Path 2 (Side)**: Save raw jobs → Database

Both complete before looping back.

---

## 📝 Node-by-Node Guide

### NODE 1: When clicking 'Execute workflow'
**Type:** Manual Trigger  
**Purpose:** Start the workflow  

**What it does:**
- Entry point - click to run
- No configuration needed
- Produces no data itself

---

### NODE 2: Load Data from Table
**Type:** Data Table (Get)  
**Purpose:** Load all companies to process  

**Configuration:**
- Table: `companies to verify` (or your test table)
- Operation: Get all rows
- returnAll: `true`

**Input Schema:**
```
company_id: string (e.g., "simspace")
company_name: string (e.g., "SimSpace")
domain_guess: string (e.g., "simspace.com")
ats_type: string (e.g., "ashby")
```

**Output:** Array of company objects

---

### NODE 3: Build Request & Store Context
**Type:** Code (JavaScript)  
**Purpose:** Format data for Apify + preserve context  

**Key Variables:**
- `items` - All companies from previous node
- `company` - Current company being processed in .map()

**Important Concept: _context_ fields**
```javascript
_context_company_id: company.company_id
```
These custom fields:
- Are ignored by Apify (unrecognized)
- Pass through to the response
- Let us match results back to companies
- Like putting a tracking label on each package

**Output:** Array of Apify-formatted requests

---

### NODE 4: Loop Over Items
**Type:** Split In Batches  
**Purpose:** Process companies ONE at a time  

**Settings:**
- Batch Size: 1 (processes one company per iteration)
- Mode: Each item separately

**Why Loop?**
1. **Fault Tolerance:** One failure doesn't crash entire workflow
2. **Incremental Saving:** Results saved after each company
3. **Progress Tracking:** See real-time progress
4. **Cost Control:** Know exactly how many API calls succeeded

**Two Outputs:**
- **Loop branch** (bottom pin): Fires for EACH company
- **Done branch** (top pin): Fires ONCE when all done

**Accessing Current Company:**
```javascript
const currentCompany = $('Loop Over Items').item.json;
```

---

### NODE 5: Run Apify
**Type:** Apify Integration  
**Purpose:** Call Apify API to get job listings  

**Critical Settings:**
- `alwaysOutputData: true` - Output even if 0 jobs
- `onError: continueErrorOutput` - Don't crash on errors
- `customBody: ={{ $json }}` - Send all fields

**Two Outputs:**
1. **Success** (top pin): API succeeded (0+ jobs)
2. **Error** (bottom pin): API failed

**Scenarios:**
- SimSpace: Returns 8 jobs → Success with 8 items
- Fake company: Returns 0 jobs → Success with 0 items
- API timeout: → Error with error object

---

### NODE 6: IF (Check for Real Job Data)
**Type:** IF Node  
**Purpose:** Distinguish real jobs from empty results  

**Condition:**
```javascript
{{ !!($json && ($json.id || $json.title || $json.organization)) }}
```

**Breakdown:**
- `$json` - Current item
- `$json.id || $json.title || $json.organization` - Check for job fields
- `!!` - Convert to boolean (true/false)

**Why `!!`?**
- Without it: Returns actual value ("1919889452")
- With it: Returns true/false
- IF node needs boolean

**Routing:**
- **TRUE**: Has job data → Validate domains
- **FALSE**: No job data → Log no results

---

### NODE 7: Validate domains
**Type:** Code (JavaScript)  
**Purpose:** Core validation logic - check if domain guess is correct  

**Major Variables:**
```javascript
allJobs          // All jobs for current company
currentCompany   // Current company from loop
domainGuess      // Domain we're trying to validate
domainsSet       // Set of unique domains found (auto-deduplicates)
atsSet           // Set of unique ATS platforms found
result           // Final validation object
```

**Process Flow:**

**STEP 1: Get Jobs and Company**
```javascript
const allJobs = $input.all().map(item => item.json);
const currentCompany = $('Loop Over Items').item.json;
const domainGuess = currentCompany._context_domain_guess;
```

**STEP 2: Initialize Result Object**
```javascript
let result = {
  company_id: currentCompany._context_company_id,
  company_name: currentCompany._context_company_name,
  domain_guess: domainGuess,
  jobs_found: allJobs.length,
  status: 'unknown',  // Will become: guess_correct, guess_wrong, or not_found
  domain_verified: null,
  domains_seen: [],
  ats_platforms_seen: []
};
```

**STEP 3: Check if Jobs Exist**
```javascript
if (allJobs.length === 0) {
  // NO JOBS FOUND
  result.status = 'not_found';
  result.note = 'No jobs found for this company';
}
```

**STEP 4: Extract Domains (if jobs exist)**
```javascript
const domainsSet = new Set();  // Auto-deduplication
const atsSet = new Set();

allJobs.forEach(job => {
  // Extract domain from domain_derived field
  if (job.domain_derived) {
    domainsSet.add(job.domain_derived);
  }
  
  // FALLBACK: Extract from organization_url
  if (job.organization_url) {
    try {
      const url = new URL(job.organization_url);
      domainsSet.add(url.hostname.replace('www.', ''));
    } catch (e) {
      // Invalid URL, skip
    }
  }
  
  // Extract ATS platform
  if (job.source) {
    atsSet.add(job.source);
  }
});
```

**Why Two Domain Sources?**
- `domain_derived` - Primary (most reliable)
- `organization_url` - Fallback (when domain_derived is null)

**STEP 5: Validate**
```javascript
if (result.domains_seen.includes(domainGuess)) {
  // ✅ MATCH
  result.status = 'guess_correct';
} else {
  // ❌ MISMATCH
  result.status = 'guess_wrong';
  result.note = `Expected "${domainGuess}", found "${result.domain_verified}"`;
}
```

**Output:** Single validation result object

---

### NODE 8: Convert arrays to strings
**Type:** Code (JavaScript)  
**Purpose:** Transform arrays to database-friendly strings  

**Why?**
Database text fields can't store arrays directly.

**Transformation:**
```javascript
// Before: domains_seen = ["simspace.com", "jobs.simspace.com"]
// After:  domains_seen = "simspace.com, jobs.simspace.com"

domains_seen: item.domains_seen.join(', '),
ats_platforms_seen: item.ats_platforms_seen.join(', ')
```

---

### NODE 9: Log no results
**Type:** Code (JavaScript)  
**Purpose:** Format "no jobs found" cases for database  

**When it fires:** IF node False branch (0 jobs returned)

**Creates record:**
```javascript
{
  company_id: currentCompany._context_company_id,
  status: 'no_results',
  jobs_found: 0,
  note: 'API succeeded but returned 0 jobs - company not found in Apify or name mismatch'
}
```

---

### NODE 10: Log Error
**Type:** Code (JavaScript)  
**Purpose:** Format API errors for database  

**When it fires:** Run Apify Error branch (API failed)

**Creates record:**
```javascript
{
  company_id: currentCompany._context_company_id,
  status: 'api_error',
  jobs_found: 0,
  note: `Apify API Error: ${errorInfo.message || JSON.stringify(errorInfo)}`
}
```

---

### NODE 11: Save Apify data
**Type:** Code (JavaScript)  
**Purpose:** Extract key fields from jobs for searchability  

**Major Variables:**
```javascript
currentJobs    // All jobs from Run Apify
currentCompany // Current company from loop
```

**What it does:**
Creates ONE ROW PER JOB with extracted fields:

**Company Context:**
- company_id, company_name, domain_guess

**Job Identifiers:**
- job_id, job_title, organization, organization_url

**Dates:**
- date_posted, date_created

**Location:**
- location_locality (city)
- location_region (state)
- location_country
- remote_derived (boolean)

**Salary:**
- salary_currency, salary_min, salary_max, salary_unit

**Job Details:**
- employment_type, experience_level, work_arrangement

**Source:**
- source_ats (e.g., "greenhouse")
- source_domain (ATS domain)
- domain_derived (company domain)

**Backup:**
- job_data_full (complete JSON as string)

**Why These Fields?**
- Searchable (can query by title, salary, location)
- CSV-friendly (no size limits)
- Full backup available (job_data_full)

**Key Extraction Example:**
```javascript
// Extract first location from locations_raw array
location_locality: job.locations_raw?.[0]?.address?.addressLocality || ''
```

**The `?.` operator:**
- Optional chaining
- Prevents errors if field doesn't exist
- Example: `job.locations_raw?.[0]` 
  - If locations_raw is null → returns undefined (no error)
  - If locations_raw exists → accesses [0]

---

### NODE 12: Insert good row
**Type:** Data Table (Insert)  
**Purpose:** Save successful validations to domain_verification table  

**Receives data from:**
- Convert arrays to strings (guess_correct, guess_wrong)

**Field Mappings:**
```
company_id → ={{ $json.company_id }}
status → ={{ $json.status }}
jobs_found → ={{ $json.jobs_found }}
... (11 fields total)
```

**The `={{ }}` syntax:**
- n8n expression language
- `$json.field` accesses field from input data
- `now` generates current timestamp

---

### NODE 13: Insert bad row
**Type:** Data Table (Insert)  
**Purpose:** Save errors/no-results to domain_verification table  

**Receives data from:**
- Log no results (no_results status)
- Log Error (api_error status)

**Same table as Insert good row** - all results in one place

---

### NODE 14: Insert row (Raw Jobs)
**Type:** Data Table (Insert)  
**Purpose:** Save raw job data to apify_jobs_raw table  

**Receives data from:** Save Apify data

**One row per job** with all extracted fields

---

## 🔑 Key Concepts

### 1. Loop Isolation
**Problem:** n8n loops create execution boundaries  
**Impact:** Data inside loop can't be accessed from Done branch  
**Solution:** Use database tables to accumulate data

### 2. Context Preservation
**Problem:** Apify returns mixed results from multiple companies  
**Solution:** Add _context_ fields that pass through API unchanged  

### 3. Fault Tolerance
**Problem:** One API failure crashes entire batch  
**Solution:** Process one company at a time with error handling  

### 4. Parallel Paths
**Concept:** Run Apify has TWO outputs from Success branch  
**Benefit:** Save validation + raw data simultaneously  
**Key:** Both complete before loop continues  

### 5. Status-Based Routing
**Four possible outcomes:**
- `guess_correct` - Domain matches
- `guess_wrong` - Domain doesn't match
- `no_results` - API succeeded, 0 jobs
- `api_error` - API failed

---

## 📊 Status Taxonomy

| Status | Meaning | Path | Database |
|--------|---------|------|----------|
| `guess_correct` | Jobs found, domain matches ✅ | Success → Validate | domain_verification |
| `guess_wrong` | Jobs found, domain mismatch ⚠️ | Success → Validate | domain_verification |
| `no_results` | API succeeded, 0 jobs returned 📭 | Success → IF False → Log No Results | domain_verification |
| `api_error` | API call failed ❌ | Error → Log Error | domain_verification |

---

## 🐛 Troubleshooting

### Workflow Stops After First Company
**Cause:** Insert nodes not looping back  
**Fix:** Verify connections loop back to "Loop Over Items"

### No Jobs Saved to apify_jobs_raw
**Cause:** Save Apify data path not connected  
**Fix:** Verify Run Apify → Save Apify data connection exists

### All Companies Show api_error
**Cause:** Apify credentials invalid  
**Fix:** Check Apify API key in credentials

### Companies Show no_results But Should Have Jobs
**Cause:** Company name spelling mismatch  
**Fix:** Check exact spelling in Apify (e.g., "OpenAI" vs "Open AI")

### Database Shows Duplicate Rows
**Cause:** Workflow run multiple times  
**Fix:** Clear tables before production runs, or add unique constraints

---

## ✅ Production Checklist

**Before Running:**
- [ ] Update input table to full company list (not test data)
- [ ] Verify Apify credentials are valid
- [ ] Clear output tables (domain_verification, apify_jobs_raw)
- [ ] Test with 2-3 companies first

**During Run:**
- [ ] Monitor loop progress (watch node executions)
- [ ] Check for error branches activating
- [ ] Verify database rows appearing incrementally

**After Run:**
- [ ] Query domain_verification for status breakdown
- [ ] Check for any api_error statuses
- [ ] Verify apify_jobs_raw has expected job count
- [ ] Export results to CSV

**Validation Queries:**
```sql
-- Status breakdown
SELECT status, COUNT(*) as count 
FROM domain_verification 
GROUP BY status;

-- Companies with wrong guesses
SELECT company_name, domain_guess, domain_verified, note
FROM domain_verification
WHERE status = 'guess_wrong';

-- Companies with no jobs
SELECT company_name, note
FROM domain_verification
WHERE status IN ('no_results', 'api_error');
```

---

## 📚 Additional Resources

**n8n Documentation:**
- Loop nodes: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.splitinbatches/
- Code node: https://docs.n8n.io/code-examples/
- Error handling: https://docs.n8n.io/workflows/error-handling/

**Apify API:**
- Career Site Job Listing API: https://apify.com/fantastic-jobs/career-site-job-listing-api
- API Documentation: https://docs.apify.com/api/v2

---

## 🙋 FAQ

**Q: Why one company at a time? Isn't that slower?**  
A: Fault tolerance is more important than speed. If company #30 fails, companies #31-62 still process.

**Q: Can I increase the job limit from 10 to 100?**  
A: Yes, change `limit: 10` in "Build Request & Store Context" node. Be aware of cost increase.

**Q: Why save raw jobs if we have validation results?**  
A: Validation tells us IF domains match. Raw jobs let us analyze WHAT jobs exist (salary, location, etc.)

**Q: Can I run this multiple times?**  
A: Yes, but clear tables first or you'll get duplicates.

**Q: What if Apify is down?**  
A: All companies will show `api_error` status. Re-run when Apify is back up.

---

**End of Documentation**
