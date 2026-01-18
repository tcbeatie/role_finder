# RoleFinder

**An intelligent, AI-powered job monitoring system that discovers, evaluates, and delivers personalized job opportunities via daily digest.**

## Who This Project Is For

This repository is intended for **developers and technical users** who want to run and self-host their own job-tracking solution using RoleFinder.

**If you'd rather not manage the technical setup and ongoing maintenance**, see the [Managed Service Option](#managed-service-option) below for a fully hosted solution.

---

RoleFinder is a production-grade automation system built on n8n that monitors target companies for relevant job postings, evaluates each position using AI against your specific criteria, and delivers a professional daily email digest sorted by match quality. Designed by a senior technical product manager for use by anyone, it surfaces high-signal opportunities with minimal noise through intelligent filtering and multi-dimensional scoring.

---

## Key Features

- 🎯 **Targeted Discovery**: Monitors companies via Apify Career Site API
- 🤖 **AI-Powered Evaluation**: AI scores jobs across 5 dimensions
- 📧 **Professional Digests**: Daily HTML email with best matches first
- 🔄 **Fault Tolerant**: Company-by-company processing survives individual failures
- 💾 **Complete Tracking**: Raw job data, AI evaluations, and email history preserved
- 📊 **Cost Efficient**: ~$1.84/day with >10,000% ROI on time saved
- 🏗️ **Production Ready**: Comprehensive documentation, error handling, monitoring queries

---

## Managed Service Option

**Don't want to self-host?** A fully managed RoleFinder service is available where all infrastructure, setup, and maintenance is handled for you.

### What's Included

- **Zero-Touch Setup**: No n8n installation, API configuration, or workflow imports
- **Infrastructure Management**: Hosting, monitoring, and uptime guaranteed
- **API Credential Handling**: Apify, Claude, and Gmail integrations managed
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

- **Email**: [TBD - will be added before publication]
- **Website**: [TBD - will be added before publication]
- **Response Time**: Within 24-48 hours

---

## Non-Goals

RoleFinder is purpose-built for a specific use case. To maintain focus and manage expectations, here's what it deliberately does **not** do:

- **Not a general-purpose job scraper** - Monitors specific target companies via Apify API, not every job board or career site
- **Not real-time alerting** - Runs on scheduled intervals (daily), not instant push notifications when jobs are posted
- **Not guaranteed to catch every posting** - Depends on Apify's data coverage, company career page updates, and API timing
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
│  MAIN v3.1 (Orchestrator)                                   │
│  Purpose: Load profile and orchestrate complete pipeline    │
│  Input:   Manual/scheduled trigger                          │
│  Actions: 1. Load profile from candidate_profile table      │
│           2. Call Loop Companies v3.1 (with profile)        │
│           3. Call Send Email v2.1 (after companies done)    │
└────┬────────────────────────────────────────────────────┬───┘
     │ Passes profile ↓                                   │
┌─────────────────────────────────────────────────────────────┐
│  WORKFLOW 1: Loop Companies v3.1                            │
│  Purpose: Discover jobs from target companies               │
│  Input:   Profile data from Main + companies table          │
│  Output:  Raw jobs → database, enriched jobs → Loop Jobs    │
│  Key:     Merges profile with each company before loop      │
└────────────────┬────────────────────────────────────────────┘
                 │ Calls for each company ↓
┌─────────────────────────────────────────────────────────────┐
│  WORKFLOW 2: Loop Jobs v4.1                                 │
│  Purpose: AI evaluation and HTML card generation            │
│  Input:   Jobs with _context_* fields (incl. profile data)  │
│  Output:  Formatted job cards → email_queue table           │
│  Key:     Uses _context_resume_text from parent (no PII)    │
└─────────────────────────────────────────────────────────────┘
     │ When all companies complete, Main calls ↓          │
