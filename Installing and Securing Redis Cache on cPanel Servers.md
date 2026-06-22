## Overview

This guide explains how to install, optimize, and secure Redis on cPanel servers running AlmaLinux 8/9, RockyLinux 8/9, or Ubuntu 22.04/24.04. It covers:

* Redis server installation
* Redis PHP extension installation
* Performance tuning
* Security hardening
* Verification and troubleshooting

Redis is commonly used for object caching in applications such as WordPress, Magento, and Laravel, helping reduce database load and improve website performance.

---

# Prerequisites

Before you begin, ensure you have:

* Root SSH access
* WHM/cPanel installed
* At least 1 GB RAM (2 GB+ recommended)
* A recent server backup
* Internet connectivity for package installation

---

# Step 1: Access Your Server

You can manage Redis through SSH or WHM.

### WHM Access

```
https://your-server-ip:2087
```

Log in using the root account.

### SSH Access

Connect via SSH:

```bash
ssh root@your-server-ip
```

---

# Step 2: Install Redis Server

## AlmaLinux / RockyLinux 8 & 9

Update the system and install Redis:

```bash
dnf update -y
dnf install epel-release -y

dnf install https://rpms.remirepo.net/enterprise/remi-release-$(rpm -E %rhel).rpm -y

dnf --enablerepo=remi install redis -y

systemctl enable redis
systemctl start redis
systemctl status redis
```

Verify Redis is running:

```bash
redis-cli ping
```

Expected output:

```text
PONG
```

---

## Ubuntu 22.04 / 24.04

Install Redis:

```bash
apt update -y
apt install redis-server -y

systemctl enable redis-server
systemctl start redis-server
systemctl status redis-server
```

Verify installation:

```bash
redis-cli ping
```

Expected output:

```text
PONG
```

---

# Step 3: Secure Redis

Edit the Redis configuration file:

**RHEL-based systems**

```bash
vi /etc/redis.conf
```

**Ubuntu**

```bash
vi /etc/redis/redis.conf
```

Configure the following:

```conf
bind 127.0.0.1 ::1
protected-mode yes

requirepass YourStrongPasswordHere
```

Restart Redis:

```bash
systemctl restart redis
```

Ubuntu:

```bash
systemctl restart redis-server
```

---

# Step 4: Install the Redis PHP Extension

The Redis PHP extension allows PHP applications to communicate with Redis.

## Method 1: EasyApache 4 (Recommended)

In WHM:

1. Navigate to **EasyApache 4**
2. Click **Customize**
3. Select **PHP Extensions**
4. Search for **redis**
5. Enable the required packages:

Examples:

```text
ea-php81-php-redis
ea-php82-php-redis
ea-php83-php-redis
```

6. Click **Review**
7. Click **Provision**

---

## Method 2: PECL Installation

If EasyApache packages are unavailable:

```bash
/opt/cpanel/ea-php81/root/usr/bin/pecl install redis
```

Add the extension to PHP:

```ini
extension=redis.so
```

You can add this via:

**WHM → MultiPHP INI Editor**

or manually in the appropriate `php.ini` file.

---

# Step 5: Verify Installation

Check that the PHP module is loaded:

```bash
php -m | grep redis
```

Expected output:

```text
redis
```

Create a PHP test file:

```php
<?php phpinfo(); ?>
```

Save as:

```text
public_html/info.php
```

Open it in a browser and search for:

```text
redis
```

A Redis section confirms successful installation.

---

# Troubleshooting

## Redis Service Won't Start

Check service logs:

```bash
journalctl -u redis
```

or

```bash
journalctl -u redis-server
```

---

## CloudLinux/CageFS

Refresh CageFS after installation:

```bash
cagefsctl --force-update
```

---

## Firewall Issues

Ensure Redis is not publicly accessible on port 6379.

---

# Redis Performance Tuning

For production environments, especially WordPress hosting, optimize Redis using the following recommendations.

---

## 1. Linux Kernel Optimization

### Disable Transparent Huge Pages (THP)

THP can cause latency spikes and increased memory usage.

Temporarily disable:

```bash
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag
```

### Enable Memory Overcommit

Redis recommends enabling memory overcommit:

```bash
sysctl vm.overcommit_memory=1
```

Make it persistent:

```bash
cat <<EOF >> /etc/sysctl.conf
vm.overcommit_memory = 1
vm.nr_hugepages = 0
EOF

sysctl -p
```

---

## 2. Configure Memory Limits

Edit Redis configuration:

