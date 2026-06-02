# How to Change PHP Version in cPanel

## Overview

cPanel allows you to change the PHP version used by your website easily. This is useful for compatibility, performance, and security reasons.

> **Important:** The PHP version used in the terminal (SSH/Terminal) is separate from the PHP version used by your website.

---

## 1. Check Current PHP Version (Terminal)

To check the PHP version in the terminal, run:

```bash id="phpver"
php -v
```

This shows the CLI (command-line) PHP version.

---

## 2. Change PHP Version for Your Website (Recommended Method)

To change the PHP version used by your website:

### Using MultiPHP Manager

1. Log in to **cPanel**
2. Open **MultiPHP Manager**
3. Select your domain
4. Choose the desired PHP version (e.g., 8.1, 8.2)
5. Click **Apply**

Your website will now run on the selected PHP version.

---

## 3. Verify Website PHP Version

Create a PHP info file in your website root:

```php id="phpinfo"
<?php phpinfo(); ?>
```

Then open it in your browser:

```
https://yourdomain.com/phpinfo.php
```

Check the **PHP Version** section.

> Remember to delete this file after checking for security reasons.

---

## 4. Using Specific PHP Versions in Terminal (Advanced Users)

On cPanel servers, multiple PHP versions are available via EA-PHP.

You can run a specific version directly:

### PHP 8.2

```bash id="php82"
 /opt/cpanel/ea-php82/root/usr/bin/php -v
```

### PHP 8.1

```bash id="php81"
 /opt/cpanel/ea-php81/root/usr/bin/php -v
```

To run a script using a specific version:

```bash id="runphp"
 /opt/cpanel/ea-php82/root/usr/bin/php script.php
```

---

## 5. Optional: Set Default PHP Alias (Terminal Only)

You can create a temporary alias:

```bash id="alias1"
alias php=/opt/cpanel/ea-php82/root/usr/bin/php
```

To make it permanent:

```bash id="alias2"
echo 'alias php=/opt/cpanel/ea-php82/root/usr/bin/php' >> ~/.bashrc
source ~/.bashrc
```

> This only affects terminal sessions and does NOT change website PHP version.

---

## 6. Important Notes

* Changing PHP in **MultiPHP Manager affects only websites**
* Terminal PHP is controlled separately by the server environment
* Some hosting providers restrict CLI PHP switching for security reasons
* Always ensure your application supports the selected PHP version before switching

---

## Why Change PHP Version?

### Compatibility

Some applications require specific PHP versions.

### Performance

Newer PHP versions improve speed and memory efficiency.

### Security

Older PHP versions may no longer receive security updates.

### Testing

Developers may need to test websites across multiple PHP versions.

---

## Summary

* Use **MultiPHP Manager** to change website PHP version
* Use full PHP paths for CLI when needed
* Terminal PHP and website PHP are not the same
