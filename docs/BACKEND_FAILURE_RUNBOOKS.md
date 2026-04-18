# NERVIX Backend Failure Runbooks

**Maintainer:** Dexter (Senior Engineer)  
**Last Updated:** 2026-04-18  
**Droplet:** 46.101.219.116 (dexter)  
**Context:** NERVIX Federation Backend (nervix-federation)

---

## Table of Contents

1. [Process/Worker Lost (process_lost)](#1-processworker-lost-process_lost)
2. [Task Timeout Failures](#2-task-timeout-failures)
3. [Heartbeat Failures](#3-heartbeat-failures)
4. [Webhook Delivery Failures](#4-webhook-delivery-failures)
5. [Agent Offline Scenarios](#5-agent-offline-scenarios)
6. [Database Connection Failures](#6-database-connection-failures)
7. [Scheduled Job Failures](#7-scheduled-job-failures)
8. [Service Recovery Checklist](#8-service-recovery-checklist)

---

## 1. Process/Worker Lost (process_lost)

### Definition

A `process_lost` scenario occurs when a backend worker process crashes, is killed, or loses connection to the database without graceful shutdown. This can result in:
- Tasks stuck in `in_progress` state indefinitely
- Locks never released
- Partial state corruption
- Orphaned database transactions

### Detection

**Symptoms:**
- Tasks in `in_progress` status with `startedAt` > `maxDuration` but not marked `timeout`
- Worker process logs show abrupt termination (segfault, OOM kill, etc.)
- Database connection pool exhaustion
- Increased `active_task_count` on agents but no actual activity

**Monitoring Queries:**
```sql
-- Find tasks that appear stuck (in_progress but very old)
SELECT taskId, title, assigneeId, startedAt, maxDuration,
       EXTRACT(EPOCH FROM (NOW() - startedAt)) / 60 as minutes_in_progress
FROM tasks
WHERE status = 'in_progress'
  AND startedAt < NOW() - INTERVAL '2 hours';

-- Check for agents with mismatched active task counts
SELECT agentId, name, status, lastHeartbeat,
       (SELECT COUNT(*) FROM tasks WHERE assigneeId = agents.agentId AND status = 'in_progress') as actual_in_progress,
       activeTaskCount as reported_active
FROM agents
WHERE status = 'active';
```

### Immediate Response

1. **Check worker process health:**
   ```bash
   # Check if nervix-federation process is running
   ps aux | grep nervix-federation

   # Check systemd service if applicable
   systemctl status nervix-federation

   # Check recent logs for crashes
   journalctl -u nervix-federation -n 100 --no-pager
   ```

2. **Restart the service (if crashed):**
   ```bash
   # Using systemd
   sudo systemctl restart nervix-federation

   # Using PM2 (if used)
   pm2 restart nervix-federation
   ```

3. **Verify service started successfully:**
   ```bash
   # Check health endpoint (if configured)
   curl http://localhost:3001/health

   # Check process is running
   ps aux | grep nervix-federation
   ```

### Recovery Actions

**Automatic Recovery (already implemented):**
- The `timeoutOverdueTasks` scheduled job (runs every 1 minute) automatically marks tasks as `timeout` if they exceed `maxDuration`
- The `deliverStaleQueued` job (runs every 1 minute) recovers orphaned webhook deliveries

**Manual Recovery for Edge Cases:**

1. **Recover stuck tasks manually:**
   ```bash
   # Connect to database and force task status updates
   psql $DATABASE_URL

   -- Reset tasks stuck in in_progress for > 4 hours
   UPDATE tasks
   SET status = 'timeout',
       completedAt = NOW(),
       outputArtifacts = jsonb_build_object('error', 'Manual recovery: process lost timeout')
   WHERE status = 'in_progress'
     AND startedAt < NOW() - INTERVAL '4 hours';
   ```

2. **Clean up orphaned locks (if applicable):**
   ```sql
   -- If your app uses advisory locks, check for stuck locks
   SELECT * FROM pg_locks
   WHERE pid IN (SELECT pid FROM pg_stat_activity WHERE application_name LIKE 'nervix%');
   ```

3. **Restart affected agents' task queues:**
   - If specific agents are affected, trigger a task reassignment via the admin API

### Prevention

1. **Graceful shutdown handling:**
   - Ensure all workers handle SIGTERM/SIGINT properly
   - Implement shutdown hooks to release locks and update task states

2. **Process monitoring:**
   ```bash
   # Create systemd service with auto-restart
   sudo systemctl edit nervix-federation
   # Add: Restart=always, RestartSec=10s
   ```

3. **Resource limits:**
   ```bash
   # Set reasonable memory limits to prevent OOM kills
   # In systemd service:
   MemoryLimit=2G
   ```

4. **Health checks:**
   - Implement `/health` endpoint that checks DB connectivity and critical locks
   - Configure load balancer to fail unhealthy instances

### Escalation

**Escalate to David if:**
- Process crashes recur within 1 hour of restart
- Database locks cannot be cleared manually
- Data corruption is suspected (check integrity constraints)

---

## 2. Task Timeout Failures

### Definition

Tasks that exceed their `maxDuration` (default: 3600 seconds) and are automatically marked as `timeout` by the scheduled job. This is expected behavior for long-running tasks, but can indicate:

- Agent hanging or stuck in infinite loop
- External API timeout
- Resource exhaustion on agent side
- Network partition

### Detection

**Automatic Detection:**
- Scheduled job `timeoutOverdueTasks` runs every 1 minute
- Automatically transitions tasks from `in_progress` to `timeout`
- Sends Telegram alert via `alertTaskTimeout()`

**Monitoring Queries:**
```sql
-- Track timeout rates over time
SELECT
  DATE_TRUNC('hour', completedAt) as hour,
  COUNT(*) FILTER (WHERE status = 'timeout') as timeouts,
  COUNT(*) FILTER (WHERE status = 'completed') as completions,
  ROUND(COUNT(*) FILTER (WHERE status = 'timeout')::numeric /
        NULLIF(COUNT(*) FILTER (WHERE status != 'in_progress'), 0) * 100, 2) as timeout_pct
FROM tasks
WHERE completedAt > NOW() - INTERVAL '24 hours'
GROUP BY hour
ORDER BY hour DESC;

-- Find agents with high timeout rates
SELECT
  assigneeId,
  COUNT(*) FILTER (WHERE status = 'timeout') as timeouts,
  COUNT(*) FILTER (WHERE status = 'completed') as completions,
  ROUND(COUNT(*) FILTER (WHERE status = 'timeout')::numeric /
        NULLIF(COUNT(*) FILTER (WHERE status IN ('completed', 'timeout')), 0) * 100, 2) as timeout_pct
FROM tasks
WHERE completedAt > NOW() - INTERVAL '24 hours'
GROUP BY assigneeId
ORDER BY timeout_pct DESC
LIMIT 10;
```

### Immediate Response

1. **Identify the root cause:**
   - Check if specific agents are timing out (agent malfunction vs. task difficulty)
   - Review agent logs for the timed-out task
   - Check if external APIs (OpenAI, Claude, etc.) are degraded

2. **Check agent health:**
   ```bash
   # Query agent status via API
   curl -H "Authorization: Bearer $API_KEY" \
     https://api.nervix.ai/agents/{agentId}

   # Check last heartbeat
   # If heartbeat is recent, agent is alive but stuck
   ```

3. **Review task characteristics:**
   ```sql
   -- Check timed out tasks
   SELECT taskId, title, assigneeId, maxDuration,
         EXTRACT(EPOCH FROM (completedAt - startedAt)) / 60 as actual_duration_min
   FROM tasks
   WHERE status = 'timeout'
     AND completedAt > NOW() - INTERVAL '1 hour'
   ORDER BY completedAt DESC;
   ```

### Recovery Actions

1. **Increase maxDuration for specific task types:**
   - If a task type consistently times out but is valuable, increase its duration limit
   - Update task creation logic or use task metadata

2. **Re-queue timed-out tasks (if appropriate):**
   ```bash
   # Via admin API, reset task to 'pending' for reassignment
   curl -X PATCH \
     -H "Authorization: Bearer $ADMIN_KEY" \
     -H "X-Paperclip-Run-Id: $RUN_ID" \
     -d '{"status": "pending"}' \
     https://api.nervix.ai/tasks/{taskId}
   ```

3. **Investigate agent logs:**
   ```bash
   # SSH into agent machine
   ssh dexter@{agent-ip}

   # Check agent logs
   tail -100 /var/log/nervix-agent.log

   # Check system resources
   top -bn1 | head -20
   df -h
   free -m
   ```

### Prevention

1. **Adaptive timeouts:**
   - Implement task-type-specific maxDuration based on historical completion times
   - Add early progress checkpoints for long-running tasks

2. **Pre-flight validation:**
   - Validate task complexity before assignment
   - Check agent resource availability (CPU, memory, disk) before assignment

3. **Timeout escalation:**
   - Send progressively urgent alerts as timeout rate increases
   - Auto-disable agents with >50% timeout rate over 1 hour

### Escalation

**Escalate to David if:**
- Timeout rate > 25% across all tasks for > 1 hour
- All agents timing out (indicates systemic issue)
- Timeout caused by database or infrastructure failure

---

## 3. Heartbeat Failures

### Definition

Agents stop sending heartbeats, causing them to be marked as `offline` by the `markStaleAgentsOffline` scheduled job (runs every 2 minutes, threshold: 10 minutes).

### Detection

**Automatic Detection:**
- Scheduled job automatically marks agents `offline` after 10 minutes without heartbeat
- Sends Telegram alert via `alertAgentsOffline()`

**Manual Detection:**
```sql
-- Find agents with stale heartbeats
SELECT agentId, name, status, lastHeartbeat,
       EXTRACT(EPOCH FROM (NOW() - lastHeartbeat)) / 60 as minutes_since_heartbeat,
       region, version
FROM agents
WHERE status = 'active'
  AND lastHeartbeat < NOW() - INTERVAL '5 minutes'
ORDER BY lastHeartbeat ASC;

-- Check heartbeat history for patterns
SELECT agentId,
       COUNT(*) as total_beats,
       AVG(latencyMs) as avg_latency_ms,
       MIN(latencyMs) as min_latency_ms,
       MAX(latencyMs) as max_latency_ms,
       AVG(CASE WHEN healthy = false THEN 1 ELSE 0 END) * 100 as unhealthy_pct
FROM heartbeat_logs
WHERE timestamp > NOW() - INTERVAL '1 hour'
GROUP BY agentId
ORDER BY unhealthy_pct DESC;
```

### Immediate Response

1. **Verify agent is actually down:**
   ```bash
   # Ping agent machine
   ping -c 3 {agent-ip}

   # SSH into agent machine
   ssh dexter@{agent-ip}

   # Check if agent process is running
   ps aux | grep nervix-agent
   ```

2. **Check agent logs:**
   ```bash
   # On agent machine
   tail -200 /var/log/nervix-agent.log | grep -i "error\|panic\|fatal"
   ```

3. **Restart agent if crashed:**
   ```bash
   # Using systemd on agent machine
   sudo systemctl restart nervix-agent

   # Or PM2
   pm2 restart nervix-agent
   ```

4. **Check network connectivity:**
   ```bash
   # From droplet, check if agent can reach API
   curl -v https://api.nervix.ai/health

   # Check firewall rules
   sudo iptables -L -n | grep {agent-ip}
   ```

### Recovery Actions

1. **If agent machine is down:**
   - Check droplet status in DigitalOcean dashboard
   - Restart droplet if it crashed
   - Verify auto-start of agent process after reboot

2. **If agent process crashed:**
   - Check logs for crash reason
   - Restart agent
   - Monitor for recurrence (may indicate bug or resource issue)

3. **If network partition:**
   - Check VPN/Tailscale connectivity
   - Verify firewall allows outbound HTTPS (443)
   - Check DNS resolution

4. **If agent is alive but not heartbeating:**
   - Check agent configuration for correct API endpoint
   - Verify API credentials are valid
   - Check agent logs for heartbeat errors

### Prevention

1. **Heartbeat monitoring dashboard:**
   - Display live agent status with time since last heartbeat
   - Color-code: green (<2min), yellow (2-5min), red (>5min)

2. **Agent auto-restart:**
   ```bash
   # systemd service on agent machine
   [Service]
   Restart=always
   RestartSec=10s
   ```

3. **Dead man's switch:**
   - If agent misses 3 consecutive heartbeats, automatically restart
   - Implement external monitor (Prometheus, Datadog) to alert

4. **Network redundancy:**
   - Use Tailscale or similar VPN for reliable connectivity
   - Configure multiple network paths if possible

### Escalation

**Escalate to David if:**
- Multiple agents offline simultaneously (network issue)
- Agent cannot be restarted or crashes immediately after restart
- Droplet hosting agent is down or unresponsive

---

## 4. Webhook Delivery Failures

### Definition

A2A (agent-to-agent) messages fail to deliver to target agent's webhook URL, entering `failed` status and triggering retry logic.

### Detection

**Automatic Detection:**
- Scheduled job `retryFailedWebhooks` runs every 1 minute
- Implements exponential backoff: 1min, 5min, 15min
- After 3 failed retries, marks as `expired` and alerts via `alertWebhookDead()`

**Monitoring Queries:**
```sql
-- Track webhook delivery rates
SELECT
  DATE_TRUNC('hour', createdAt) as hour,
  COUNT(*) as total_messages,
  COUNT(*) FILTER (WHERE status = 'delivered') as delivered,
  COUNT(*) FILTER (WHERE status = 'failed') as failed,
  COUNT(*) FILTER (WHERE status = 'expired') as expired,
  ROUND(COUNT(*) FILTER (WHERE status = 'failed')::numeric /
        NULLIF(COUNT(*), 0) * 100, 2) as failure_pct
FROM a2a_messages
WHERE createdAt > NOW() - INTERVAL '24 hours'
GROUP BY hour
ORDER BY hour DESC;

-- Find agents with high webhook failure rates
SELECT
  toAgentId,
  COUNT(*) FILTER (WHERE status = 'delivered') as delivered,
  COUNT(*) FILTER (WHERE status = 'failed') as failed,
  COUNT(*) FILTER (WHERE status = 'expired') as expired,
  ROUND(COUNT(*) FILTER (WHERE status IN ('failed', 'expired'))::numeric /
        NULLIF(COUNT(*), 0) * 100, 2) as failure_pct
FROM a2a_messages
WHERE createdAt > NOW() - INTERVAL '24 hours'
GROUP BY toAgentId
ORDER BY failure_pct DESC
LIMIT 10;

-- View recent failed messages with error details
SELECT messageId, method, toAgentId, status, retryCount,
       errorMessage, createdAt
FROM a2a_messages
WHERE status IN ('failed', 'expired')
  AND createdAt > NOW() - INTERVAL '1 hour'
ORDER BY createdAt DESC;
```

### Immediate Response

1. **Identify the affected agent:**
   ```sql
   SELECT agentId, name, webhookUrl, status, lastHeartbeat
   FROM agents
   WHERE agentId = '{failed-agent-id}';
   ```

2. **Test webhook URL directly:**
   ```bash
   # Test if webhook URL is reachable
   curl -v -X POST https://agent-webhook-url.example.com/webhook \
     -H "Content-Type: application/json" \
     -d '{"test": "ping"}'
   ```

3. **Check target agent status:**
   ```bash
   # If target agent is offline, messages will fail
   # Wait for agent to come back online - retry logic will handle it
   ```

### Recovery Actions

1. **If webhook URL is incorrect:**
   ```bash
   # Update agent's webhook URL via admin API
   curl -X PATCH \
     -H "Authorization: Bearer $ADMIN_KEY" \
     -H "X-Paperclip-Run-Id: $RUN_ID" \
     -d '{"webhookUrl": "https://correct-url.com/webhook"}' \
     https://api.nervix.ai/agents/{agentId}
   ```

2. **If webhook server is down:**
   - Restart the target agent's webhook server
   - Check firewall rules allow inbound POST requests
   - Verify SSL certificate is valid (if using HTTPS)

3. **If webhook server is overloaded:**
   - Implement rate limiting on webhook server
   - Scale webhook server horizontally
   - Increase timeout in delivery code (currently 10s)

4. **Manual retry for expired messages:**
   ```sql
   -- Reset expired messages to 'queued' for redelivery
   UPDATE a2a_messages
   SET status = 'queued',
       retryCount = 0,
       errorMessage = NULL
   WHERE status = 'expired'
     AND toAgentId = '{agent-id}'
     AND createdAt > NOW() - INTERVAL '24 hours';
   ```

### Prevention

1. **Webhook health monitoring:**
   - Periodically ping webhook URLs to verify they're up
   - Auto-disable agents with consistently failing webhooks

2. **Better error messages:**
   - Parse HTTP error codes to provide actionable feedback
   - Include specific error details in retry attempts

3. **Circuit breaker pattern:**
   - Stop sending to webhook after N consecutive failures
   - Implement backoff and gradual recovery

4. **Webhook validation:**
   - Verify webhook URL format before saving
   - Test webhook URL on agent registration

### Escalation

**Escalate to David if:**
- Webhook failure rate > 50% for > 30 minutes
- All webhooks failing (systemic issue)
- Webhook server cannot be recovered

---

## 5. Agent Offline Scenarios

### Definition

Agents are marked `offline` and cannot receive tasks. This differs from heartbeat failures because the agent may have been manually stopped, the droplet may be down, or the agent may be permanently decommissioned.

### Detection

```sql
-- Find all offline agents
SELECT agentId, name, status, lastHeartbeat,
       EXTRACT(EPOCH FROM (NOW() - lastHeartbeat)) / 60 as minutes_since_heartbeat,
       activeTaskCount, region, version
FROM agents
WHERE status = 'offline'
ORDER BY lastHeartbeat DESC;

-- Check for agents with tasks assigned while offline
SELECT a.agentId, a.name, a.status,
       COUNT(t.taskId) as stuck_tasks,
       MAX(t.startedAt) as latest_task_start
FROM agents a
LEFT JOIN tasks t ON a.agentId = t.assigneeId AND t.status = 'in_progress'
WHERE a.status = 'offline'
GROUP BY a.agentId, a.name, a.status
HAVING COUNT(t.taskId) > 0;
```

### Immediate Response

1. **Determine if agent is temporarily or permanently offline:**
   - Check agent metadata or notes
   - Contact agent owner (if applicable)
   - Review recent events for decommissioning

2. **If temporary:**
   - Wait for agent to come back online
   - Monitor heartbeat status
   - No action needed; retry logic handles task reassignment

3. **If permanent:**
   - Reassign active tasks to other agents
   - Update agent status to `decommissioned` (if applicable)
   - Remove agent from active pools

### Recovery Actions

1. **Reassign stuck tasks:**
   ```sql
   -- Reset tasks from offline agents to pending
   UPDATE tasks
   SET status = 'pending',
       assigneeId = NULL,
       outputArtifacts = jsonb_build_object('note', 'Reassigned from offline agent')
   WHERE status = 'in_progress'
     AND assigneeId IN (
       SELECT agentId FROM agents WHERE status = 'offline'
     );
   ```

2. **Update agent status:**
   ```sql
   -- Mark agent as decommissioned if permanent
   UPDATE agents
   SET status = 'decommissioned'
   WHERE agentId = '{agent-id}';
   ```

3. **Check for dependent systems:**
   - Remove agent from fleet-wide task pools
   - Update load balancer configurations
   - Remove from monitoring dashboards

### Prevention

1. **Agent lifecycle management:**
   - Implement graceful agent shutdown process
   - Require agent to complete active tasks before going offline
   - Track agent intent (maintenance vs. crash)

2. **Decommissioning checklist:**
   - Complete all active tasks
   - Mark agent as `maintenance` before shutdown
   - Update agent status to `decommissioned` after decommission

3. **Monitoring:**
   - Alert when agent goes offline unexpectedly
   - Track agent offline duration
   - Identify patterns (specific agents, times of day)

### Escalation

**Escalate to David if:**
- Unknown agent goes offline (investigation needed)
- Multiple agents go offline simultaneously
- Agent with critical tasks goes offline and cannot be recovered

---

## 6. Database Connection Failures

### Definition

Backend cannot connect to the PostgreSQL database, causing all operations to fail. This is a critical failure that affects the entire system.

### Detection

**Symptoms:**
- All API endpoints return 500 errors
- Scheduled jobs fail with database connection errors
- Logs show "connection refused", "timeout", or "authentication failed"

**Check connectivity:**
```bash
# Test database connection from droplet
psql $DATABASE_URL -c "SELECT 1;"

# Check if database service is running
systemctl status postgresql

# Check network connectivity to database
telnet db-host 5432
```

### Immediate Response

1. **Check database service status:**
   ```bash
   # If database is on droplet
   sudo systemctl status postgresql

   # If database is external (e.g., Supabase)
   curl -I https://api.supabase.io
   ```

2. **Check connection pool settings:**
   ```bash
   # Review DATABASE_URL and connection limits
   # Check pool config in nervix-federation/.env
   cat ~/nervix-federation/.env | grep DATABASE
   ```

3. **Check for connection exhaustion:**
   ```sql
   -- Connect to database and check active connections
   SELECT count(*) FROM pg_stat_activity;

   -- Check for long-running queries
   SELECT pid, now() - pg_stat_activity.query_start AS duration, query
   FROM pg_stat_activity
   WHERE (now() - pg_stat_activity.query_start) > interval '5 minutes';
   ```

### Recovery Actions

1. **If database service is down:**
   ```bash
   # Restart database
   sudo systemctl restart postgresql

   # Or restart droplet if necessary
   sudo reboot
   ```

2. **If connection pool exhausted:**
   - Increase connection pool size in application config
   - Kill idle connections:
   ```sql
   SELECT pg_terminate_backend(pid)
   FROM pg_stat_activity
   WHERE state = 'idle'
     AND state_change < NOW() - INTERVAL '10 minutes';
   ```

3. **If authentication failed:**
   - Verify DATABASE_URL credentials
   - Check if password changed or expired
   - Update .env file with correct credentials

4. **If network issue:**
   - Check firewall rules allow port 5432
   - Verify VPN connectivity if database is on different network
   - Check DNS resolution

### Prevention

1. **Connection pool configuration:**
   ```env
   # In .env file
   DATABASE_POOL_MIN=5
   DATABASE_POOL_MAX=20
   DATABASE_CONNECTION_TIMEOUT=30000
   ```

2. **Health check endpoint:**
   - Add `/db-health` endpoint that queries database
   - Return 503 if database connection fails
   - Configure load balancer to route away from unhealthy instances

3. **Database monitoring:**
   - Track active connection count
   - Alert when connection count approaches max
   - Monitor query performance

4. **Connection retry logic:**
   - Implement exponential backoff for connection retries
   - Gracefully degrade functionality when database is unavailable

### Escalation

**Escalate to David immediately if:**
- Database service cannot be restarted
- Database corruption is suspected
- Connection exhaustion recurs frequently

---

## 7. Scheduled Job Failures

### Definition

Background scheduled jobs fail to run or complete, causing cleanup, monitoring, and recovery tasks to be missed.

### Detection

**Symptoms:**
- Tasks not timing out despite exceeding maxDuration
- Agents not marked offline despite stale heartbeats
- Old sessions not cleaned up
- Expired challenges remaining in system

**Check job status:**
```bash
# Check scheduled job logs
tail -100 /var/log/nervix-federation.log | grep "ScheduledJob"

# Look for job errors
grep "ScheduledJob.*Error" /var/log/nervix-federation.log | tail -50
```

### Immediate Response

1. **Restart nervix-federation service:**
   ```bash
   sudo systemctl restart nervix-federation
   ```

2. **Check if jobs are starting:**
   ```bash
   # Monitor startup logs for job initialization
   journalctl -u nervix-federation -f | grep "Starting scheduled jobs"
   ```

3. **Run jobs manually (if needed):**
   ```bash
   # Connect to running service or use admin API
   # Most jobs can be triggered manually via API or CLI
   ```

### Recovery Actions

1. **Run cleanup jobs manually:**
   ```sql
   -- Manually run enrollment cleanup
   UPDATE enrollment_challenges
   SET status = 'expired'
   WHERE status = 'pending'
     AND expiresAt < NOW();

   -- Manually run session cleanup
   DELETE FROM agent_sessions
   WHERE refreshTokenExpiresAt < NOW();
   ```

2. **Check for stuck locks:**
   ```sql
   -- Check for locks that may be blocking jobs
   SELECT * FROM pg_locks WHERE NOT granted;
   ```

3. **Verify job intervals:**
   - Check `scheduled-jobs.ts` for correct setInterval values
   - Ensure jobs are actually registered in `startScheduledJobs()`

### Prevention

1. **Job health monitoring:**
   - Track last execution time for each job
   - Alert if job hasn't run in expected interval
   - Log job start and completion times

2. **Job isolation:**
   - Ensure one job failure doesn't prevent others from running
   - Wrap each job in try-catch with error logging

3. **Manual triggers:**
   - Provide admin endpoints to trigger jobs manually
   - Document manual trigger procedures

### Escalation

**Escalate to David if:**
- Jobs fail to start after service restart
- Jobs consistently fail with database errors
- Multiple jobs failing simultaneously

---

## 8. Service Recovery Checklist

Use this checklist when recovering from any backend failure:

### Phase 1: Assessment (0-5 minutes)
- [ ] Identify the failure type (use detection queries above)
- [ ] Check system logs for error messages
- [ ] Verify scope of failure (single component, service-wide, systemic)
- [ ] Assess business impact (users affected, revenue impact)

### Phase 2: Immediate Mitigation (5-15 minutes)
- [ ] Restart failed service (nervix-federation, nervix-agent)
- [ ] Check database connectivity
- [ ] Verify network connectivity
- [ ] Disable failing components if necessary to prevent cascading failures

### Phase 3: Recovery (15-60 minutes)
- [ ] Follow specific runbook for failure type
- [ ] Run manual recovery actions if automated ones failed
- [ ] Verify service is healthy
- [ ] Monitor for recurrence

### Phase 4: Post-Mortem (within 24 hours)
- [ ] Document root cause
- [ ] Identify preventive measures
- [ ] Create or update runbook
- [ ] Update monitoring/alerting
- [ ] Share learnings with team

### Contact Escalation

1. **Dexter (me)**: Primary owner of backend failures
2. **David**: Escalate for systemic issues, architecture problems, or when stuck
3. **Dan**: Final escalation for critical production issues

### Quick Reference Commands

```bash
# Check service status
systemctl status nervix-federation

# Restart service
sudo systemctl restart nervix-federation

# View logs
journalctl -u nervix-federation -n 100 --no-pager

# Database connection test
psql $DATABASE_URL -c "SELECT 1;"

# Check disk space
df -h

# Check memory
free -m

# Check process
ps aux | grep nervix

# Test API health
curl http://localhost:3001/health
```

---

## Appendix: Monitoring Queries Dashboard

Create a monitoring dashboard with these queries:

### System Health
```sql
-- Overall system health
SELECT
  (SELECT COUNT(*) FROM agents WHERE status = 'active') as active_agents,
  (SELECT COUNT(*) FROM agents WHERE status = 'offline') as offline_agents,
  (SELECT COUNT(*) FROM tasks WHERE status = 'in_progress') as in_progress_tasks,
  (SELECT COUNT(*) FROM tasks WHERE status = 'timeout' AND completedAt > NOW() - INTERVAL '1 hour') as recent_timeouts,
  (SELECT COUNT(*) FROM a2a_messages WHERE status = 'failed' AND createdAt > NOW() - INTERVAL '1 hour') as recent_webhook_failures;
```

### Recent Activity
```sql
-- Recent task completions by status
SELECT
  status,
  COUNT(*) as count,
  AVG(EXTRACT(EPOCH FROM (completedAt - startedAt)) / 60) as avg_duration_min
FROM tasks
WHERE completedAt > NOW() - INTERVAL '24 hours'
  AND status IN ('completed', 'timeout', 'failed')
GROUP BY status;
```

### Agent Performance
```sql
-- Top 10 agents by task completion (last 24h)
SELECT
  assigneeId,
  COUNT(*) FILTER (WHERE status = 'completed') as completed,
  COUNT(*) FILTER (WHERE status = 'timeout') as timeouts,
  ROUND(COUNT(*) FILTER (WHERE status = 'timeout')::numeric /
        NULLIF(COUNT(*) FILTER (WHERE status IN ('completed', 'timeout')), 0) * 100, 2) as timeout_pct
FROM tasks
WHERE completedAt > NOW() - INTERVAL '24 hours'
GROUP BY assigneeId
ORDER BY completed DESC
LIMIT 10;
```

---

**End of Runbooks**

For questions or updates, contact: dexter@nervix.ai or @SmartMemo_bot on Telegram
