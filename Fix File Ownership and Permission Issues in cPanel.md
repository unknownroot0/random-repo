Incorrect file ownership or permissions can cause a variety of website issues, including:

* ❌ **403 Forbidden** errors
* ❌ File upload failures
* ❌ Website access problems
* ❌ Application or script malfunctions

This guide will help you reset file ownership and permissions to the recommended cPanel defaults.

> **Important:** Replace **`test`** in the commands below with your actual cPanel username.

---

## 📋 Recommended Permissions

| Item              | Permission |
| ----------------- | ---------- |
| Files             | `644`      |
| Directories       | `755`      |
| CGI-BIN Directory | `755`      |

These permission values provide a good balance between functionality and security.

---

## 1️⃣ Reset File Ownership

Ensure all files and directories are owned by the correct cPanel user:

```bash id="j8phx5"
chown -R test:test /home/test/public_html/
```

---

## 2️⃣ Set Correct File Permissions

Set all files under `public_html` to **644**:

```bash id="t7hrdb"
find /home/test/public_html/ -type f -exec chmod 644 {} \;
```

✅ This allows files to be readable by the web server while remaining secure.

---

## 3️⃣ Set Correct Directory Permissions

Set all directories under `public_html` to **755**:

```bash id="ffvyr8"
find /home/test/public_html/ -type d -exec chmod 755 {} \;
```

✅ This allows directories to be accessible while preventing unauthorized modifications.

---

## 4️⃣ Fix CGI-BIN Permissions (Optional)

If your website uses CGI scripts, ensure the `cgi-bin` directory has the proper permissions:

```bash id="98grjz"
chmod -R 755 /home/test/public_html/cgi-bin/
```

---

## 5️⃣ Verify Ownership Again

As a final step, reapply ownership across the account:

```bash id="3nz7zz"
chown -R test:test /home/test/public_html/
```

---

## 💡 Example

If your cPanel username is **john**, use:

```bash id="jsn5hz"
chown -R john:john /home/john/public_html/
```

---

## ⚠️ Important Notes

> **Run these commands as the root user via SSH.**

### Security Best Practices

* ✅ Use `644` for files
* ✅ Use `755` for directories
* ❌ Avoid `777` permissions unless specifically required
* ✅ Create a backup before making bulk permission changes

### Common Issues Resolved

These commands can help resolve:

* 403 Forbidden errors
* Ownership mismatches after migrations
* Permission errors following manual file uploads
* Script execution problems
* WordPress, Laravel, and other CMS permission issues

---

## 🎯 Summary

Running the commands above will:

* Restore proper file ownership
* Apply secure default permissions
* Fix many common website access issues
* Ensure compatibility with most cPanel-hosted applications

If problems persist after applying these changes, contact your hosting provider for further investigation.
