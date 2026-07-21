# SOP-APACHE-001: Resolution of Addon Domain Misrouting to Primary Domain via VirtualHost Reconstruction

**Document Control:**
*   **Version:** 1.0
*   **Status:** Active
*   **Author:** Senior Linux Systems Engineer
*   **Last Updated:** July 21, 2026
*   **Applicable Systems:** cPanel/WHM, EasyApache 4, Apache HTTPD


# Phase 1 — Investigation & Verification

Your technical explanation of the incident is **fundamentally correct and aligns with official cPanel and Apache documentation**. Below is the independent verification of your claims:

1. **Apache VirtualHost Selection**: Your understanding is accurate. According to the official Apache HTTP Server Documentation, "Name-based virtual hosts for the best-matching set of `<VirtualHost>`s are processed in the order they appear in the configuration. The first matching `ServerName` or `ServerAlias` is used." [[21]] If the primary domain’s VirtualHost appears first and contains a stale `ServerAlias` for the addon domain, Apache will serve the primary domain’s `DocumentRoot`.
2. **cPanel Configuration Hierarchy**: Your understanding of `userdata` vs. `httpd.conf` is correct. cPanel uses YAML files in `/var/cpanel/userdata/<username>/` as the **source of truth** for domain configurations. [[2]] The `/etc/apache2/conf/httpd.conf` file is a generated artifact. 
3. **Stale Data Persistence**: cPanel explicitly warns that `/etc/apache2/conf/httpd.conf` "should not be edited directly, as cPanel overwrites this file for many reasons." [[44]] However, if a manual edit was made, a previous rebuild was interrupted, or a migration artifact persisted, `httpd.conf` can drift from `userdata`.
4. **Repair Mechanism**: The `/usr/local/cpanel/scripts/rebuildhttpdconf` script is the official cPanel method to "rebuild Apache's main configuration file" based on the `userdata` source of truth. [[1]]

**Conclusion of Phase 1**: Your root cause hypothesis and proposed repair sequence are technically sound and represent industry best practices for cPanel environments.

---

# Phase 2 — Root Cause Analysis

* **Why an addon domain appears to redirect**: It is typically *not* an HTTP 301/302 redirect. Instead, the client sends a `Host: addon.com` header. Apache matches this to the primary domain’s VirtualHost (due to the stale `ServerAlias`) and serves the primary domain’s `index.php` or `index.html`. The URL in the browser bar does not change, but the content is wrong.
* **How Apache selects a VirtualHost**: Apache first matches the IP address and port (e.g., `*:80` or `*:443`). Among all matching VirtualHosts, it evaluates `ServerName` and `ServerAlias` directives. The **first** match in the configuration file order wins. [[21]]
* **How duplicate ServerAlias entries affect selection**: If both the primary and addon VirtualHosts claim `bestbd.com.bd`, Apache stops searching at the first occurrence (the primary domain) and routes the traffic there.
* **How cPanel generates Apache VirtualHosts**: cPanel’s `apache_conf_distiller` and `rebuildhttpdconf` scripts read the YAML files in `/var/cpanel/userdata/<username>/` and render them into `/etc/apache2/conf/httpd.conf` using internal templates.
* **What userdata is**: A directory (`/var/cpanel/userdata/`) containing YAML files that represent the definitive configuration state of every domain, subdomain, and addon domain for each cPanel account.
* **userdata vs. httpd.conf**: `userdata` is the **source** (intent); `httpd.conf` is the **output** (execution). 
* **Why stale VirtualHost data can remain**: 
  1. Manual edits to `httpd.conf` (which are overwritten only on the next rebuild, but can cause issues if a rebuild is never run).
  2. An interrupted `/scripts/rebuildhttpdconf` process (e.g., server reboot, OOM killer).
  3. Legacy migration artifacts from older cPanel versions or third-party control panels.
* **How `rebuildhttpdconf` works**: It reads all valid `userdata` files, validates them, and completely overwrites `/etc/apache2/conf/httpd.conf` with a freshly generated configuration, eliminating any manual or stale drift. [[1]]
* **Why `configtest` must always be run**: It validates the syntax of the newly generated `httpd.conf`. If a syntax error exists, restarting Apache will cause a service outage.
* **Why restarting Apache completes the repair**: Apache only reads `httpd.conf` into memory at startup or reload. The restart applies the corrected VirtualHost mapping.
* **Why DNS troubleshooting failed**: DNS operates at Layer 3/4 (resolving `bestbd.com.bd` to the server’s IP). The issue was at Layer 7 (HTTP Host header routing). The traffic was reaching the correct server, but the web server was misdirecting it internally.
* **Other scenarios with similar symptoms**: 
  - Missing or misconfigured `ServerAlias` in the *addon* domain’s userdata.
  - Browser or server-level caching (e.g., Varnish, Redis, or Cloudflare) serving an old redirect.
  - `.htaccess` rules in the primary domain’s document root forcing a rewrite based on the requested host.

