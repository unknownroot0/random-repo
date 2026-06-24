# Knowledge Base Article: Adjusting Redis Memory Limit and Eviction Policy in CloudLinux CageFS Environments

**Article ID:** KB-2026-0624-REDIS-02
**Target Audience:** Level 2 Support / System Administrators
**Environment:** CloudLinux OS, CageFS, systemd Redis, LiteSpeed or Apache stack
**Last Updated:** June 24, 2026
**Risk Level:** Medium (service restart required)

---

## 1. Overview

This article explains how to safely adjust the **Redis memory limit** and **eviction policy** in a CloudLinux CageFS-based hosting environment.

Redis is typically deployed as a **system-wide service** (not per-user), and memory tuning is applied globally unless multiple Redis instances are explicitly configured.

These changes are commonly required when:

* Applications experience Redis `OOM command not allowed`
* Cache eviction is too aggressive or too permissive
* Memory utilization is either too high or inefficient
* Performance tuning is required for WordPress / Magento / Laravel workloads

---

## 2. Architecture Understanding (Important)

In standard hosting environments:

* Redis runs as a **single systemd service**
* Configuration is stored in:

  ```
  /etc/redis.conf
  ```
* CageFS does **NOT isolate Redis per user by default**
* All users typically share the same Redis instance unless custom multi-instance setup exists

> ⚠️ If your environment uses Redis per user, it is a custom architecture and must be documented separately.

---

## 3. Preconditions

Before making changes:

* Root or sudo access required
* Confirm Redis service name:

  ```bash
  systemctl status redis
  ```
* Identify current memory usage:

  ```bash
  redis-cli INFO memory
  ```
* Take configuration backup:

  ```bash
  cp /etc/redis.conf /etc/redis.conf.bak.$(date +%F)
  ```

---

## 4. Current Configuration Check

Verify current settings:

```bash
grep -E "maxmemory|maxmemory-policy" /etc/redis.conf
```

Example output:

```text
maxmemory 268435456
maxmemory-policy noeviction
```

---

## 5. Procedure: Update Redis Memory Limit

### Step 1: Edit Redis Configuration

Open configuration file:

```bash
vi /etc/redis.conf
```

Modify or add:

```ini
maxmemory 536870912
```

### Recommended values:

| RAM Size | Redis maxmemory |
| -------- | --------------- |
| 2 GB VPS | 256MB           |
| 4 GB VPS | 512MB           |
| 8 GB VPS | 1–2GB           |
| 16 GB+   | 2–4GB           |

---

## 6. Procedure: Configure Eviction Policy

Set appropriate eviction policy:

```ini
maxmemory-policy allkeys-lru
```

### Policy options:

| Policy         | Behavior                                     |
| -------------- | -------------------------------------------- |
| noeviction     | Reject writes when full                      |
| allkeys-lru    | Evict least recently used keys (recommended) |
| volatile-lru   | Evict only keys with TTL                     |
| allkeys-random | Random eviction                              |

> ✅ Recommended for hosting environments: `allkeys-lru`

---

## 7. Apply Changes

Restart Redis service:

```bash
systemctl restart redis
```

Verify service status:

```bash
systemctl status redis
```

---

## 8. Validation Steps

### Check applied memory settings:

```bash
redis-cli INFO memory | grep maxmemory
```

Expected output:

```text
maxmemory:536870912
```

### Check eviction policy:

```bash
redis-cli CONFIG GET maxmemory-policy
```

Expected:

```text
1) "maxmemory-policy"
2) "allkeys-lru"
```

---

## 9. CageFS Considerations

* CageFS does not restrict Redis memory directly
* Redis changes are **global system-level changes**
* No per-user Redis tuning is supported unless:

  * Redis is containerized per user (non-standard)
  * Redis cluster per tenant is implemented

---

## 10. Troubleshooting

### Issue: Redis not restarting

Check logs:

```bash
journalctl -u redis -xe
```

Common causes:

* Invalid config syntax
* Memory value exceeds system RAM
* Port conflict

---

### Issue: OOM errors still occurring

Check system memory:

```bash
free -m
```

Consider:

* Increasing system RAM
* Reducing Redis maxmemory usage
* Reviewing application cache usage

---

### Issue: High eviction rate

Check stats:

```bash
redis-cli INFO stats | grep evicted_keys
```

If high:

* Increase maxmemory
* Tune application caching behavior

---

## 11. Safe Rollback Procedure

If issues occur:

```bash
cp /etc/redis.conf.bak.YYYY-MM-DD /etc/redis.conf
systemctl restart redis
```

---

## 12. Summary

* Redis is a **system-wide service in standard CloudLinux environments**
* Memory and eviction tuning must be done via `/etc/redis.conf`
* Recommended eviction policy: `allkeys-lru`
* Always restart Redis after changes
* Validate using `redis-cli INFO memory`

---

## 13. Final Notes for Support

* Never apply per-user Redis assumptions unless explicitly deployed
* Always verify service architecture before applying changes
* Prefer incremental memory adjustments
* Monitor after restart for at least 5–10 minutes
