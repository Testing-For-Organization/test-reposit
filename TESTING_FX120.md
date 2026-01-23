# FX-120 Testing Guide: Rescan After PR Merge

## Overview
This guide helps you test the FX-120 feature: **Rescan after user merged the PR into their repo**

## Prerequisites
1. Database connection configured (check `.envrc` file)
2. Dash server running (or access to database)
3. At least one Felix-created PR in your repository

## Step-by-Step Testing

### Step 1: Check Database Connection

```bash
# Make sure your database environment variables are set
cd dash
source .envrc  # or export variables manually

# Test connection (if you have psql installed)
psql -h $DB_ENDPOINT -p $DB_PORT -U $DB_USERNAME -d $DB_NAME
```

### Step 2: Find Your Repository and PRs

**Option A: Using SQL (Recommended)**

```bash
# Connect to database
psql -h $DB_ENDPOINT -p $DB_PORT -U $DB_USERNAME -d $DB_NAME

# Then run queries from test_pr_merge.sql
```

**Option B: Using Helper Script**

```bash
cd /home/syedali/Desktop/Alchemain-felix-dash
go run test_pr_helper.go check "https://github.com/your-account/your-repo/pull/123"
```

### Step 3: Verify PR Exists in Database

Run this SQL query (replace with your PR URL):

```sql
SELECT 
    id,
    title,
    status,
    external_reference AS pr_url,
    repository_id
FROM felix.pull_requests
WHERE external_reference = 'https://github.com/your-account/your-repo/pull/123';
```

**Expected Result:**
- ✅ If PR found: You'll see the PR details → **Ready to test!**
- ❌ If PR not found: This PR wasn't created by Felix → **Cannot test with this PR**

### Step 4: Test the Merge Flow

#### 4.1: Check Current PR Status

```sql
SELECT status, external_reference 
FROM felix.pull_requests 
WHERE external_reference = 'your-pr-url';
```

#### 4.2: Merge the PR on GitHub

1. Go to GitHub
2. Open the Felix-created PR
3. Click "Merge pull request"
4. Confirm the merge

#### 4.3: Monitor Logs

Watch your dash server logs. You should see:

**Success Logs:**
```
"handling pull request event" action=closed number=123
"pr merged, checking if felix-created" pr_url=... repository_id=...
"felix pr merged, triggering rescan" pr_url=... repository_id=... pr_id=...
"rescan triggered successfully after pr merge" repository_id=... pr_url=...
```

**If PR not found:**
```
"pr merged, checking if felix-created" pr_url=...
"pr not found in database, skipping rescan (not a felix-created pr)" pr_url=...
```

### Step 5: Verify Rescan Job Was Created

Check if the `scan_and_plan` job was published:

```sql
-- Check job queue (if you have access)
SELECT * FROM felix.job_queue 
WHERE action = 'scan_and_plan' 
ORDER BY created_at DESC 
LIMIT 5;
```

Or check your job processing system (AWS Batch, etc.)

## Troubleshooting

### Problem: PR not found in database

**Solution:**
- Make sure the PR was created by Felix (check PR author)
- PRs created before the `UpdateJobAndPR` implementation might not be in DB
- Create a new PR using Felix upgrade command

### Problem: No logs appearing

**Check:**
1. Is dash server running?
2. Are webhooks configured correctly?
3. Is the webhook endpoint accessible?
4. Check webhook delivery in GitHub Settings → Webhooks

### Problem: Rescan not triggered

**Check:**
1. Verify PR status in database is being updated
2. Check if `GetByExternalReference` is finding the PR
3. Verify application lookup is working
4. Check if job publisher is working

## Quick Test Checklist

- [ ] Database connection working
- [ ] Found at least one Felix-created PR in database
- [ ] PR is in "open" or "Open" status
- [ ] Dash server is running
- [ ] Webhooks are configured
- [ ] Merged the PR on GitHub
- [ ] Checked logs for success messages
- [ ] Verified rescan job was created

## SQL Queries Reference

All useful queries are in `test_pr_merge.sql` file.

## Need Help?

If you're stuck:
1. Check the logs first
2. Verify database connection
3. Make sure PR exists in database
4. Check webhook configuration

