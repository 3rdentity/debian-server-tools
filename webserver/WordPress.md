# WordPress website lifecycle

## Where do I find ...?

- Development environment: [/webserver/WP-config-dev.md](/webserver/WP-config-dev.md)
- Development tools: wordpress-sitebuild/
- Production environment: [/webserver/Production-website.md](/webserver/Production-website.md)
- Production on cPanel and migration to cPanel: [wordpress-plugin-construction/shared-hosting-aid/cPanel/README.md](https://github.com/szepeviktor/wordpress-plugin-construction/blob/master/shared-hosting-aid/cPanel/README.md)
- Content plugins: [wordpress-plugin-construction/README.md](https://github.com/szepeviktor/wordpress-plugin-construction/blob/master/README.md)
- WordPress installation: standard, subdirectory (optionally using git) [in this document](#standard-directory-structure)
- WordPress imigration: dev->live, live->other domain [/webserver/Production-website.md](/webserver/Production-website.md#migration)


### Standard Directory structure

```
wp-cli.yml
wp-config.php
DOCROOT/─┬─index.php
         ├─wp-load.php
         ├─wp-login.php
         ├─xmlrpc.php
         ├─wp-admin/
         ├─wp-includes/
         └─wp-content/
```


### Subdirectory structure

```
wp-cli.yml
DOCROOT/─┬─index.php (modified)
         ├─wp-config.php
         ├─wp-login.php (trap)
         ├─xmlrpc.php (trap)
         ├─CORE/─┬─index.php
         │       ├─wp-load.php
         │       ├─wp-login.php
         │       ├─wp-admin/
         │       └─wp-includes/
         └─static/ (wp-content)
```


### Half-secret structure

```
wp-cli.yml
DOCROOT/─┬─index.php (modified)
         ├─wp-config.php
         ├─wp-login.php (trap)
         ├─xmlrpc.php (trap)
         ├─CORE/─┬─index.php
         │       ├─wp-load.php
         │       ├─wp-login.php
         │       ├─wp-admin/
         │       └─wp-includes/
         └─wp-content/
```


### Installation by WP-CLI

`wp-cli.yml`

```yaml
path: $WPROOT
url: $WPHOMEURL
debug: true
user: viktor
core update:
    locale: hu_HU
skip-plugins:
    # Version randomizer
    - better-wp-security
```

```bash
# Existing database
./wp-createdb.sh

# New installation
wp core download --locale=hu_HU
wp core config --dbname="$DBNAME" --dbuser="$DBUSER" --dbpass="$DBPASS" \
    --dbhost="$DBHOST" --dbprefix="prod" --dbcharset="$DBCHARSET" --extra-php <<EOF
// Extra PHP code
EOF
wp core install --title="WP" --admin_user="viktor" --admin_email="viktor@szepe.net" --admin_password="12345"

wp option set home "$WPHOMEURL"
wp option set blog_public "0"
wp option set admin_email "support@company.net"
```

@TODO Move to wp-lib

### Remove default content

```bash
wp post delete $(wp post list --name="$(wp eval 'echo sanitize_title( _x( "hello-world", "Default post slug" ) );')" --posts_per_page=1 --format=ids)
wp post delete $(wp post list --post_type=page --name="$(wp eval 'echo __( "sample-page" );')" --posts_per_page=1 --format=ids)
wp comment delete 1
wp option update blogdescription ""
wp plugin uninstall akismet
wp plugin uninstall hello-dolly
wp theme delete twentyfifteen
wp theme delete twentyfourteen
```

### Use child theme

Purchased themes can be updated using a child theme.

```bash
wp plugin install child-theme-configurator --activate
```

Keep changes in git.

### Redis object cache as a service

[Free 30 MB Redis instance by redislab](https://redislabs.com/redis-cloud)


### Development

See /webserver/Wp-config-dev.md


### Plugins

#### Core

```bash
export WPSZV="https://github.com/szepeviktor/wordpress-plugin-construction/raw/master"
mkdir wp-content/mu-plugins/

# InnoDB table engine
wget -qO- https://github.com/szepeviktor/debian-server-tools/raw/master/mysql/alter-table.sql \
 | mysql -N $(wp eval 'echo DB_NAME;') | mysql

# disable updates
wget -P wp-content/mu-plugins/ ${WPSZV}/mu-disable-updates/disable-updates.php

# disable comments
wget -P wp-content/mu-plugins/ https://github.com/solarissmoke/disable-comments-mu/raw/master/disable-comments-mu.php
wget -P wp-content/mu-plugins/disable-comments-mu/ https://github.com/solarissmoke/disable-comments-mu/raw/master/disable-comments-mu/comments-template.php

# disable feeds
#wp plugin install disable-feeds --activate

# smilies
wp plugin install classic-smilies --activate

# mail
wp plugin install wp-mailfrom-ii smtp-uri --activate
#wget -P wp-content/mu-plugins/ https://github.com/danielbachhuber/mandrill-wp-mail/raw/master/mandrill-wp-mail.php
wp eval 'var_dump(wp_mail("admin@szepe.net","First outgoing",site_url()));'
# define( 'SMTP_URI', 'smtp://FOR-THE-WEBSITE%40DOMAIN.TLD:PWD@localhost' );

# multilanguage
wp plugin install polylang --activate
```

#### Security

```bash
# users/login

wp plugin install password-bcrypt
cp -v wp-content/plugins/password-bcrypt/wp-password-bcrypt.php wp-content/mu-plugins/
wp plugin uninstall password-bcrypt
wp plugin install user-session-control --activate
# user role editor
wp plugin install user-role-editor --activate
# keepass-button
wget -P wp-content/mu-plugins/ ${WPSZV}/mu-keepass-button/keepass-button.php

# Fail2ban Wordpress
wget https://github.com/szepeviktor/wordpress-fail2ban/raw/master/block-bad-requests/wp-fail2ban-bad-request-instant.inc.php
wget -P wp-content/mu-plugins/ https://github.com/szepeviktor/wordpress-fail2ban/raw/master/mu-plugin/wp-fail2ban-mu-instant.php

# security suite + audit

# audit
wp plugin install wp-user-activity --activate
# simple audit
wp plugin install simple-history --activate
# Sucuri
wp plugin install custom-sucuri sucuri-scanner --activate

# prevent spam

# Installation: https://github.com/szepeviktor/wordpress-plugin-construction/tree/master/mu-nofollow-robot-trap
wget -P wp-content/mu-plugins/ ${WPSZV}/mu-nofollow-robot-trap/nofollow-robot-trap.php
# CF7 Robot Trap
wget -P wp-content/plugins/ ${WPSZV}/contact-form-7-robot-trap/cf7-robot_trap.php
# obfuscate-email
#wp plugin install obfuscate-email --activate
```

#### Forcing

```bash
# mu-lock-session-ip
wget -P wp-content/mu-plugins/ ${WPSZV}/mu-lock-session-ip/lock-session-ip.php

# prevent-concurrent-logins
wp plugin install prevent-concurrent-logins --activate

# mu-disallow-weak-passwords
wget -P wp-content/mu-plugins/ ${WPSZV}/mu-disallow-weak-passwords/disallow-weak-passwords.php

# mu-banned-email-addresses
wget -P wp-content/mu-plugins/ ${WPSZV}/mu-banned-email-addresses/banned-email-addresses.php

# media
wget -P wp-content/mu-plugins/ ${WPSZV}/mu-image-upload-control/image-upload-control.php
wget -P wp-content/mu-plugins/ ${WPSZV}/mu-image-upload-control/image-upload-control-hu.php

# protect plugins
wget -P wp-content/mu-plugins/ ${WPSZV}/mu-protect-plugins/protect-plugins.php
```

#### Object cache

```php
// In wp-config.php
define( 'WP_CACHE_KEY_SALT', 'SITE-SHORT_' );
$redis_server = array(
    'host'     => '127.0.0.1',
    'port'     => 6379,
    'auth'     => '12345',
    'database' => 0,
);
```

```bash
wget -P wp-content/mu-plugins/ ${WPSZV}/mu-cache-flush-button/flush-cache-button.php

# Redis @danielbachhuber
wp plugin install wp-redis --activate
wp redis enable
wp transient delete-all

# Memcached @HumanMade
wget -P wp-content/ https://github.com/humanmade/wordpress-pecl-memcached-object-cache/raw/master/object-cache.php
wp transient delete-all

# Memcache (no "d") @Automattic
#wget -P wp-content/ https://github.com/Automattic/wp-memcached/raw/master/object-cache.php
#wp transient delete-all

# APCu
# WARNING! APCu is not available from CLI by default during WP-Cron/WP-CLI
#wget -P wp-content/ https://github.com/l3rady/WordPress-APCu-Object-Cache/raw/master/object-cache.php
#wp transient delete-all
# Worse plugin: wp plugin install apcu

# FOCUS Cache - FILE-based object cache
#wp plugin install focus-object-cache
wget -P wp-content/ ${WPSZV}/focus-cache/object-cache.php

# Tiny cache
wget -P wp-content/mu-plugins/ https://github.com/szepeviktor/tiny-cache/raw/master/tiny-translation-cache.php
wget -P wp-content/mu-plugins/ https://github.com/szepeviktor/tiny-cache/raw/master/tiny-nav-menu-cache.php
wget -P wp-content/mu-plugins/ https://github.com/szepeviktor/tiny-cache/raw/master/tiny-cache.php
```


#### Optimize HTML + HTTP

Resource optimization

```bash
# JPEG image quality
# add_filter( 'jpeg_quality', function ( $quality ) { return 91; } );

# resource-versioning
wp plugin install resource-versioning --activate

# Tiny CDN
wp plugin install tiny-cdn --activate

# CDN, Page Cache, Minify
#wp plugin install w3-total-cache --activate
#wp plugin install https://github.com/szepeviktor/w3-total-cache-fixed/releases/download/0.9.5.4.2/w3-total-cache-fixed-for-v0.9.5.x-users.zip --activate

# minit
wp plugin install https://github.com/kasparsd/minit/archive/master.zip
wp plugin install https://github.com/markoheijnen/Minit-Pro/archive/master.zip

# safe redirect manager
wp plugin install safe-redirect-manager --activate

# WP-FFPC
# backends: APCu, Memcached with ngx_http_memcached_module
# https://github.com/petermolnar/wp-ffpc
wp plugin install https://github.com/petermolnar/wp-ffpc/archive/master.zip --activate

## Autoptimize - CONFLICTS with resource-versioning
##     define( 'AUTOPTIMIZE_WP_CONTENT_NAME', '/static' );
#wp plugin install autoptimize --activate
```

Set up CDN.


#### Plugin fixes

MU Plugin Template

`disable-3rd-party-updates.php`

```php
<?php
/*
Plugin Name: disable custom updates (MU)
Version: 0.0.0
Description: This MU plugin contains hooks to disable custom updates.
Plugin URI: https://github.com/szepeviktor/debian-server-tools/blob/master/webserver/WordPress.md#plugin-fixes
Author: Viktor Szépe
*/
```

See /website/wordpress/ directory.

### On deploy and Staging->Production Migration

@TODO Move to Production-website.md

Also in /webserver/Production-website.md

- `wp transient delete-all`
- `wp db query "DELETE FROM $(wp eval 'global $table_prefix;echo $table_prefix;')options WHERE option_name LIKE '%_transient_%'"`
- Remove development wp_options -> Option Inspector
- Delete unused Media files @TODO `for $m in files; search $m in DB;`
- `wp db optimize`
- WP Cleanup


#### Settings

- General Settings
- Writing Settings
- Reading Settings
- Media Settings
- Permalink Settings
- WP Mail From


#### Search & replace items

```bash
wp search-replace --precise --recurse-objects --all-tables-with-prefix ...
wp search-replace --precise --recurse-objects --all-tables-with-prefix ...
wp search-replace --precise --recurse-objects --all-tables-with-prefix ...
wp search-replace --precise --recurse-objects --all-tables-with-prefix ...
```

1. http://DOMAIN.TLD or https (no trailing slash)
1. /home/PATH/TO/SITE (no trailing slash)
1. EMAIL@ADDRESS.ES (all addresses)
1. DOMAIN.TLD (now without http)

Manually replace constants in `wp-config.php`


### Moving a site to a subdirectory

1. siteurl += /site
1. search-and-replace: /wp-includes/ -> /site/wp-includes/
1. search-and-replace: /wp-content/ -> /static/

```bash
wget -O srdb.php https://github.com/interconnectit/Search-Replace-DB/raw/master/index.php
wget https://github.com/interconnectit/Search-Replace-DB/raw/master/srdb.class.php
```

### Signature

```bash
wget -P wp-content/mu-plugins/ https://github.com/szepeviktor/wordpress-sitebuild/raw/master/theme-development/Signature.php
```