┌─────────────────────────────────────────────────────────────┐
│  WORKFLOW 3: Send Email v2.1                                │
│  Purpose: Aggregate and deliver daily digest                │
│  Input:   workflow_run_id from Main                         │
│  Output:  Professional HTML email via Gmail                 │
└─────────────────────────────────────────────────────────────┘
```

### Profile Externalization Architecture

**Key Innovation**: Candidate profiles are stored in the `candidate_profile` database table, not hardcoded in workflows. This enables:

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

### Workflow Details

**Main v3.1 (5 nodes)** - Top-level orchestrator
- Entry point via manual trigger or cron schedule
- Loads candidate profile from `candidate_profile` database table
- Calls Loop Companies v3.1 sub-workflow (passes profile data)
- Waits for all companies to complete processing
- Calls Send Email v2.1 sub-workflow (passes workflow_run_id)
- Completes when email is delivered

**Loop Companies v3.1 (14 nodes)** - Job discovery
- Receives profile data from Main workflow
- Loads target companies from `companies` database table
- Merges profile with each company (1-to-many relationship)
- Iterates one company at a time (fault tolerance)
- For each company:
  - Calls Apify API to discover jobs
  - Saves raw job data immediately to `jobs` table (backup path)
  - Enriches jobs with `_context_*` fields including profile data
  - Calls Loop Jobs v4.1 sub-workflow with enriched job data
- Handles API errors and no-results gracefully
- Returns summary to Main when all companies complete

**Loop Jobs v4.1 (8 nodes)** - AI evaluation
- Receives jobs with `_context_*` fields from Loop Companies
- Uses `_context_resume_text` from parent (no hardcoded profile)
- Iterates one job at a time (AI rate limiting + error isolation)
- For each job:
  - Sends job + profile to Claude Sonnet 4.5 for evaluation
  - Parses structured JSON response (scores, reasoning, recommendation)
  - Merges AI evaluation with complete job data (parallel paths)
  - Generates professional HTML job card
  - Saves to `email_queue` table with `workflow_run_id`
- Returns to Loop Companies when all jobs complete

**Send Email v2.1 (5 nodes)** - Email delivery
- Receives `workflow_run_id` from Main workflow
- Queries `email_queue` for all jobs from this workflow run
- Aggregates and sorts by AI score (best first)
- Groups by recommendation category (excellent/good/consider)
- Generates dynamic subject line based on quality
- Builds professional HTML email with statistics
- Sends via Gmail API with delivery confirmation

---

## Technology Stack

- **Orchestration**: n8n (workflow automation platform)
- **Job Discovery**: Apify Career Site Job Listing API
- **AI Evaluation**: Anthropic Claude Sonnet 4.5
- **Email Delivery**: Gmail API (OAuth2)
- **Data Storage**: n8n Data Tables (or PostgreSQL for production)
- **Deployment**: n8n-hosted, then later Docker-based (n8n + database)

---

## Data Flow & Traceability

Every job is traceable through the entire pipeline via workflow_run_id:

1. **Discovery**: Loop Companies discovers job
2. **Context**: Enriched with `_context_workflow_run_id`, `_context_company_id`
3. **Evaluation**: Loop Jobs evaluates with AI, saves to email_queue
4. **Delivery**: Send Email queries by workflow_run_id, includes in digest
5. **Tracking**: Email footer shows workflow_run_id for reference

Database tables:
- `candidate_profile` - User profiles with resume and target criteria (enables multi-user support)
- `companies` - Target companies to monitor
- `jobs` - Raw job data with searchable fields
- `email_queue` - Formatted job cards linked by workflow_run_id
- `errors` - API failures and no-results tracking

---

## AI Evaluation Criteria

RoleFinder uses AI to score each job across five customizable dimensions. Below is an **example configuration** used for a Senior Technical Product Manager search:

1. **Seniority Match** (Example: Senior/Lead/Principal/Director level, not Junior/Associate)
2. **Domain Match** (Example: Infrastructure, Platform Engineering, Developer Experience, API platforms, Cloud systems)
3. **Technical Depth** (Example: Backend systems, integrations, developer tools expertise)
4. **Location Fit** (Example: Remote, Hybrid NYC Metro, or Hybrid Bay Area)
5. **Compensation** (Example: $190k+ minimum threshold)

**Note**: These criteria are fully customizable in the `candidate_profile` database table. Adapt them to match your specific job search requirements and priorities.

Overall recommendation categories:
- **EXCELLENT_MATCH**: 8-10 score, strong fit across dimensions
- **GOOD_MATCH**: 6-8 score, solid candidate with minor gaps
- **CONSIDER**: 4-6 score, worth reviewing despite gaps
- **POOR_FIT**: 1-4 score, significant mismatches

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
- Gmail API: $0.00 (free tier)
- n8n hosting: ~$1.00/day (varies by provider)
- **Total: ~$1.84/day** (~$55/month, ~$671/year)

**Note**: Your actual costs will vary based on the number of companies monitored and their job posting frequency.

**ROI:**
- Time saved: 2-3 hours/day of manual searching
- Your hourly value: $100-150/hour (based on target comp)
- Daily value: $200-450 in time saved
- **ROI: >10,000%**

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

- **Operator_Guide.md** - Comprehensive system guide
  - Architecture deep-dive
  - All four workflows explained
  - Database schema
  - Profile externalization pattern
  - Cost analysis
  - Troubleshooting guide
  - Production checklist
  - 30+ FAQ entries

- **Main.json** - Main orchestrator v3.1 (5 nodes)
- **Loop_Companies.json** - Workflow 1 v3.1 (14 nodes)
- **Loop_Jobs.json** - Workflow 2 v4.1 (8 nodes)
- **Send_Email.json** - Workflow 3 v2.1 (5 nodes)

Each workflow JSON includes comprehensive inline comments suitable for junior developer handoff.

---

## Quick Start

### Prerequisites
- n8n instance (cloud or self-hosted)
- Apify account with Career Site API access
- Anthropic API key (Claude access)
- Gmail account with API enabled

### Setup Steps

1. **Import Workflows**
   ```bash
   # Import all four workflow JSON files into n8n
   # Main.json (v3.1)
   # Loop_Companies.json (v3.1)
   # Loop_Jobs.json (v4.1)
   # Send_Email.json (v2.1)
   ```

2. **Configure Credentials**
   - Apify OAuth2: Connect account
   - Anthropic API: Add API key
   - Gmail OAuth2: Connect account

3. **Setup Database Tables**
   ```sql
   -- Create required tables
   CREATE TABLE candidate_profile (
     profile_id VARCHAR(50) PRIMARY KEY,
     profile_name VARCHAR(100),
     resume_text TEXT,
     target_criteria TEXT,
     notes TEXT
   );

   CREATE TABLE companies (
     company_id VARCHAR(50) PRIMARY KEY,
     company_name VARCHAR(100),
     domain VARCHAR(100)
   );

   CREATE TABLE jobs (...);  -- See documentation
   CREATE TABLE email_queue (...);
   CREATE TABLE errors (...);

   -- Add indexes
   CREATE INDEX idx_email_queue_workflow ON email_queue(workflow_run_id);
   ```

4. **Populate Candidate Profile**
   ```sql
   -- Add your profile data
   INSERT INTO candidate_profile VALUES (
     'default',           -- profile_id
     'Your Name',         -- profile_name
     'Your complete resume/experience text here...',  -- resume_text
     '{"titleSearch": ["product manager"], "titleExclusionSearch": ["product marketing"]}',  -- target_criteria
     'Optional notes'     -- notes
   );
   ```

5. **Populate Companies**
   ```sql
   -- Add target companies
   INSERT INTO companies VALUES
     ('anthropic', 'Anthropic', 'anthropic.com'),
     ('openai', 'OpenAI', 'openai.com'),
     -- ... add 40 companies
   ```

6. **Update Configuration**
   - Set recipient email in Send Email v2.1 workflow
   - Verify profile_id filter in Main v3.1 (defaults to 'default')
   - Adjust scoring criteria in Loop Jobs v4.1 prompt if needed

7. **Test Execution**
   ```bash
   # Test with 3 companies first
   # 1. Run Main v3.1 manually
   # 2. Verify jobs in database
   # 3. Check email received
   # 4. Review formatting
   ```

8. **Schedule Daily Run**
   - Add cron trigger to Main v3.1: `0 6 * * *` (6 AM daily)
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
- Verify Gmail API credentials
- Check Sent folder (may not deliver to self)

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

See Operator_Guide.md for detailed troubleshooting guide.

---

## Extending RoleFinder

### Adding More Companies
Simply add rows to companies table - no workflow changes needed.

### Updating Your Profile
Update the `candidate_profile` table with new resume or criteria - no workflow changes needed.
```sql
UPDATE candidate_profile
SET resume_text = 'Updated resume text...',
    target_criteria = '{"titleSearch": ["..."], ...}'