---

# Phase 3 — Diagnostic Workflow

| Step | Action & Command | Purpose | Expected Output | Abnormal Output | Interpretation |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **1** | `curl -I http://bestbd.com.bd` | Check raw HTTP response headers. | `HTTP/1.1 200 OK` (or 301/302 if a real redirect exists). | `HTTP/1.1 200 OK` but `Server` header shows primary domain, or content is wrong. | Confirms if it’s a true HTTP redirect or a VirtualHost misrouting. |
| **2** | `dig +short bestbd.com.bd` | Verify DNS resolution. | Server’s public IP address. | Different IP or NXDOMAIN. | If IP is wrong, fix DNS. If IP is correct, proceed to Layer 7. |
| **3** | `apachectl -S \| grep bestbd.com.bd` | List all VirtualHosts claiming the domain. | Should show **only one** entry pointing to the addon domain’s config file. | Shows **two or more** entries (e.g., primary domain and addon domain). | Confirms duplicate VirtualHost claims. |
| **4** | `grep -i "bestbd.com.bd" /etc/apache2/conf/httpd.conf` | Inspect the active Apache configuration. | Should appear only under the addon domain’s `<VirtualHost>` block. | Appears in *both* the primary and addon `<VirtualHost>` blocks. | Confirms stale `ServerAlias` in `httpd.conf`. |
| **5** | `grep -i "bestbd.com.bd" /var/cpanel/userdata/*/*` | Inspect the cPanel source of truth. | Should appear **only** in `/var/cpanel/userdata/<user>/bestbd.com.bd`. | Appears in `/var/cpanel/userdata/<user>/primary.com`. | If found in primary’s userdata, the userdata itself is corrupt and must be fixed *before* rebuilding. |
| **6** | `diff` comparison | Compare userdata vs. httpd.conf. | N/A | Mismatch found. | Confirms configuration drift. |

---

# Phase 4 — Repair Procedure

**Prerequisite**: Always take a manual backup of the current configuration before rebuilding.
```bash
cp /etc/apache2/conf/httpd.conf /etc/apache2/conf/httpd.conf.pre-rebuild-$(date +%F)
```

| Command | Why it is used | Internal Changes | Expected Output | Possible Failures | Recovery Actions |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `1. /usr/local/cpanel/scripts/rebuildhttpdconf` | Regenerates `httpd.conf` strictly from `/var/cpanel/userdata/`. | Overwrites `/etc/apache2/conf/httpd.conf`. Removes stale `ServerAlias` entries not present in userdata. | `info [rebuildhttpdconf] ... Built /etc/apache2/conf/httpd.conf` | Fails if userdata is corrupt, missing, or hostname is invalid. | Restore from `.pre-rebuild` backup. Fix userdata YAML manually, then retry. |
| `2. /usr/sbin/apachectl configtest` | Validates Apache configuration syntax before applying. | Reads `httpd.conf` and included files, checking for syntax errors. | `Syntax OK` | `Syntax error on line X...` | **Do not restart.** Review line X in `httpd.conf`. Fix the underlying userdata or remove offending custom includes. |
| `3. /usr/local/cpanel/scripts/restartsrv_httpd` | Safely restarts the Apache daemon to load the new config. | Sends SIGTERM to old processes, starts new processes with new config. | `Waiting for “httpd” to restart gracefully…started.` | Fails to start, leaving Apache down. | Run `systemctl status httpd` and `journalctl -xeu httpd`. Revert to backup: `cp /etc/apache2/conf/httpd.conf.pre-rebuild-* /etc/apache2/conf/httpd.conf` and restart. |

**Why this sequence is recommended**: It follows the immutable infrastructure principle: *Generate from source of truth → Validate → Apply*. Skipping `configtest` risks taking down all websites on the server if a syntax error is introduced.

---

# Phase 5 — Validation

