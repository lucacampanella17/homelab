# INC-003: Database Connection Failed - Docker Networking

- **Status:** Resolved
- **Date:** 2026-04-08
- **Severity:** Critical
- **Resolution time:** not recorded
- **Resolved by:** Luca

---

## Reported Problem

The web application shows a database connection error.
Message: `SQLSTATE[HY000] [2002] Connection refused`

## Observed Symptoms

- The web application loads correctly (NGINX and Apache work)
- The "Database Status" section shows: ERROR
- Message: "could not connect to database"
- MariaDB container: status "Up (healthy)"
- All services running, but no communication with the database

## Diagnosis

**1. Check the MariaDB status:**
```bash
docker compose ps mariadb
# Result: Up (healthy)
# Conclusion: MariaDB is running
```

**2. Test the connection from the host:**
```bash
docker exec mariadb mysql -p<DB_ROOT_PASSWORD> -e "SHOW DATABASES;"
# Result: works
# Conclusion: MariaDB accepts connections
```

**3. Test the connection from Apache:**
```bash
docker exec -it apache-app bash
ping mariadb
# Result: ping: mariadb: Name or service not known
# PROBLEM IDENTIFIED: Apache cannot resolve the DNS name "mariadb"
```

**4. Check DNS resolution:**
```bash
docker exec apache-app nslookup mariadb
# Result: server can't find mariadb: NXDOMAIN
# Conclusion: the name "mariadb" does not exist in Apache's DNS
```

**5. Inspect the Apache networks:**
```bash
docker inspect apache-app | grep -A 20 Networks
# Result:
# "Networks": {
#     "sysadmin-lab_frontend": { ... }
# }
# Apache is only on the FRONTEND network
```

**6. Inspect the MariaDB networks:**
```bash
docker inspect mariadb | grep -A 20 Networks
# Result:
# "Networks": {
#     "sysadmin-lab_backend": { ... }
# }
# MariaDB is on the BACKEND network
```

**7. List all the networks:**
```bash
docker network ls | grep sysadmin-lab
# Result:
# sysadmin-lab_frontend
# sysadmin-lab_backend
# sysadmin-lab_monitoring
```

**8. Inspect the backend network:**
```bash
docker network inspect sysadmin-lab_backend
# Connected containers: mariadb, mysqld-exporter, nginx-proxy
# Apache is NOT in this list
```

## Root Cause

**Docker network isolation problem:**

Apache was configured in `docker-compose.yml` to join only the `frontend`
network, while MariaDB is on the `backend` network.

```yaml
apache-app:
  networks:
    - frontend    # frontend only
```

**In Docker:**
- Containers on the **same network** can talk to each other through automatic DNS
- Containers on **different networks** are completely isolated
- The internal DNS only resolves names within the same network

**Result:**
- Apache tries to connect to `mariadb:3306`
- DNS cannot resolve `mariadb` (it is not on its network)
- Connection failure: "Connection refused"

## Options Evaluated

**Option A - Temporary quick fix:**
```bash
docker network connect sysadmin-lab_backend apache-app
```
- No downtime
- Not permanent (lost when the container is recreated)
- Use: emergencies in production

**Option B - Permanent solution (CHOSEN):**
Modify `docker-compose.yml`:
```yaml
apache-app:
  networks:
    - backend    # network where MariaDB lives
```
- Permanent and documented solution
- Survives container recreation
- Requires a short downtime
- Use: definitive fix

**Decision:** Option B - definitive solution, accepting a short downtime.

## Solution Applied

**Commands executed:**

```bash
# 1. Edit docker-compose.yml
nano docker-compose.yml

# 2. Modify the apache-app section:
#    BEFORE:
#    networks:
#      - frontend
#
#    AFTER:
#    networks:
#      - backend

# 3. Save (Ctrl+O, Enter, Ctrl+X)

# 4. Recreate the services
docker compose down
docker compose up -d

# 5. Wait for full startup
sleep 15
```

## Post-Fix Verification

**1. Check the Apache networks:**
```bash
docker inspect apache-app | grep -A 30 Networks
# Now shows "sysadmin-lab_backend"
```

**2. DNS test from Apache:**
```bash
docker exec apache-app ping -c 3 mariadb
# 64 bytes from mariadb (172.22.0.X): icmp_seq=1 ttl=64
```

**3. DNS resolution test:**
```bash
docker exec apache-app nslookup mariadb
# Server: 127.0.0.11
# Address: 172.22.0.X
```

**4. MySQL connection test:**
```bash
docker exec apache-app mysql -h mariadb -u root -p<DB_ROOT_PASSWORD> -e "SHOW DATABASES;"
# Shows: app_database, information_schema, mysql...
```

**5. Web application test:**
```bash
curl http://localhost | grep -i "CONNECTED"
# Connection status: CONNECTED
```

**6. Check in the browser (http://localhost):**
- "Database Status" section: green
- Host: `mariadb:3306`
- Database: `app_database`
- MariaDB version: 10.11.X

**7. Full health check:**
```bash
./scripts/monitoring/health-check.sh
# All checks passing
```

## Final Result

- Database connection restored
- Apache can communicate with MariaDB
- DNS resolves `mariadb` correctly
- Application 100% functional

## Lessons Learned

### 1. Docker networking
- **Isolation by design:** different networks mean complete isolation
- **Automatic DNS:** only works within the same network
- **Multi-network:** a container can be attached to several networks

### 2. Network troubleshooting

**Systematic process:**
1. Is the target service running? `docker compose ps`
2. Can I connect from the host? `docker exec <db> mysql...`
3. Can I connect from the client? `docker exec <app> mysql -h...`
4. Does DNS resolve? `ping`, `nslookup`
5. Are they on the same network? `docker inspect`

### 3. Essential commands
```bash
# See the networks of a container
docker inspect <container> | grep -A 20 Networks

# Inspect a network
docker network inspect <network-name>

# Connect a container to a network (temporary)
docker network connect <network> <container>

# Disconnect
docker network disconnect <network> <container>
```

### 4. Difference between services running and services communicating
- MariaDB was running
- Apache was running
- But they could not communicate
- **Container healthy is not the same as container reachable**

## Prevention

To avoid this problem in the future:

1. **Document the network architecture:**
   - Create a network diagram in README.md
   - Specify which containers belong to which networks

2. **Validation in CI/CD:**
```bash
# Validation script
#!/bin/bash
# Check that Apache is on the backend network
docker inspect apache-app | grep "sysadmin-lab_backend" || exit 1
```

3. **More complete health checks:**
   Include a connectivity test to dependencies (the password comes from an
   environment variable, never hardcoded):
```yaml
healthcheck:
  test: ["CMD", "mysql", "-h", "mariadb", "-u", "root", "-p${DB_ROOT_PASSWORD}", "-e", "SELECT 1"]
```

4. **Alerting:**
   - Alert if Apache cannot connect to the database
   - Monitor DNS errors in the logs

5. **Updated runbook:**
   - Document this case in the runbook
   - Network troubleshooting commands

## Useful Commands for Future Troubleshooting

```bash
# Diagnose Docker network problems
docker network ls
docker network inspect <network-name>
docker inspect <container> | grep -A 30 Networks

# DNS test between containers
docker exec <container> ping <other-container>
docker exec <container> nslookup <other-container>
docker exec <container> getent hosts <other-container>

# Specific connectivity test
docker exec <container> nc -zv <host> <port>
docker exec <container> telnet <host> <port>

# Check connection error logs
docker logs <container> | grep -i "connection"
docker logs <container> | grep -i "refused"
```

## Timing

- Downtime: about 30 seconds
