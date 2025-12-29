# RoleRadar TODO List

## 🚀 High Priority (Production Readiness)

### 1. Schedule Daily Runs
- [ ] **Configure n8n native cron trigger**
  - Add cron trigger to Loop Companies workflow
  - Schedule: `0 6 * * *` (6:00 AM daily)
  - Test trigger fires correctly
  - Verify timezone settings (local vs UTC)
  - Monitor first 3 scheduled executions
  
- [ ] **Alternative: Implement external scheduler** (if n8n cron unreliable)
  - Set up external cron (system crontab or GitHub Actions)
  - Create webhook trigger in Loop Companies
  - Call webhook URL from external scheduler
  - Add authentication to webhook

**Why**: Core functionality - daily automated runs
**Estimated effort**: 30 minutes

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

### 3. Externalize Resume/Profile Data
Currently hardcoded in Loop Jobs workflow → Move to external storage

- [ ] **Create profile storage**
  - **Option A**: New database table `candidate_profiles`
    ```sql
    CREATE TABLE candidate_profiles (
      profile_id VARCHAR(50) PRIMARY KEY,
      profile_name VARCHAR(100),
      resume_text TEXT,
      target_criteria JSONB,
      created_at TIMESTAMP,
      updated_at TIMESTAMP
    );
    ```
  - **Option B**: External file (GitHub Gist, S3, or local file)
  - **Option C**: n8n data table with single row

- [ ] **Update Loop Jobs workflow**
  - Add "Load Profile" node at start
  - Remove hardcoded profile from "Add Resume Context"
  - Replace with: `profile_text: $('Load Profile').item.json.resume_text`
  - Test with loaded profile

- [ ] **Create profile management workflow** (optional)
  - CRUD operations for profiles
  - Version history tracking
  - Multiple profile support (future)

**Why**: Easier profile updates, version control, multi-user support
**Estimated effort**: 2 hours

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
  - Test all three workflows end-to-end

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
  - Check: Loop Companies ran in last 24 hours
  - Check: Email was sent successfully
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
- [ ] **User management system**
  - Add users table with profiles
  - Multiple candidate profiles
  - Each user gets personalized email
  - Share company list, separate evaluations

- [ ] **Web interface for configuration**
  - Update profile without editing workflow
  - Manage company list
  - Configure email preferences
  - View historical digests

- [ ] **Team/shared mode**
  - Multiple users tracking same companies
  - Shared email with all evaluations
  - Collaborative job board
  - Comment/discuss jobs in UI

**Why**: Expand beyond single user, team utility
**Estimated effort**: 20+ hours (significant project)

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

### Do First (This Week)
1. ✅ Schedule daily runs
2. ✅ Validate API credits
3. ✅ Test with full company list

### Do Next (This Month)
4. Externalize resume/profile
5. Database optimization (indexes)
6. Monitoring & alerting setup
7. Email template improvements

### Consider (Next Quarter)
8. Batch Apify requests (cost savings)
9. Self-host n8n (bigger cost savings)
10. Historical analysis dashboard
11. AI improvements

### Future/Maybe
12. Multi-user support
13. Direct ATS integrations
14. Real-time webhooks
15. Mobile app

---

## 🎯 Quick Wins (< 1 hour each)

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
- [x] All three workflows deployed and documented
- [ ] Daily runs scheduled and reliable
- [ ] Monitoring and alerts configured
- [ ] Database optimized with indexes
- [ ] Tested with full 40 company list
- [ ] Email quality validated
- [ ] Cost tracking implemented

**Optimized:**
- [ ] Resume externalized
- [ ] Apify requests batched (75% cost reduction)
- [ ] Self-hosted n8n (60% hosting cost reduction)
- [ ] Historical dashboard deployed

**Extended:**
- [ ] Multi-user support
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

*Last Updated: December 29, 2025*