WHERE profile_id = 'default';
```

### Supporting Multiple Users
Add multiple profiles to `candidate_profile` table, modify Main v3.1 to select different profile_id.

### Customizing AI Criteria
Update scoring criteria in Loop Jobs v4.1 AI prompt or modify `target_criteria` in database.

### Different Email Formats
Modify HTML template in Send Email v2.1 Build Email node - test with pin data.

### Multiple Recipients
Add comma-separated emails or loop in Send Email v2.1 workflow.

### Alternative Delivery
Replace Gmail node with Slack, Discord, or database save.

### Weekly Digests
Change cron schedule in Main v3.1 and modify email query to include last 7 days.

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
- Email delivery: >99% (Gmail API)

---

## Project Status

**Current State:**
- ✅ Production-ready four-workflow architecture with orchestrator
- ✅ Profile externalization (multi-user ready)
- ✅ Comprehensive documentation (116KB)
- ✅ Fault-tolerant design with error handling
- ✅ Cost-efficient ($1.84/day)
- ✅ Complete traceability and monitoring
- ✅ Clean workflow files (no PII in JSON)

**Future Enhancements:**
- [ ] Web dashboard for historical analysis
- [ ] Mobile notifications for excellent matches
- [ ] ML-powered company prioritization
- [ ] Multi-user support with personalized profiles
- [ ] Integration with ATS APIs directly (bypass Apify)
- [ ] Real-time job alerts (webhook-based)

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
3. Modify scoring criteria in Loop Jobs v4.1 to match your targets
4. Populate `companies` table with your target list
5. Configure your own API credentials (Apify, Anthropic, Gmail)

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
- [Gmail API](https://developers.google.com/gmail/api) - Email delivery

Inspired by the need for intelligent, low-noise job discovery that respects your time and expertise.

---

## About the Creator

RoleFinder was created by **Ted Beatie**, a Senior Technical Product Manager with [X years] of experience in [domain/industry - e.g., developer tools, infrastructure, platform engineering].

### Why I Built This

After years of relying on traditional job boards, I grew frustrated with the noise-to-signal ratio. LinkedIn, Indeed, and other platforms surface hundreds of irrelevant roles while missing opportunities at companies I actually wanted to work for. I needed a system that:

- **Monitored specific companies** where I wanted to work, not every company on the internet
- **Used AI to evaluate fit** based on my actual experience and criteria, not keyword matching
- **Delivered high-signal alerts** instead of overwhelming me with noise
- **Saved hours of manual searching** every single day

RoleFinder is that system. I built it for myself, and now I use it daily to monitor [40/X] target companies and receive personalized job digests every morning.

### From Personal Tool to Open Source

What started as a personal automation project evolved into a production-grade system with:
- Clean architecture separating discovery, evaluation, and delivery
- Externalized profile management for multi-user support
- Comprehensive error handling and fault tolerance
- 116KB of documentation suitable for junior developer handoff

I'm open-sourcing RoleFinder because I believe others face the same frustrations with traditional job search. Whether you're a senior engineer, product manager, designer, or any other role - if you know *where* you want to work but struggle to track opportunities, this system can help.

### Managed Service

For those who want the benefits without the technical setup, I now offer a [fully managed service](#managed-service-option) where I handle all infrastructure, monitoring, and maintenance.

---

## Contact

### For Self-Hosted Users

For questions about architecture, implementation, or technical issues:
- **GitHub Issues**: Open an issue in this repository for bugs or feature requests
- **Documentation**: Review Operator_Guide.md for comprehensive technical details
- **Support**: Best-effort community support via GitHub

### For Managed Service Inquiries

Interested in having RoleFinder fully managed for you?
- **Email**: [TBD - will be added before publication]
- **Website**: [TBD - will be added before publication]
- **Response Time**: Within 24-48 hours
- **Pricing**: Contact for personalized quote based on your needs

---

**RoleFinder** - Intelligent role monitoring for active job-seekers.

*Last Updated: January 17, 2026*
