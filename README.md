# RoleFinder

**An intelligent, AI-powered job monitoring system that discovers, evaluates, and delivers personalized job opportunities via daily digest.**

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

## Architecture Overview

RoleFinder uses a three-workflow pipeline that separates concerns for clean architecture:

```
┌─────────────────────────────────────────────────────────────┐
│  WORKFLOW 1: Loop Companies                                 │
│  Purpose: Discover jobs from target companies               │
│  Input:   companies table (40 companies)                    │
│  Output:  Raw jobs → database, enriched jobs → Loop Jobs    │
└────────────────┬────────────────────────────────────────────┘
                 │ Calls for each company ↓
┌─────────────────────────────────────────────────────────────┐
│  WORKFLOW 2: Loop Jobs                                      │
│  Purpose: AI evaluation and HTML card generation            │
│  Input:   Jobs with context from parent workflow            │
│  Output:  Formatted job cards → email_queue table           │
└────────────────┬────────────────────────────────────────────┘
                 │ When all complete, Main calls ↓
┌─────────────────────────────────────────────────────────────┐
│  WORKFLOW 3: Send Email                                     │
│  Purpose: Aggregate and deliver daily digest                │
│  Input:   workflow_run_id from parent                       │
│  Output:  Professional HTML email via Gmail                 │
└─────────────────────────────────────────────────────────────┘
```

### Workflow Details

**Loop Companies (14 nodes)**
- Loads target companies from database
- Iterates one company at a time (fault tolerance)
- Calls Apify API to discover jobs
- Saves raw job data immediately (backup path)
- Calls Loop Jobs sub-workflow with enriched job data
- Handles API errors and no-results gracefully
- Returns to company loop for next iteration

**Loop Jobs (9 nodes)**
- Receives jobs from parent with context fields
- Iterates one job at a time (AI rate limiting)
- Adds candidate profile and target criteria
- Calls Claude Sonnet 4.5 for intelligent evaluation
- Parses structured JSON response (scores, reasoning)
- Merges AI evaluation with complete job data
- Generates professional HTML job card
- Saves to email queue with workflow tracking
- Returns to parent when all jobs complete

**Send Email (5 nodes)**
- Queries email queue for all jobs from workflow run
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
- `companies` - Target companies to monitor
- `jobs` - Raw job data with searchable fields
- `email_queue` - Formatted job cards linked by workflow_run_id
- `errors` - API failures and no-results tracking

---

## AI Evaluation Criteria

Each job is scored (1-10) across five dimensions:

1. **Seniority Match**: Senior/Lead/Principal/Director level (not Junior/Associate)
2. **Domain Match**: Infrastructure, Platform Engineering, Developer Experience, API platforms, Cloud systems
3. **Technical Depth**: Backend systems, integrations, developer tools expertise
4. **Location Fit**: Remote, Hybrid NYC Metro, or Hybrid Bay Area
5. **Compensation**: $190k+ minimum threshold

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

## Cost Analysis

**Daily Breakdown:**
- Apify API: $0.48 (120 jobs × $0.004)
- Claude AI: $0.36 (120 jobs × $0.003)
- Gmail API: $0.00 (included)
- n8n hosting: $1.00 (amortized)
- **Total: $1.84/day** ($55/month, $671/year)

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

- **WORKFLOW_DOCUMENTATION_README.md** - Comprehensive system guide (116KB)
  - Architecture deep-dive
  - All three workflows explained
  - Database schema
  - Cost analysis
  - Troubleshooting guide
  - Production checklist
  - 30+ FAQ entries

- **Loop_Companies.json** - Workflow 1 (14 nodes)
- **Loop_Jobs.json** - Workflow 2 (9 nodes)
- **Send_Email.json** - Workflow 3 (5 nodes)

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
   # Import all three workflow JSON files into n8n
   # Loop_Companies_FULLY_DOCUMENTED.json
   # Loop_Jobs_FULLY_DOCUMENTED.json
   # Send_Email_FULLY_DOCUMENTED.json
   ```

2. **Configure Credentials**
   - Apify OAuth2: Connect account
   - Anthropic API: Add API key
   - Gmail OAuth2: Connect account

3. **Setup Database Tables**
   ```sql
   -- Create required tables
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

4. **Populate Companies**
   ```sql
   -- Add target companies
   INSERT INTO companies VALUES 
     ('anthropic', 'Anthropic', 'anthropic.com'),
     ('openai', 'OpenAI', 'openai.com'),
     -- ... add 40 companies
   ```

5. **Update Configuration**
   - Set recipient email in Send Email workflow
   - Update candidate profile in Loop Jobs workflow
   - Adjust scoring criteria if needed

6. **Test Execution**
   ```bash
   # Test with 3 companies first
   # 1. Run Loop Companies manually
   # 2. Verify jobs in database
   # 3. Check email received
   # 4. Review formatting
   ```

7. **Schedule Daily Run**
   - Add cron trigger to Loop Companies: `0 6 * * *` (6 AM daily)
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

See WORKFLOW_DOCUMENTATION_README.md for detailed troubleshooting guide.

---

## Extending RoleFinder

### Adding More Companies
Simply add rows to companies table - no workflow changes needed.

### Customizing AI Criteria
Update candidate profile and scoring criteria in Loop Jobs workflow.

### Different Email Formats
Modify HTML template in Build Email node - test with pin data.

### Multiple Recipients
Add comma-separated emails or loop in Send Email workflow.

### Alternative Delivery
Replace Gmail node with Slack, Discord, or database save.

### Weekly Digests
Change cron schedule and modify email query to include last 7 days.

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
- ✅ Production-ready three-workflow architecture
- ✅ Comprehensive documentation (116KB)
- ✅ Fault-tolerant design with error handling
- ✅ Cost-efficient ($1.84/day)
- ✅ Complete traceability and monitoring

**Future Enhancements:**
- [ ] Web dashboard for historical analysis
- [ ] Mobile notifications for excellent matches
- [ ] ML-powered company prioritization
- [ ] Multi-user support with personalized profiles
- [ ] Integration with ATS APIs directly (bypass Apify)
- [ ] Real-time job alerts (webhook-based)

---

## Contributing

This is currently a personal project configured for my specific job search needs. However, the architecture is designed to be generalizable.

If you're interested in adapting RoleFinder for your own use:
1. Fork the repository
2. Update the candidate profile in Loop Jobs
3. Modify scoring criteria to match your targets
4. Populate companies table with your target list
5. Configure your own API credentials

Pull requests for bug fixes or architectural improvements are welcome.

---

## License

This project is currently unlicensed and for personal use. A proper open-source license may be added in the future.

---

## Acknowledgments

Built with:
- [n8n](https://n8n.io/) - Workflow automation platform
- [Apify](https://apify.com/) - Career site job listing API
- [Anthropic Claude](https://www.anthropic.com/) - AI evaluation engine
- [Gmail API](https://developers.google.com/gmail/api) - Email delivery

Inspired by the need for intelligent, low-noise job discovery that respects your time and expertise.

---

## Contact

For questions about architecture or implementation:
- Open an issue in this repository
- Review the comprehensive documentation in WORKFLOW_DOCUMENTATION_README.md

---

**RoleFinder** - Intelligent job monitoring for senior technical product managers.

*Last Updated: December 29, 2025*
