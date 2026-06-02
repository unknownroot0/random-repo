# Composer Installation Issue on cPanel EA-PHP 8.2

## Problem

Composer installation failed with:

```text
The allow_url_fopen setting is incorrect.
Add the following to php.ini:
allow_url_fopen = On
```

## Investigation

Checked PHP configuration:

```bash
php --ini
```

Loaded configuration file:

```text
/opt/cpanel/ea-php82/root/etc/php.ini
```

Verified setting:

```bash
php -i | grep allow_url_fopen
```

Result:

```text
allow_url_fopen => Off => Off
```

## Resolution

Instead of modifying system `php.ini`, Composer was installed by enabling the setting only for runtime:

```bash
curl -sS https://getcomposer.org/installer -o composer-setup.php
php -d allow_url_fopen=On composer-setup.php
```

Installation succeeded:

```text
Composer (version 2.10.0) successfully installed to: /home/USERNAME/composer.phar
```

## Moving Composer

A local bin directory was created:

```bash
mkdir -p ~/bin/composer
mv ~/composer.phar ~/bin/composer/
```

Composer is then accessed via:

```bash
php ~/bin/composer/composer.phar --version
```

## Issue Encountered

A directory conflict occurred because `~/bin/composer` already existed, causing the PHAR file to be placed inside it instead of being renamed.

Correct path became:

```text
~/bin/composer/composer.phar
```

## Alias Setup (Final Step)

To simplify usage, an alias was added:

```bash
echo 'alias composer="php $HOME/bin/composer/composer.phar"' >> ~/.bashrc
source ~/.bashrc
```

This allows Composer to be used globally as:

```bash
composer --version
composer install
composer update
```

## Verification

```bash
composer --version
```

Output:

```text
Composer version 2.10.0
PHP version 8.2.30
```

## Final Status

Composer was successfully installed on a cPanel EA-PHP 8.2 environment without modifying global PHP configuration. The issue was resolved using a runtime override (`-d allow_url_fopen=On`) and a user-level alias for ease of use.
