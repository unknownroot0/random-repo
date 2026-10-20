Updated with demonstration IPs instead of the actual server and client IPs.

# L1 Support Runbook: Enable Remote PostgreSQL Access on cPanel

**Category:** Server Administration / Database Connectivity\
**Issue:** Remote PostgreSQL connection failure\
**Server:** `demo-server`\
**PostgreSQL Server IP:** `192.0.2.10`\
**Authorized Client IP:** `198.51.100.25`\
**PostgreSQL Port:** TCP `5432`\
**Resolution:** Restart PostgreSQL to apply the configured listening address.

> The IP addresses above are reserved for documentation and examples. Replace them with the actual addresses when troubleshooting a live server.

## 1. Issue Summary

A client could not connect remotely to PostgreSQL on a cPanel server. Investigation confirmed that the client's IP was allowed in PostgreSQL authentication rules and the CSF firewall.

The PostgreSQL configuration included the public server IP, but the running service was listening only on loopback addresses.

**Root cause:** The running PostgreSQL service had not applied the configured `listen_addresses` setting. Restarting PostgreSQL resolved the issue.

## 2. Troubleshooting Procedure

### Step 1: Check PostgreSQL processes and configuration files

Run as root:

```
ps aux | grep '[p]ostgres'

find /var/lib/pgsql -name postgresql.conf -o -name pg_hba.conf
```

Example configuration directory:

```
/var/lib/pgsql/data/
```

Confirm the active paths before making changes.

### Step 2: Check the listening configuration

```
grep -nE '^[[:space:]]*listen_addresses|^[[:space:]]*port' \
/var/lib/pgsql/data/postgresql.conf
```

Example:

```
listen_addresses = '127.0.0.200, localhost, 192.0.2.10'
```

The public IP may be configured correctly in the file but not yet applied to the running service.

### Step 3: Check active listening sockets

```
ss -lntp | grep ':5432'
```

Before the fix, the example output showed only:

```
127.0.0.200:5432
127.0.0.1:5432
```

The public IP was missing, indicating that PostgreSQL was not listening on that interface.

### Step 4: Check PostgreSQL authentication rules

```
grep -n '198\.51\.100\.25' \
/var/lib/pgsql/data/pg_hba.conf
```

Example existing rule:

```
host    all    all    198.51.100.25    255.255.255.255    md5
```

This allows connections from the specified client IP using password authentication, subject to database permissions.

If the rule is missing, add an appropriately restricted rule using an authentication method supported by the installed PostgreSQL version.

### Step 5: Check CSF firewall rules

```
csf -g 198.51.100.25
```

Confirm that inbound TCP port `5432` is permitted from the authorized client IP.

If required, add this rule to `/etc/csf/csf.allow`:

```
tcp|in|d=5432|s=198.51.100.25
```

Reload CSF only after making a necessary change:

```
csf -r
```

Also check any upstream firewall or provider security group.

## 3. Resolution Procedure

When the public IP is already configured in `listen_addresses`, the authentication rule exists, and the firewall permits the client, restart PostgreSQL to apply the configuration.

```
/scripts/restartsrv_postgresql
```

Confirm that the restart completes successfully.

**Important:** PostgreSQL must be restarted to apply changes to `listen_addresses`. Editing the configuration file alone does not update the active listening sockets.

## 4. Verify the Resolution

Run:

```
ss -lntp | grep ':5432'
```

Expected example output after the fix:

```
127.0.0.200:5432
192.0.2.10:5432
127.0.0.1:5432
```

The presence of `192.0.2.10:5432` confirms that PostgreSQL is listening on the configured public interface.

## 5. Test Remote Connectivity

From the authorized client machine:

```
nc -vz 192.0.2.10 5432
```

Alternatively, test with the PostgreSQL client:

```
psql -h 192.0.2.10 \
     -p 5432 \
     -U YOUR_DB_USER \
     -d YOUR_DATABASE
```

Replace the example IP and database placeholders with the actual values.

### Common errors

| Error                          | Recommended action                             |
| ------------------------------ | ---------------------------------------------- |
| Connection succeeds            | Confirm resolution                             |
| Connection timed out           | Check routing and upstream firewalls           |
| Connection refused             | Check PostgreSQL service and listening sockets |
| No `pg_hba.conf` entry         | Review client IP and authentication rules      |
| Password authentication failed | Verify credentials                             |
| Database permission denied     | Check database privileges                      |

## 6. L1 Support Checklist

- Confirm PostgreSQL is running.
- Identify the active configuration files.
- Check `listen_addresses` and port `5432`.
- Verify listening sockets using `ss`.
- Check the client IP in `pg_hba.conf`.
- Verify CSF and upstream firewall rules.
- Back up configuration files before editing.
- Restart PostgreSQL when required.
- Confirm the public listener appears.
- Test connectivity from the authorized client.
- Record the findings and resolution.

## 7. Escalation Criteria

Escalate to L2/System Administration if:

- PostgreSQL fails to restart.
- The public IP remains absent from the listening sockets.
- PostgreSQL logs show configuration errors.
- Remote connections still time out despite verified firewall rules.
- Production database availability could be affected by further changes.

Include service status, relevant logs, configuration settings, listening socket output, and firewall findings in the escalation.

## Final Resolution

**Status:** Resolved

Restarting PostgreSQL applied the existing `listen_addresses` configuration. The service then listened on the public IP, and the existing authentication and firewall rules already allowed the authorized client.

**L1 takeaway:** If the public IP is correctly configured but missing from `ss -lntp`, verify the active configuration and restart PostgreSQL before making unnecessary changes to authentication rules or firewall settings.
