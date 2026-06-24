

**Applies to:** LiteSpeed / OpenLiteSpeed environments with per‑user Redis isolation via CageFS  
**Audience:** System administrators, senior DevOps engineers  
**Last validated:** 2026-06-22  

---

## Overview

Each hosted user with Redis access runs an isolated instance inside their CageFS. The memory cap is defined by a quota file (`redis.size`) and enforced by the service wrapper script `/usr/local/lsws/lsns/bin/redis_svc.sh`. By default, the instance uses the `noeviction` policy, which will hard‑crash the application when the memory limit is reached. This article describes how to safely increase the memory limit and inject a safer eviction policy (`allkeys-lru`) so that keys are evicted automatically instead of causing application errors.

---

## Prerequisites

- **Root** or equivalent `sudo` access to the host server.
- Target username (e.g., `cozymar1`).
- The Redis instance must be managed by the LiteSpeed CageFS wrapper; the script path `/usr/local/lsws/lsns/bin/redis_svc.sh` must exist.
- The user’s CageFS base directory (typically `/home2/<username>`) must be accessible.

---

## Procedure

### 1. Remove the immutable file attribute

The quota file is protected by the extended filesystem immutable flag (`+i`). Strip it to allow editing:

```bash
chattr -i /home2/cozymar1/.cagefs/tmp/redis/redis.size
```

### 2. Temporarily grant write permission

The file normally has read‑only permissions (`400`). Change it to allow the root user to write:

```bash
chmod 600 /home2/cozymar1/.cagefs/tmp/redis/redis.size
```

### 3. Edit the memory limit value

Open the file with an editor and replace the old value with the new limit in **megabytes** (plain integer, no unit suffix).  
Example: changing from `64` (64 MB) to `256` (256 MB).

```bash
vi /home2/cozymar1/.cagefs/tmp/redis/redis.size
```

### 4. Restore original permissions and re‑apply immutability

```bash
chmod 400 /home2/cozymar1/.cagefs/tmp/redis/redis.size
chattr +i /home2/cozymar1/.cagefs/tmp/redis/redis.size
```

### 5. Restart the user’s Redis instance

The service wrapper must be called with the `stop` and `start` subcommands and the username.  
**Important:** The `start` command may run the daemon in the foreground of your shell session, blocking the terminal. To handle this, you can:

- Use `Ctrl+Z` to suspend the process, then `bg` to push it into the background, **or**
- Prepend `nohup` and redirect output, **or**
- Run the command inside a `screen`/`tmux` session.

```bash
/usr/local/lsws/lsns/bin/redis_svc.sh stop cozymar1
/usr/local/lsws/lsns/bin/redis_svc.sh start cozymar1   # handle foreground as needed
```

After the start command, the new memory limit from `redis.size` is loaded.

### 6. Hot‑inject an eviction policy (if needed)

Out‑of‑the‑box Redis instances default to `noeviction`, causing `OOM` errors when the memory limit is hit. We inject the `allkeys-lru` policy at runtime via the user’s Unix socket:

```bash
redis-cli -s /home2/cozymar1/.cagefs/tmp/redis.sock config set maxmemory-policy allkeys-lru
```

> **⚠ Warning:** This change is **not persistent**. A service restart will revert the policy to the default. To make the policy permanent, see the section below.

---

## Verification

Query the active instance directly to confirm the limit and policy:

```bash
redis-cli -s /home2/cozymar1/.cagefs/tmp/redis.sock info memory | grep -E "maxmemory|policy"
```

Expected output:

```
maxmemory:268435456
maxmemory_policy:allkeys-lru
```

- `maxmemory` is shown in bytes: 256 × 1024 × 1024 = **268,435,456** (256 MB).
- `maxmemory_policy` is `allkeys-lru` — Least Recently Used eviction across all keys.

---

## Making the Eviction Policy Permanent

The `redis_svc.sh` script builds the Redis configuration from a template that does **not** read `redis.size` for the policy. To survive restarts, you must permanently add the policy to the instance configuration.

### Recommended approach

Locate the Redis configuration file template used by the wrapper. Common locations:

- `/usr/local/lsws/lsns/etc/redis.conf.template`  
- A per‑user config inside the CageFS (e.g., `/home2/cozymar1/.cagefs/tmp/redis/redis.conf`)

Add or modify the line:

```
maxmemory-policy allkeys-lru
```

Then restart the user’s instance to pick up the permanent change.

If no per‑user template exists, consider adding a post‑start hook in the wrapper script, or use a startup script that executes the `redis-cli config set` command automatically after the instance is online (e.g., a `systemd` unit with an `ExecStartPost` line, if applicable).

---

## Important Notes & Caveats

- **Isolation boundary:** All changes are strictly confined to the user’s CageFS environment. No global Redis instances (port `6379`) or other tenant accounts are affected.
- **Foreground process handling:** The start command may attach to the terminal. Always ensure the process is safely backgrounded to avoid accidental termination when you log out.
- **File permissions:** The `redis.size` file should remain immutable and read‑only (`400`) under normal operation to prevent accidental or malicious modifications.
- **Pre‑change backup:** Before modifying any quota file, consider making a backup (`cp redis.size redis.size.bak`), especially if the environment has compliance requirements.

---

## Reference

- **Service script:** `/usr/local/lsws/lsns/bin/redis_svc.sh`
- **User socket path:** `/home2/<username>/.cagefs/tmp/redis.sock`
- **Quota file:** `/home2/<username>/.cagefs/tmp/redis/redis.size`

---

*This article is ready for internal knowledge base or client‑facing documentation submission. For any questions, contact the DevOps team.*