After the repair, verify the following:
1. **VirtualHost Ownership**: `apachectl -S | grep bestbd.com.bd` should now return exactly **one** match, pointing to the addon domain’s user and document root.
2. **Configuration Validity**: `apachectl configtest` returns `Syntax OK`.
3. **Website Behavior**: `curl -I http://bestbd.com.bd` returns `200 OK`, and the HTML content matches the addon domain’s expected site (verify via `curl -s http://bestbd.com.bd | grep -i "addon domain specific text"`).
4. **SSL Behavior**: `curl -I https://bestbd.com.bd` should return `200 OK` with a valid certificate for `bestbd.com.bd` (not the primary domain’s certificate).
5. **No Redirects**: Ensure the response is `200 OK`, not `301 Moved Permanently` (unless intentionally configured).

---

# Phase 6 — Edge Cases

* **Addon vs. Parked/Alias Domains**: Parked domains *intentionally* share the primary domain’s VirtualHost. Ensure the domain was actually added as an *Addon* (which creates a separate DocumentRoot and VirtualHost) and not a Parked domain.
* **Custom Apache Includes**: Files in `/etc/apache2/conf.d/includes/` are appended to `httpd.conf`. A custom include could contain a rogue `ServerAlias` that survives a `rebuildhttpdconf`. Always grep the entire `/etc/apache2/conf/` directory, not just `httpd.conf`.
* **Manual `httpd.conf` Edits**: Any manual changes to `httpd.conf` are destroyed by `rebuildhttpdconf`. Customizations must be placed in `/etc/apache2/conf.d/includes/` or via cPanel’s Include Editor in WHM.
* **Userdata Corruption**: If the YAML file in `/var/cpanel/userdata/` is itself corrupt or missing the addon domain, `rebuildhttpdconf` will omit the domain entirely. In this case, run `/scripts/updateuserdatacache` or rebuild the userdata file first. [[2]]
* **Multiple cPanel Accounts Claiming Same Domain**: cPanel generally prevents this at the UI level, but if forced via CLI or database manipulation, `rebuildhttpdconf` may behave unpredictably. Use `/scripts/whoowns bestbd.com.bd` to verify.
* **AutoSSL**: If the VirtualHost was misconfigured for an extended period, AutoSSL may have failed to issue/renew the certificate. After fixing the VirtualHost, trigger a manual AutoSSL check in WHM.
* **Reverse Proxy (e.g., LiteSpeed, Nginx)**: If the server uses a reverse proxy, `apachectl` commands still apply to the backend Apache, but you must also verify the proxy’s configuration (e.g., `/etc/nginx/nginx.conf` or LiteSpeed VHost maps) and restart the proxy service.

---

# Phase 7 — Rollback

If `rebuildhttpdconf` introduces new problems (e.g., missing custom configurations, service failure):
1. **Immediate Service Restoration**: 
   ```bash
   cp /etc/apache2/conf/httpd.conf.pre-rebuild-YYYY-MM-DD /etc/apache2/conf/httpd.conf
   /usr/local/cpanel/scripts/restartsrv_httpd
   ```
2. **Investigate the Failure**: Check `/usr/local/cpanel/logs/error_log` and `/etc/apache2/logs/error_log` to determine why the rebuild failed or produced an undesirable outcome.
3. **Correct the Source of Truth**: If the rebuild failed due to bad userdata, edit the specific YAML file in `/var/cpanel/userdata/<user>/` to fix syntax or missing directives, then run `/scripts/updateuserdatacache` before attempting the rebuild again.
4. **Restore Custom Includes**: If custom includes were accidentally disrupted, verify their presence in `/etc/apache2/conf.d/includes/` and restore from server backups if necessary.

---

# Phase 8 — Standard Operating Procedure (SOP)

# SOP: Resolution of Addon Domain Misrouting to Primary Domain

## 1. Title
Resolution of Addon Domain Misrouting to Primary Domain via Apache VirtualHost Reconstruction

## 2. Purpose
To provide a standardized, safe, and verifiable procedure for diagnosing and resolving incidents where an addon domain serves the primary domain’s content due to Apache VirtualHost configuration drift.

## 3. Scope
Applies to all cPanel/WHM servers running EasyApache 4 where DNS resolution is verified as correct, but HTTP traffic is misrouted at the web server layer.

## 4. Background
cPanel manages Apache configurations by generating `/etc/apache2/conf/httpd.conf` from YAML source files located in `/var/cpanel/userdata/`. Under certain conditions (manual edits, interrupted processes, migration artifacts), the generated `httpd.conf` can drift from the source of truth, resulting in duplicate `ServerAlias` entries. Apache processes VirtualHosts in order, meaning the first matching entry wins, causing addon domains to incorrectly serve the primary domain’s content.

