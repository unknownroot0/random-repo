# Composer Installation Issue on cPanel EA-PHP 8.2

## Problem

While installing Composer using the standard installer:

```bash
curl -sS https://getcomposer.org/installer | php
```

the installer failed with the following error:

```text
The allow_url_fopen setting is incorrect.
Add the following to the end of your php.ini:
    allow_url_fopen = On
```

## Investigation

Checked the active PHP configuration:

```bash
php --ini
```

Output showed that the CLI PHP configuration was loaded from:

```text
/opt/cpanel/ea-php82/root/etc/php.ini
```

Verified the setting:

```bash
php -i | grep allow_url_fopen
```

Result:

```text
allow_url_fopen => Off => Off
```

This confirmed that `allow_url_fopen` was disabled for the command-line PHP environment.

## Resolution

Instead of modifying the global PHP configuration, the setting was enabled temporarily when running the Composer installer:

```bash
curl -sS https://getcomposer.org/installer -o composer-setup.php
php -d allow_url_fopen=On composer-setup.php
```

The installation completed successfully:

```text
Composer (version 2.10.0) successfully installed to: /home/USERNAME/composer.phar
```

## Moving Composer

Created a local bin directory and moved the Composer PHAR file:

```bash
mkdir -p ~/bin/composer
mv ~/composer.phar ~/bin/composer/
```

Verified the file:

```bash
ls -lh ~/bin/composer/
```

Output:

```text
composer.phar
```

## Common Mistake Encountered

The directory `~/bin/composer` already existed. Therefore:

```bash
mv ~/composer.phar ~/bin/composer
```

moved the file into the directory rather than renaming it.

As a result:

```bash
php ~/bin/composer --version
```

did not work because `~/bin/composer` was a directory.

The correct command was:

```bash
php ~/bin/composer/composer.phar --version
```

## Verification

```bash
php ~/bin/composer/composer.phar --version
```

Output:

```text
Composer version 2.10.0
PHP version 8.2.30
```

## Final Status

Composer was successfully installed and verified on a cPanel EA-PHP 8.2 environment without modifying the global PHP configuration. The issue was resolved by temporarily enabling `allow_url_fopen` using the `-d` PHP runtime option during installation.