```conf
maxmemory 1gb
maxmemory-policy allkeys-lru
maxmemory-samples 5
```

### Recommended Policies

| Use Case                 | Policy       |
| ------------------------ | ------------ |
| General caching          | allkeys-lru  |
| Frequently accessed data | allkeys-lfu  |
| Mixed cache/database     | volatile-lru |

A good rule is to allocate approximately 70–80% of available RAM.

Restart Redis:

```bash
systemctl restart redis
```

---

## 3. Optimize Persistence

### Balanced Configuration

```conf
save 900 1
save 300 10
save 60 10000

appendonly yes
appendfsync everysec

auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
```

### Cache-Only Servers

For maximum performance:

```conf
appendonly no
```

Disable all `save` directives.

This sacrifices persistence but improves speed.

---

## 4. Additional Performance Tweaks

```conf
tcp-keepalive 60
timeout 0

maxclients 10000

slowlog-log-slower-than 10000
slowlog-max-len 128
```

For Redis 6+:

```conf
io-threads 4
io-threads-do-reads yes
```

---

## 5. Monitor Performance

### Check Memory Usage

```bash
redis-cli INFO memory
```

### Check Statistics

```bash
redis-cli INFO stats
```

### View Active Configuration

```bash
redis-cli CONFIG GET maxmemory
```

### Latency Testing

```bash
redis-cli --latency
```

---

# Recommended Redis Settings

| Setting                | Recommended Value   |
| ---------------------- | ------------------- |
| vm.overcommit_memory   | 1                   |
| Transparent Huge Pages | Disabled            |
| maxmemory              | 70–80% of RAM       |
| maxmemory-policy       | allkeys-lru         |
| appendfsync            | everysec            |
| save                   | Minimal or disabled |

---

# Redis Security Hardening

Redis should never be exposed directly to the internet.

---

## 1. Bind Redis to Localhost

Configure:

```conf
bind 127.0.0.1 ::1
protected-mode yes
```

Restart Redis.

---

## 2. Enable Authentication

### Legacy Password Authentication

```conf
requirepass VeryStrongRandomPassword
```

Test:

```bash
redis-cli
AUTH VeryStrongRandomPassword
PING
```

Expected response:

```text
PONG
```

---

### Redis 6+ ACL Authentication

Disable the default user:

```conf
user default off
```

Create an application user:

```conf
user cacheuser on >StrongPassword ~* +@all
```

Update your applications to use the new credentials.

---

## 3. Disable Dangerous Commands

To prevent abuse, disable high-risk commands:

```conf
rename-command FLUSHALL ""
rename-command FLUSHDB ""
rename-command CONFIG ""
rename-command KEYS ""
rename-command EVAL ""
```

---

## 4. Restrict Access with a Firewall

### AlmaLinux / RockyLinux (firewalld)

```bash
firewall-cmd --permanent --remove-port=6379/tcp --zone=public
firewall-cmd --reload
```

### Ubuntu (UFW)

```bash
ufw allow from 127.0.0.1 to any port 6379
ufw deny 6379
ufw reload
```

If using ConfigServer Security & Firewall (CSF), ensure port 6379 is not publicly accessible.

---

## 5. Additional Security Recommendations

### Verify Redis Runs as a Non-Root User

```bash
ps aux | grep redis
```

### Keep Redis Updated

RHEL-based systems:

```bash
dnf update redis
```

Ubuntu:

```bash
apt update
apt upgrade redis-server
```

### Monitor Redis Logs

```bash
tail -f /var/log/redis/redis.log
```

### Limit Memory Usage

Always configure `maxmemory` to help prevent resource exhaustion attacks.

---

# Security Best Practices Summary

| Security Measure   | Recommendation              |
| ------------------ | --------------------------- |
| Network Binding    | localhost only              |
| Authentication     | Strong password or ACLs     |
| Dangerous Commands | Disable when possible       |
| Firewall           | Block public access to 6379 |
| Updates            | Apply regularly             |
| Monitoring         | Review logs and metrics     |

---

# Conclusion

Redis is one of the most effective ways to improve application performance on cPanel servers. By combining proper installation, memory tuning, and security hardening, you can provide fast and reliable object caching for WordPress and other PHP applications while keeping your server secure and stable.

For most cPanel hosting environments, the recommended configuration is:

* Redis bound to localhost
* Authentication enabled
* `maxmemory` set to 70–80% of available RAM
* `allkeys-lru` eviction policy
* `appendfsync everysec`
* Public access to port 6379 blocked