## 5. Incident Summary
- **Symptom**: Addon domain displays primary domain content.
- **Initial Findings**: DNS is correct and propagating. No Layer 3/4 issues.
- **Root Cause**: Stale `ServerAlias` in the primary domain’s VirtualHost within `/etc/apache2/conf/httpd.conf`.
- **Resolution**: Rebuild Apache configuration from cPanel userdata, validate syntax, and restart the service.

## 6. Symptoms
- Browser shows addon domain URL, but primary domain content.
- `curl -I` shows `200 OK` (not a 301/302 redirect).
- `apachectl -S` shows the domain claimed by multiple VirtualHosts.

## 7. Prerequisites
- Root SSH access to the cPanel server.
- Verification that DNS resolves to the correct server IP.
- Identification of the affected cPanel username and domain name.

## 8. Safety Checks
- [ ] Verify current Apache status: `systemctl status httpd`
- [ ] Create a manual backup of the current configuration: 
  `cp /etc/apache2/conf/httpd.conf /etc/apache2/conf/httpd.conf.pre-rebuild-$(date +%F)`

## 9. Diagnostic Procedure
1. Verify HTTP response: `curl -I http://<addon_domain>`
2. Verify DNS: `dig +short <addon_domain>`
3. Check VirtualHost mapping: `apachectl -S | grep <addon_domain>`
4. Check active config for duplicates: `grep -i "<addon_domain>" /etc/apache2/conf/httpd.conf`
5. Check source of truth for duplicates: `grep -i "<addon_domain>" /var/cpanel/userdata/*/*`
   - *Decision Point*: If the domain is found in the *primary* domain’s userdata file, the userdata is corrupt. Fix the userdata YAML before proceeding. If it is only in `httpd.conf`, proceed to Repair.

## 10. Repair Procedure
Execute the following sequence as `root`:
```bash
# 1. Regenerate configuration from userdata
/usr/local/cpanel/scripts/rebuildhttpdconf

# 2. Validate syntax (CRITICAL: Do not skip)
/usr/sbin/apachectl configtest

# 3. Apply changes
/usr/local/cpanel/scripts/restartsrv_httpd
```

## 11. Validation
- [ ] `apachectl configtest` returns `Syntax OK`.
- [ ] `apachectl -S | grep <addon_domain>` returns exactly one match.
- [ ] `curl -I http://<addon_domain>` returns `200 OK` with correct content.
- [ ] `curl -I https://<addon_domain>` returns `200 OK` with the correct SSL certificate.

## 12. Rollback
If the service fails to start or behavior worsens:
1. Restore backup: `cp /etc/apache2/conf/httpd.conf.pre-rebuild-YYYY-MM-DD /etc/apache2/conf/httpd.conf`
2. Restart service: `/usr/local/cpanel/scripts/restartsrv_httpd`
3. Escalate to Senior Systems Engineer for userdata corruption analysis.

## 13. Troubleshooting Matrix
| Symptom | Probable Cause | Action |
| :--- | :--- | :--- |
| `rebuildhttpdconf` fails with "Missing IP" | Server hostname or IP configuration is invalid. | Verify WHM Basic Setup and IP mapping. |
| `apachectl configtest` fails | Syntax error in generated config or custom include. | Check error output, inspect line number, fix userdata or remove bad include. |
| Addon domain shows 404 after rebuild | Addon domain missing from userdata entirely. | Run `/scripts/updateuserdatacache` or manually restore userdata YAML. |
| SSL shows primary domain cert | AutoSSL failed to run after VHost fix. | Trigger AutoSSL via WHM → Manage AutoSSL. |

## 14. Lessons Learned
- Never edit `/etc/apache2/conf/httpd.conf` directly. Use cPanel’s Include Editor or `/etc/apache2/conf.d/includes/`.
- DNS correctness does not guarantee HTTP correctness; always verify Layer 7 VirtualHost mapping.
- Always validate configuration syntax before restarting critical web services.

## 15. References
- cPanel Documentation: [The rebuildhttpdconf Script](https://docs.cpanel.net/whm/scripts/the-rebuildhttpdconf-script/) [[1]]
- cPanel Documentation: [How to Rebuild userdata Files](https://docs.cpanel.net/knowledge-base/accounts/how-to-rebuild-userdata-files/) [[2]]
- cPanel Documentation: [How to Edit the Apache Configuration File](https://support.cpanel.net/hc/en-us/articles/360057430173) [[44]]
- Apache HTTP Server Documentation: [Name-based Virtual Host Support](https://httpd.apache.org/docs/current/vhosts/name-based.html) [[21]]
