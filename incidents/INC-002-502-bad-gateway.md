# INC-002: 502 Bad Gateway - Apache Not Responding

- **Status:** Resolved
- **Date:** 2026-04-08
- **Severity:** High
- **Resolution time:** not recorded
- **Resolved by:** Luca

---

## Reported Problem

502 Bad Gateway error when accessing the web application.
Users receive an intermittent NGINX error.

## Observed Symptoms

- `curl http://localhost` returns 502 Bad Gateway (NGINX)
- Container `apache-app`: status "Up (unhealthy)"
- Apache health check: failing
- NGINX working correctly

## Diagnosis

**1. Check the Apache logs:**
```bash
docker logs apache-app
# Result: Apache cannot connect to localhost:80
```

**2. Check the processes inside the container:**
```bash
docker exec -it apache-app ps aux
# Result: only the "sleep infinity" process is running
# Apache is NOT running
```

**3. Check the listening ports:**
```bash
docker exec apache-app netstat -tuln
# Result: port 80 is NOT open
```

**4. Review the docker-compose.yml configuration:**
```bash
cat docker-compose.yml | grep -A 20 apache-app
# Found: command: sleep infinity
# This command overrides the default Apache command
```

**5. Health check status:**
```bash
docker inspect apache-app | grep -A 10 Health
# Status: unhealthy
# FailingStreak: multiple failures
```

## Root Cause

The `apache-app` service in docker-compose.yml had this configured:
```yaml
command: sleep infinity
```

This command **overrides** the default command of the `php:8.1-apache` image,
which normally runs `apache2-foreground`.

**Result:**
- The container is "Up" (the sleep process is running)
- Apache is NOT running inside the container
- Port 80 is not listening
- NGINX cannot reach the backend, so it returns 502 Bad Gateway

**Key difference learned:**
- Container "running" is not the same as a working service
- A container can be "Up" while running any process
- The health check detects that the service is not working

## Solution Applied

**Commands executed:**

```bash
# 1. Edit docker-compose.yml
nano docker-compose.yml

# 2. Remove the problematic line:
#    command: sleep infinity    <-- REMOVED

# 3. Save the file (Ctrl+O, Enter, Ctrl+X)

# 4. Recreate the Apache container
docker compose up -d apache-app

# 5. Wait for startup
sleep 10

# 6. Check the processes (should now show Apache)
docker exec apache-app ps aux
# apache2 -DFOREGROUND running
```

## Post-Fix Verification

```bash
# 1. Direct HTTP test
curl http://localhost
# Application responds correctly

# 2. Check the health check
docker compose ps apache-app
# Status: Up (healthy)

# 3. Check that port 80 is open
docker exec apache-app netstat -tuln | grep :80
# Port 80 in LISTEN

# 4. Full health check
./scripts/monitoring/health-check.sh
# Apache + PHP: RUNNING
# Web application: HTTP OK

# 5. NGINX logs (no 502 errors)
docker logs nginx-proxy | grep 502
# No recent errors
```

## Final Result

- 502 error resolved
- Apache running correctly
- Health checks passing
- Application 100% operational

## Lessons Learned

1. **Container "Up" does not mean the service is working**
   - A container can run any process
   - Always check WHICH process it is running

2. **Health checks are essential**
   - They detect when a service is not working even if the container is up
   - Always review the health check status

3. **docker-compose.yml: `command` overrides defaults**
   - The `command:` directive replaces the image CMD
   - Use it only when necessary and document why

4. **Systematic troubleshooting:**
   - Logs, then processes, then ports, then configuration
   - Each step confirms or rules out a hypothesis

5. **NGINX 502 means the backend is not responding**
   - 502 Bad Gateway points to a problem with the upstream
   - It is not an NGINX problem but a backend problem

## Prevention

To prevent this problem from happening again:

1. **Mandatory code review** for docker-compose.yml
2. **Document** any `command:` override with comments
3. **CI/CD check** that validates healthy containers after deployment
4. **Alerting** in Grafana for health check failures
5. **Runbook** with this case documented

## Useful Commands for Future Troubleshooting

```bash
# Check processes in a container
docker exec <container> ps aux

# Check listening ports
docker exec <container> netstat -tuln

# Health check status
docker inspect <container> | grep -A 10 Health

# NGINX logs for upstream errors
docker logs nginx-proxy | grep -i upstream

# Direct test to the backend (bypassing NGINX)
docker exec <backend-container> curl localhost
```
