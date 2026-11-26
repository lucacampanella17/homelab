# INC-001: NGINX Down - Missing SSL Certificates

- **Status:** Resolved
- **Date:** 2026-04-08
- **Severity:** Critical
- **Resolution time:** 15 minutes
- **Resolved by:** Luca

---

## Reported Problem

Web service unavailable. Users cannot access the application.

## Observed Symptoms

- Container `nginx-proxy` in "Restarting" state (restart loop)
- `curl http://localhost` returns Connection refused
- Web application 100% unavailable

## Diagnosis

1. **Check the container status:**
```bash
docker compose ps nginx-proxy
# Result: Restarting (1)
```

2. **Review the container logs:**
```bash
docker logs nginx-proxy
# ERROR: nginx: [emerg] open() "/etc/nginx/ssl/server.crt" failed
#        (2: No such file or directory)
```

3. **Check that the SSL files exist:**
```bash
ls -la services/web/nginx/ssl/
# Result: directory empty / does not exist
```

4. **Review the NGINX configuration:**
```bash
cat services/web/nginx/conf.d/default.conf
# Found: references to SSL certificates in these lines:
# ssl_certificate /etc/nginx/ssl/server.crt;
# ssl_certificate_key /etc/nginx/ssl/server.key;
```

## Root Cause

The NGINX configuration references SSL certificates that do not exist on the
system. On startup NGINX fails because it cannot find
`/etc/nginx/ssl/server.crt` and `server.key`, which puts the container into a
restart loop.

## Options Evaluated

**Option A - Generate self-signed SSL certificates (CHOSEN)**
- Permanent solution
- Simulates a real production environment
- Allows HTTPS to be tested

**Option B - Comment out the SSL configuration**
- Temporary quick fix
- Does not solve the real problem
- HTTPS would not work

**Option C - Empty files (workaround)**
- Not functional (NGINX validates the file format)
- Bad practice

## Solution Applied

**Chosen option:** A - Self-signed certificates

**Commands executed:**

```bash
# 1. Create the certificates directory
mkdir -p services/web/nginx/ssl/

# 2. Generate self-signed SSL certificates (valid for 365 days)
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout services/web/nginx/ssl/server.key \
  -out services/web/nginx/ssl/server.crt \
  -subj "/C=ES/ST=Barcelona/L=Barcelona/O=SysLab/CN=localhost"

# 3. Verify that the files were created
ls -la services/web/nginx/ssl/
# server.crt  server.key

# 4. Restart NGINX
docker compose restart nginx-proxy

# 5. Wait and check the status
sleep 5
docker compose ps nginx-proxy
# Status: Up (healthy)
```

## Post-Fix Verification

```bash
# 1. HTTP test
curl http://localhost
# Responds correctly

# 2. HTTPS test (self-signed certificate)
curl -k https://localhost
# Works over SSL

# 3. Full health check
./scripts/monitoring/health-check.sh
# 13/15 checks passed

# 4. Check the logs (no errors)
docker logs nginx-proxy | tail -20
# No SSL errors
```

## Final Result

- Web service fully restored
- NGINX running correctly
- HTTP and HTTPS operational
- Users can access the application

## Prevention - Improvements Implemented

To prevent this problem from happening again:

1. **Document the SSL requirements:**
   - Add to README.md that SSL certificates are required
   - Include the generation script in the initial setup

2. **Automation:** create a `generate-ssl-certs.sh` script:
```bash
#!/bin/bash
if [ ! -f services/web/nginx/ssl/server.crt ]; then
  echo "Generating SSL certificates..."
  openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout services/web/nginx/ssl/server.key \
    -out services/web/nginx/ssl/server.crt \
    -subj "/C=ES/ST=Barcelona/L=Barcelona/O=SysLab/CN=localhost"
  echo "Certificates generated"
fi
```

3. **Monitoring:**
   - Add a certificate expiry alert (90 days in advance)
   - Automatic renewal script

4. **Documentation:**
   - Expiry date: 2027-04-08
   - Renewal reminder in the calendar

## Lessons Learned

1. **Always check the logs first** - the errors are clear and specific.
2. **Think in terms of production** - choose permanent solutions over quick fixes.
3. **Verify dependencies** - NGINX needs the files to exist before it starts.
4. **Document everything** - it makes future troubleshooting easier.

## Impact

- **Downtime:** about 15 minutes
- **Users affected:** all (100%)
- **Data loss:** none
- **Services impacted:** frontend only (database and monitoring kept working)

## Metrics

- Diagnosis time: 5 min
- Implementation time: 5 min
- Verification time: 5 min
- **Total: 15 minutes**
