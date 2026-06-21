Automating MySQL database backups is a great way to ensure your data is regularly protected without requiring manual intervention. This guide explains how to create a simple cron job that downloads and stores a compressed backup of your database directly within your hosting account.

---

## 📋 Cron Job Command

Add the following command to your cron job:

```bash id="a85yuy"
curl http://cPanelusername:password@domain.com:2082/getsqlbackup/DB.gz \
--output /home/username/DB.gz
```

---

## 🔄 Replace the Placeholders

Before using the command, replace the following values with your actual account details:

| Placeholder      | Description                                        |
| ---------------- | -------------------------------------------------- |
| `cPanelusername` | Your cPanel username                               |
| `password`       | Your cPanel password                               |
| `domain.com`     | Your domain name                                   |
| `DB`             | Your MySQL database name (e.g., `username_dbname`) |
| `username`       | Your Linux/cPanel account username                 |

### Example

```bash id="n68h42"
curl http://john:MyPassword123@example.com:2082/getsqlbackup/john_wpdb.gz \
--output /home/john/john_wpdb.gz
```

---

## ⏰ Creating the Cron Job

1. Log in to **cPanel**.
2. Navigate to **Advanced → Cron Jobs**.
3. Choose your desired backup frequency.
4. Paste the backup command into the **Command** field.
5. Click **Add New Cron Job**.

### Example Schedule

Run the backup every day at 2:00 AM:

```text id="43ctg4"
0 2 * * *
```

---

## 📁 Where Is the Backup Stored?

After the cron job runs successfully, the compressed database backup file will be saved in your account's home directory:

```text id="5uz1if"
/home/username/DB.gz
```

For example:

```text id="6pc7xq"
/home/john/john_wpdb.gz
```

---

## 📥 Accessing Your Backup

You can download the backup file using:

* 📂 **cPanel File Manager**
* 🌐 **FTP/SFTP**
* 🖥️ **SSH**

The backup is stored as a compressed `.gz` file to reduce disk usage.

---

## ⚠️ Security Notice

> **Avoid using your cPanel password directly in cron job commands whenever possible.**

Including passwords in plain text may expose sensitive credentials to users who can view cron job configurations or process lists.

For improved security:

* Use API authentication methods if supported by your hosting provider.
* Restrict access to cron configurations.
* Regularly update your cPanel password.
* Store backups in a secure location.

---

## ✅ Benefits of Automated Database Backups

* Scheduled and unattended backups
* Quick recovery in case of data loss
* Reduced risk of human error
* Compressed backup files to save disk space
* Easy download through File Manager or FTP

---

## 🎯 Summary

By creating a simple cron job, you can automatically generate and store compressed MySQL database backups within your cPanel account. This provides a reliable backup strategy and helps ensure your website data remains protected.
