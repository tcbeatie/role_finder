# RoleFinder TODO List

## 🚀 High Priority (Production Readiness)

### ~~1. Schedule Daily Runs~~ ✅ DONE
Main v6.1 has a `Schedule Trigger` node wired to `Load Profiles`, configured to fire at hour 7 daily. Both `Start` (manual) and `Schedule Trigger` feed the same pipeline.

---

### 2. Validate API Credits & Quotas
- [ ] **Check Anthropic API quota**
  - Log into Anthropic console
  - Review current usage and limits
  - Calculate daily usage: 120 jobs × $0.003 = $0.36/day
  - Estimate monthly: $0.36 × 30 = $10.80/month
  - Verify credit card on file and billing alerts set
  - Set up quota alerts (e.g., warning at 80% usage)

- [ ] **Check Apify API quota**
  - Log into Apify console
  - Review current usage and limits
  - Calculate daily usage: 120 jobs × $0.004 = $0.48/day
  - Estimate monthly: $0.48 × 30 = $14.40/month
  - Verify payment method
  - Set up usage alerts

- [ ] **Document quota monitoring process**
  - Add to weekly maintenance checklist
  - Create dashboard or script to check usage
  - Set up alerts for unexpected spikes

**Why**: Prevent service disruption from quota exhaustion
**Estimated effort**: 1 hour

---

### 2a. Fix Match Category Count Bugs (Issue #42)
Three correctness bugs in the run_reports pipeline discovered post-v0.6.0 merge:

- [ ] **STRONG_MATCH dead branch** — Loop Jobs emits `STRONG_MATCH` as a recommendation but no downstream path handles it; those jobs are silently dropped from category counts
- [ ] **REJECT uncounted** — `POOR_FIT`/`REJECT` category is evaluated but not tallied in Send Email's `poor_count` return value
- [ ] **Hardcoded zeros in Save Run Report** — Build Run Report node writes literal `0` for some count fields instead of reading from Send Email's structured response

Fix approach:
  - Audit Loop Jobs recommendation categories against Send Email aggregation logic
  - Confirm which label (`STRONG_MATCH` vs `GOOD_MATCH`) the AI actually returns and align node routing
  - Verify Send Email `Success Response` returns non-zero values for all five count fields
  - Confirm Build Run Report reads each count from `$('Call Send Email').item.json.*` (not hardcoded)
  - Test with a real run and query `run_reports` to verify all columns populate correctly

**Why**: run_reports data is wrong today — category counts undercount matches and make historical analysis unreliable
**Estimated effort**: 2-3 hours
**Tracking**: GitHub issue #42

---

### ~~3. Externalize Resume/Profile Data~~ ✅ DONE (v0.4.1)
Profile data moved from hardcoded workflow nodes to `candidate_profile` database table. Loaded once at Main startup, passed to sub-workflows via `_context_*` fields. No personal data in workflow JSON files.

- [ ] **Create profile management workflow** (optional future enhancement)
  - CRUD operations for profiles
  - Version history tracking

---

### ~~3a. Externalize Company Data~~ ✅ DONE (v0.4.1)
Companies moved to `companies` database table with profile_id foreign key. Main v6.1 loads and filters companies per profile.

---

### 3b. Externalize Remaining Apify Filters
`titleSearch` and `titleExclusionSearch` externalized to `target_criteria.apify_filters` in v0.5.0. Three filters still hardcoded in Loop Companies v5.1:

- [ ] **Externalize `timeRange`** — move from Build Request node to `apify_filters.timeRange`
- [ ] **Externalize `locationSearch`** — move from Build Request node to `apify_filters.locationSearch`
- [ ] **Externalize `aiWorkArrangementFilter`** — move from Build Request node to `apify_filters.aiWorkArrangementFilter`

All three should read from `_context_target_criteria.apify_filters` exactly as `titleSearch` does. No schema changes needed — the JSON keys already exist in the documented `target_criteria` structure.

**Why**: Per-user location/arrangement/timeframe filters without workflow edits; required for proper multi-user support
**Estimated effort**: 1-2 hours
**Tracking**: Deferred from v0.5.0 → v0.6.0 → now targeting v0.7.0

---

### ~~3c. Fail nicely when zero jobs~~ ✅ DONE (v0.5.1 / v0.4.1)
Send Email v4.1 implements dual-path execution: empty queue → Skip path returns structured `{status: "skipped", email_sent: false, total_jobs: 0, ...}` response; Main loop continues to next profile without hanging.

---


## 💰 Cost Optimization

### 4. Batch Apify Requests
Currently: 40 separate API calls → Optimize to single or fewer calls

