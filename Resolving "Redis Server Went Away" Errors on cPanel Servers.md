## Overview

This article documents a real-world troubleshooting scenario involving a cPanel server configured with Redis and Redis PHP caching. The issue resulted in severe WordPress performance degradation and was ultimately traced back to incorrect file permissions preventing the Redis service from starting.

This guide explains the symptoms, investigation process, root cause, and resolution steps to help administrators quickly identify and resolve similar issues.

---

## Server Environment

The affected server was configured as follows:

* **Hosting Platform:** Managed VPS Server
* **Control Panel:** cPanel/WHM
* **Operating System:** CentOS 7.9.2009 (STANDARD)
* **cPanel Version:** 110.0.17
* **Web Server:** Apache
* **PHP Versions:** 7.2 – 8.1
* **Additional Services:**

  * Redis
  * Redis PHP Cache
  * Ruby 2.7
  * Node.js 16

---

## Problem Description

A client reported that their WordPress website had become extremely slow and was experiencing poor overall performance.

During the investigation, a recently created WordPress error log was discovered. Although the file was less than 24 hours old, it had already grown significantly and was increasing rapidly.

Reviewing the log revealed the same error being generated repeatedly, often more than 100 times per second:

```text
[07-Jan-2024 21:04:10 UTC] Redis server went away
[07-Jan-2024 21:04:10 UTC] Redis server went away
[07-Jan-2024 21:04:10 UTC] Redis server went away
[07-Jan-2024 21:04:10 UTC] Redis server went away
[07-Jan-2024 21:04:10 UTC] Redis server went away
[07-Jan-2024 21:04:12 UTC] Redis server went away
```

Since the website relied heavily on Redis for object caching, the repeated connection failures were causing:

* Cache requests to fail
* Increased PHP processing overhead
* Excessive error logging
* Higher server load
* Significant website performance degradation

---

## Investigation

The next step was to verify the status of the Redis service using systemd.

Running:

```bash
systemctl status redis
```

produced output similar to the following:

```text
Starting Redis persistent key-value database...
redis-server[19791]: *** FATAL CONFIG FILE ERROR (Redis 7.2.3) ***
redis-server[19791]: Can't open the log file: Permission denied
systemd[1]: redis.service: main process exited, code=exited, status=1/FAILURE
systemd[1]: Failed to start Redis persistent key-value database.
systemd[1]: Unit redis.service entered failed state.
systemd[1]: redis.service failed.
```

The critical message was:

```text
Can't open the log file: Permission denied
```

This indicated that Redis was unable to access or write to its configured log file, causing the service to fail during startup.

---

## Root Cause

The issue was caused by incorrect file and directory permissions associated with Redis logging.

Because Redis could not write to its log location, the service failed to start entirely. As a result:

1. Redis caching became unavailable.
2. WordPress repeatedly attempted to communicate with Redis.
3. Each failed connection generated a PHP error.
4. PHP workers became occupied handling and logging these errors.
5. Website performance deteriorated significantly.

In environments where Redis is heavily utilized for object caching, this type of failure can have a substantial impact on both application performance and server resources.

---

## Resolution

To restore service, ownership and permissions on the Redis log directory were corrected.

### Step 1: Fix Redis Log Directory Ownership

```bash
chown -R redis:redis /var/log/redis
```

This ensures that the Redis service account owns the log directory and its contents.

### Step 2: Correct Log Directory Permissions

```bash
chmod -R u+rwX,g+rwX,u+rx /var/log/redis
```

These permissions allow Redis to read, write, and access the required log files and directories.

### Step 3: Verify Redis Configuration File Access

```bash
chmod +r /etc/redis/redis.conf
```

This ensures that the Redis configuration file remains readable by the service during startup.

### Step 4: Restart Redis

After correcting permissions, restart the service:

```bash
systemctl restart redis
```

Verify that Redis is running successfully:

```bash
systemctl status redis
```

---

## Verification

Once Redis was successfully restarted:

* Redis object caching resumed normal operation.
* WordPress stopped generating "Redis server went away" errors.
* Error log growth ceased.
* PHP worker utilization returned to normal levels.
* Website performance improved immediately.

---

## Preventive Measures

To prevent similar issues in the future:

* Periodically verify ownership and permissions of Redis log directories.
* Monitor Redis service status using automated monitoring tools.
* Configure alerts for Redis service failures.
* Review system updates or maintenance procedures that may inadvertently modify file permissions.
* Ensure Redis log paths are writable by the Redis service account.

Additionally, a permanent permissions adjustment was implemented on the affected server to prevent the same conditions from recurring.

---

## Conclusion

A seemingly simple permissions issue prevented Redis from accessing its log file, causing the service to fail completely. Because the affected WordPress site depended heavily on Redis caching, the outage resulted in excessive PHP errors, increased server load, and poor website performance.

When troubleshooting WordPress performance issues on servers utilizing Redis, always verify that the Redis service is running correctly and check for permission-related errors in the service logs. Resolving these underlying issues can often restore performance immediately.