- [ ] **Research Apify batching capabilities**
  - Review Career Site API documentation
  - Check if accepts array of domains
  - Verify output format with batched requests
  - Test with 2-3 companies first

- [ ] **Implement batching in Loop Companies**
  - **Option A**: Single request with all domains
    - Modify "Build Apify Requests" to combine domains
    - Update "Run Apify" to handle batch response
    - Split results back to individual companies
    - Pros: 1 API call instead of 40 (massive savings)
    - Cons: All-or-nothing (one failure breaks all)
  
  - **Option B**: Batch in groups of 10
    - Group companies into batches of 10
    - Process each batch as single Apify request
    - Still get 4x reduction in API calls
    - Better fault tolerance than single batch
    - Pros: Balance of cost savings and reliability
    - Cons: More complex logic
  
- [ ] **Update error handling**
  - Handle partial failures in batches
  - Log which companies in batch failed
  - Retry individual companies if needed

- [ ] **Test and measure**
  - Compare costs: 40 calls vs batched
  - Verify job discovery accuracy unchanged
  - Monitor for rate limiting issues
  - Document cost savings

**Why**: Reduce Apify costs by 75-90% ($14/month → $2-4/month)
**Estimated effort**: 4 hours
**Potential savings**: $10/month

---

### 5. Self-Host n8n + Database
Currently: n8n Cloud ($30/month) → Move to self-hosted ($5-10/month VPS)

- [ ] **Set up infrastructure**
  - Provision VPS (DigitalOcean, Linode, or AWS Lightsail)
  - Recommended: 2GB RAM, 50GB storage, $10-12/month
  - Install Docker and Docker Compose
  - Set up SSL certificate (Let's Encrypt)

- [ ] **Deploy n8n stack**
  - Use official n8n Docker Compose configuration
  - Include PostgreSQL container for database
  - Configure persistent volumes for data
  - Set up environment variables and secrets
  - Reference: https://docs.n8n.io/hosting/installation/docker/

- [ ] **Migrate workflows**
  - Export workflows from n8n Cloud (JSON)
  - Import into self-hosted instance
  - Reconfigure credentials (not exported)
  - Test all four workflows end-to-end (Main, Loop Companies, Loop Jobs, Send Email)

- [ ] **Migrate database**
  - Export data tables from n8n Cloud
  - Import into self-hosted PostgreSQL
  - Verify data integrity
  - Update workflow connections to new database

- [ ] **Configure monitoring**
  - Set up uptime monitoring (UptimeRobot, Healthchecks.io)
  - Configure email alerts for downtime
  - Set up log aggregation (optional)
  - Create backup script for PostgreSQL

- [ ] **Update DNS and access**
  - Point domain to VPS (if using custom domain)
  - Configure firewall (allow only necessary ports)
  - Set up SSH key authentication
  - Disable password authentication

**Why**: Reduce hosting costs by 60-70% ($30/month → $10-12/month)
**Estimated effort**: 6-8 hours (one-time setup)
**Monthly savings**: $18-20

**Alternative**: Keep n8n Cloud, only self-host database
- Lighter migration (just database)
- Still save ~$15/month on data storage
- Keep n8n convenience and updates

---

## 🔧 Operational Improvements

### 5a. Create `run_reports` Data Table
New table added in v0.6.0 — must be created in n8n before Main v6.1 can persist run history:

- [ ] **Create `run_reports` table** using schema at `tables/template_run_reports.csv`
  - Columns: `main_execution_id`, `profile_id`, `full_name`, `status`, `email_sent`, `total_jobs`, `excellent_count`, `strong_count`, `consider_count`, `poor_count`, `workflow_run_id`, `message_id`, `run_duration_ms`, `completed_at`
  - One row per profile per daily run
  - `main_execution_id` links rows from the same orchestrator execution

**Why**: Main v6.1 Save Run Report node will error without this table
**Estimated effort**: 15 minutes

---

### 6. Database Optimization
- [ ] **Create indexes for performance**
  ```sql
  -- Email queue fast lookups
  CREATE INDEX idx_email_queue_workflow ON email_queue(workflow_run_id);
  CREATE INDEX idx_email_queue_score ON email_queue(overall_score DESC);
  CREATE INDEX idx_email_queue_recommendation ON email_queue(recommendation);
  
  -- Jobs table queries
  CREATE INDEX idx_jobs_company ON jobs(company_id);
  CREATE INDEX idx_jobs_posted ON jobs(date_posted DESC);
  CREATE INDEX idx_jobs_title ON jobs(job_title);
  
  -- Errors tracking
  CREATE INDEX idx_errors_status ON errors(status);
  CREATE INDEX idx_errors_verified ON errors(verified_at DESC);
  ```

- [ ] **Add database constraints**
  ```sql
  -- Prevent duplicate jobs in same run
  ALTER TABLE email_queue ADD CONSTRAINT unique_workflow_job 
    UNIQUE(workflow_run_id, job_id);
  
  -- Ensure workflow_run_id is never null
  ALTER TABLE email_queue ALTER COLUMN workflow_run_id SET NOT NULL;
  ```

- [ ] **Implement data retention policy**
  - Email queue: Keep 30 days, archive older
  - Jobs table: Keep 90 days, archive to cold storage
  - Errors: Keep 60 days
  - Create automated cleanup scripts/workflows

**Why**: Improve query performance, prevent duplicates, manage storage
**Estimated effort**: 1 hour

---

### 7. Monitoring & Alerting
- [ ] **Set up execution monitoring**
  - Create n8n workflow for monitoring
  - Check: Main v6.1 ran in last 24 hours (query `run_reports` table for recent rows)
  - Check: Email was sent successfully (run_reports.email_sent = true)
  - Alert if workflow fails or doesn't run
  - Send to email or Slack

- [ ] **Create health check endpoint**
  - Simple webhook that returns workflow status
  - Include: last run time, jobs found, errors count
  - Use with external monitoring (UptimeRobot)

- [ ] **Cost tracking dashboard**
  - Query Anthropic API for token usage
  - Query Apify API for request count
  - Calculate daily/weekly/monthly costs
  - Alert if costs spike unexpectedly

- [ ] **Quality metrics tracking**
  - Average jobs per company
  - AI score distribution
  - Excellent matches per week
  - Companies with 0 results (need review)

**Why**: Proactive issue detection, cost control, quality assurance
**Estimated effort**: 3 hours

---

### 8. Error Handling Enhancements
- [ ] **Add retry logic to AI evaluation**
  - Currently: Single attempt per job
  - Improve: 3 retries with exponential backoff
  - Handle rate limiting gracefully
  - Log retry attempts to database

- [ ] **Implement graceful degradation**
  - If AI fails for job, save to queue anyway with placeholder
  - Mark as "needs_evaluation"
  - Create manual review workflow
  - Retry failed jobs in batch later

- [ ] **Better error notifications**
  - Send email summary of errors
  - Include: companies failed, jobs missed, costs saved
  - Actionable next steps in email

**Why**: Improve reliability, reduce manual intervention
**Estimated effort**: 2 hours

---

## 📧 Email Improvements

### 9. Email Template Enhancements
- [ ] **Add filtering controls**
  - Include link to "View only Excellent matches"
  - Generate multiple versions (full vs filtered)
  - User preference for default view

- [ ] **Improve mobile responsiveness**
  - Test in mobile email clients
  - Ensure cards render well on small screens
  - Optimize for Gmail mobile app

- [ ] **Add email analytics**
  - Track open rates (if possible)
  - Track link clicks to job postings
  - Include tracking pixel (optional)
  - Generate weekly "you clicked X jobs" report

- [ ] **Plain text fallback**
  - Generate plain text version of email
  - For email clients that don't support HTML
  - Improves deliverability

**Why**: Better user experience, insights into engagement
**Estimated effort**: 3 hours

---

### 10. Email Delivery Options
- [ ] **Support multiple recipients**
  - Allow comma-separated email list
  - Or: Load from database table
  - Each gets personalized email (multi-profile support)

- [ ] **Alternative delivery channels**
  - Slack integration (post to private channel)
  - Discord webhook
  - SMS for excellent matches only (Twilio)
  - Push notifications (future)

- [ ] **Digest frequency options**
  - Daily (current)
  - Weekly summary (top 20 jobs from week)
  - Instant alerts (excellent matches only)
  - Configurable per user

**Why**: Flexibility, multi-user support, better notifications
**Estimated effort**: 4 hours

---

## 🧪 Testing & Validation

### 11. Comprehensive Testing
- [ ] **Test with full 40 company list**
  - Currently tested with 3-5 companies
  - Run full production list
  - Verify execution completes (7-8 minutes expected)
  - Check costs align with estimates
  - Review email quality with 100+ jobs

- [ ] **Test edge cases**
  - Company with 0 jobs (no_results)
  - Company with API error
  - Company with 100+ jobs (over limit)
  - Job with missing salary
  - Job with missing location
  - Job with very long description (>10k chars)

- [ ] **Load testing**
  - Test with 100 companies (if scaling)
  - Measure execution time scaling
  - Check for memory leaks
  - Verify database performance under load

- [ ] **Email rendering tests**
  - Test in Gmail (web, mobile)
  - Test in Outlook (web, desktop)
  - Test in Apple Mail (macOS, iOS)
  - Test in other clients (Yahoo, ProtonMail)

**Why**: Confidence in production readiness
**Estimated effort**: 3 hours

---

### 12. Documentation Updates
- [ ] **Document actual company list**
  - Create `companies.csv` with 40 companies
  - Include: company_id, company_name, domain, ats_type
  - Add to repository (or keep private)
  - Document how to populate companies table

- [ ] **Create video walkthrough** (optional)
  - Screen recording of full workflow execution
  - Show: trigger → processing → email received
  - Explain architecture visually
  - Post to YouTube or Loom

- [ ] **Add troubleshooting scenarios**
  - Based on actual issues encountered
  - Screenshots of error messages
  - Step-by-step resolution guides

- [ ] **Create runbook for daily operations**
  - What to do if email not received
  - How to check workflow status
  - How to manually trigger run
  - Who to contact for API issues

**Why**: Knowledge preservation, easier onboarding
**Estimated effort**: 2 hours

---

## 🚀 Feature Enhancements (Future)

### 13. Historical Analysis Dashboard
- [ ] **Build web dashboard**
  - Technology: React + Next.js or Streamlit
  - Connect to database (read-only)
  - Visualizations:
    - Jobs found over time (line chart)
    - AI score distribution (histogram)
    - Top companies by job count (bar chart)
    - Salary trends (box plot)
    - Location breakdown (pie chart)

- [ ] **Add filtering and search**
  - Filter by date range
  - Filter by company
  - Filter by recommendation
  - Search by job title or keywords

**Why**: Better insights, track market trends
**Estimated effort**: 8-12 hours

---

### 14. AI Improvements
- [ ] **Fine-tune evaluation prompt**
  - A/B test different prompt phrasings
  - Optimize for consistent scoring
  - Reduce false positives/negatives
  - Compare Claude Sonnet vs Haiku for cost/quality

- [ ] **Add learning from user feedback**
  - Track which jobs user applies to
  - Use as training signal for AI
  - Gradually improve scoring accuracy
  - Implement feedback loop

- [ ] **Add job description summarization**
  - Include AI-generated summary in card
  - Highlight key responsibilities
  - Extract required skills
  - Estimate time investment (e.g., "High leadership component")

**Why**: Better AI accuracy, more useful insights
**Estimated effort**: 6 hours

---

### 15. Multi-User Support
**Core infrastructure done in v0.6.0** — Main v6.1 loops over all `candidate_profile` rows; each user gets a personalized digest and a `run_reports` row. Add a user by inserting a database row.

- [x] Multiple candidate profiles with per-profile companies
- [x] Loop Over Profiles in Main v6.1 — sequential, isolated processing
- [x] Per-profile email digest delivery
- [x] Per-profile run stats in `run_reports`
- [ ] **Production hardening** — test with 3+ active profiles end-to-end (planned v0.7.0)
- [ ] **Web interface for configuration** (future)
  - Update profile without editing database directly
  - Manage company list via UI
  - View historical digests and run_reports
- [ ] **Team/shared mode** (future)
  - Multiple users tracking the same company list
  - Collaborative job board with comments

**Why**: Core multi-user workflow is live; hardening and UI are the remaining gaps
**Estimated effort**: Production hardening 2 hours; UI 20+ hours

---

### 16. Advanced Job Discovery
- [ ] **Direct ATS integration**
  - Bypass Apify, call ATS APIs directly
  - Greenhouse API
  - Lever API
  - Workday API
  - Lower costs, faster updates

- [ ] **LinkedIn integration**
  - Use LinkedIn API (if available)
  - Track jobs posted on LinkedIn
  - Richer company data

- [ ] **Real-time webhooks**
  - Set up webhooks for job changes
  - Instant notifications for excellent matches
  - No need to wait for daily run

**Why**: Better coverage, lower cost, faster alerts
**Estimated effort**: 12+ hours per integration

---

## 📊 Priority Matrix

### Do First (v0.7.0 target)
1. **Fix match category count bugs** (issue #42) — correctness bug in run_reports data
2. **Create `run_reports` table** (5a) — required for Main v6.1 Save Run Report node
3. **Externalize remaining Apify filters** (3b) — timeRange, locationSearch, aiWorkArrangementFilter
4. **Validate API credits** (2)

### Do Next (This Month)
6. Database optimization (indexes + constraints)
7. Monitoring & alerting setup
8. Multi-user production hardening (test 3+ profiles)
9. Error handling enhancements (try/catch in Code nodes, retry logic)

### Consider (Next Quarter)
10. Batch Apify requests (cost savings)
11. Self-host n8n (bigger cost savings)
12. Historical analysis dashboard (run_reports data now available!)
13. AI improvements

### Future/Maybe
14. Direct ATS integrations
15. Real-time webhooks
16. Mobile app
17. Profile management web UI

---

## 🎯 Quick Wins (< 1 hour each)

- [ ] **Create `run_reports` table** from `tables/template_run_reports.csv` schema (required for v0.6.0)
- [ ] Add unique constraint to email_queue table
- [ ] Set up uptime monitoring (UptimeRobot free tier)
- [ ] Create simple health check webhook
- [ ] Document actual company list in repository
- [ ] Test email rendering in mobile Gmail
- [ ] Add database indexes
- [ ] Set up Anthropic usage alerts
- [ ] Set up Apify usage alerts

---

## 💡 Ideas / Backlog

- **Job application tracking**: Track which jobs applied to, outcomes
- **Salary negotiation helper**: Compare offers, calculate true comp
- **Interview prep**: Auto-generate company research from job data
- **Network mapping**: Identify connections at target companies
- **Market intelligence**: Aggregate trends across all tracked jobs
- **Chrome extension**: Quick-add companies from careers pages
- **API for external consumption**: Let others query your job data
- **Job recommendation engine**: ML model predicting application likelihood
- **Automated cover letter generation**: Use AI + job description + profile

---

## 📅 Estimated Timeline

**Week 1** (5 hours)
- Schedule daily runs
- Validate API credits
- Test with full company list
- Add database indexes

**Week 2** (6 hours)
- Externalize resume/profile
- Set up basic monitoring
- Email template improvements

**Month 2** (10 hours)
- Research Apify batching
- Implement and test batching
- Monitor cost savings

**Month 3** (12 hours)
- Plan self-hosting migration
- Set up infrastructure
- Migrate workflows and data
- Monitor stability

**Total initial investment**: ~33 hours over 3 months
**Ongoing maintenance**: 1-2 hours/month

---

## ✅ Completion Criteria

**Production Ready:**
- [x] All four workflows deployed and documented (Main v6.1, Loop Companies v5.1, Loop Jobs v5.1, Send Email v4.1)
- [x] Profile and company data externalized to database
- [x] Multi-user pipeline (Loop Over Profiles in Main v6.1)
- [x] Per-profile run history via `run_reports` table (v0.6.0)
- [x] Admin summary email after all profiles complete (v0.6.0)
- [x] Daily runs scheduled — Schedule Trigger in Main v6.1 fires at hour 7
- [ ] `run_reports` table created in n8n instance (5a)
- [ ] Match category count bugs fixed (issue #42)
- [ ] Monitoring and alerts configured
- [ ] Database optimized with indexes
- [ ] Tested with full 40 company list across 2+ profiles
- [ ] Email quality validated
- [ ] Cost tracking implemented

**Optimized:**
- [x] Profile/resume externalized (v0.4.1)
- [x] Apify title filters externalized (v0.5.0)
- [ ] Remaining Apify filters externalized (timeRange, locationSearch, aiWorkArrangementFilter)
- [ ] Apify requests batched (75% cost reduction)
- [ ] Self-hosted n8n (60% hosting cost reduction)
- [ ] Historical dashboard deployed (run_reports data now available as foundation)

**Extended:**
- [x] Multi-user support — core pipeline done; UI/hardening remaining
- [ ] Direct ATS integrations
- [ ] Real-time alerts

---

## 📝 Notes

**Cost Optimization ROI:**
- Current: $55/month ($30 n8n + $10 AI + $15 Apify)
- After batching: $45/month (saves $10/month Apify)
- After self-hosting: $27/month (saves $18/month hosting)
- After both: $17/month (saves $38/month = 69% reduction)
- Time investment: ~14 hours one-time
- Break-even: ~3 months

**Priority Rationale:**
High priority items (scheduling, validation, testing) ensure core functionality works reliably. Cost optimizations deferred until system proven valuable. Feature enhancements deferred until core workflow stable.

**Risk Mitigation:**
Most high-risk changes (self-hosting, batching) are optional optimizations. Core system works as-is with acceptable costs. Can operate indefinitely at current cost structure while testing improvements.

---

*Last Updated: February 13, 2026 (post v0.6.0 merge)*
