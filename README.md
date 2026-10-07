# Docker Magento2: single-node nginx + php-fpm production baseline

This stack is now being refactored from a cluster/bootstrap-style setup into a simpler production baseline for a **single Linux host**.

## What this stack provides

- `nginx` as the public frontend
- `php-fpm` for Magento application traffic
- separate `cron` container for Magento cron jobs
- `mysql` for the database
- Redis for cache and sessions
- OpenSearch for catalog search

## What changed from the old setup

- removed the Apache/mod_php bootstrap path
- removed the default Varnish/SSL-terminator flow from the primary compose file
- removed cluster autodiscovery and dynamic VCL updates
- moved toward local builds instead of prebuilt third-party images

## Prerequisites

- Docker and Docker Compose
- a Magento 2 project that can be installed into the shared `html_data` volume
- access keys for `repo.magento.com`

### PHP version note

This stack is currently built on **PHP 8.3** because **Magento 2.4.9 requires PHP 8.3+**.
If you try to install 2.4.9 on PHP 8.2, Composer will fail with a version mismatch.

The search backend in this baseline uses **OpenSearch 2.x**. Set `SEARCH_ENGINE=opensearch`.
Do not use `elasticsearch7`, because Magento 2.4.9 does not accept it.

## Environment

Copy the example env file and fill in secrets:

```bash
cp .env.example .env
```

At minimum, set:

- `API_KEY`
- `API_SECRET`
- `ADMIN_EMAIL`
- `ADMIN_PASSWORD`
- `MYSQL_PASSWORD`
- `MYSQL_ROOT_PASSWORD`

Timezone must be a valid IANA identifier such as `Europe/Warsaw` or `UTC`.
Do not use abbreviations like `CEST`; Magento rejects them during `setup:install`.

### Two-Factor Authentication (admin login)

Magento enables Two-Factor Authentication for the admin panel by default, which requires a working
email/authenticator setup. For local/dev environments without email configured, set:

```dotenv
DISABLE_2FA=yes
```

before running the bootstrap helper, and it will disable the `Magento_TwoFactorAuth` and
`Magento_AdminAdobeImsTwoFactorAuth` modules automatically. Leave it unset (or `no`) for anything
resembling production.

If Magento is already installed and you just want to disable 2FA without reinstalling:

```bash
docker compose exec app sh -lc 'cd /var/www/html/magento2 && php bin/magento module:disable Magento_TwoFactorAuth Magento_AdminAdobeImsTwoFactorAuth && php bin/magento cache:flush'
```

### Additional languages (e.g. Polish)

Magento ships only a handful of official language packs via Composer (`de_DE`, `en_US`, `es_ES`, `fr_FR`,
`nl_NL`, `pt_BR`, `zh_Hans_CN`). Everything else, including Polish, has to be added as an extra package.
A well-maintained community package for Polish is `snowdog/language-pl_pl`.

To have a language pack installed and its static content deployed automatically on a fresh install, set:

```dotenv
EXTRA_COMPOSER_PACKAGES=snowdog/language-pl_pl
STATIC_CONTENT_LOCALES=en_US pl_PL
```

before running the bootstrap helper. Both variables are space-separated lists, so you can add more than
one package/locale.

To add a language to an already-installed instance without reinstalling:

```bash
docker compose exec app sh -lc 'cd /var/www/html/magento2 && composer require --no-interaction snowdog/language-pl_pl && php bin/magento setup:upgrade'
docker compose exec app sh -lc 'cd /var/www/html/magento2 && rm -rf generated/code/* generated/metadata/* && php bin/magento setup:di:compile'
docker compose exec app sh -lc 'cd /var/www/html/magento2 && php bin/magento setup:static-content:deploy -f en_US pl_PL && php bin/magento cache:flush'
```

The newly installed locale then becomes available under
**Stores > Configuration > General > Locale Options > Locale** (set per store view), and in the admin
user's own locale preference under **System > Permissions > All Users**.

Composer's global auth for `repo.magento.com` now lives on the `html_data` volume
(`COMPOSER_HOME=/var/www/html/.composer-home`), so it survives the `app` container being rebuilt/recreated;
earlier revisions of this image stored it in `/tmp`, which was lost on every rebuild.

## Start the stack

```bash
docker compose up -d --build
```

## Install Magento once

Run the bootstrap helper only on a fresh volume:

```bash
docker compose run --rm app /usr/local/bin/install_magento.sh
```

## Useful notes

- `nginx` listens on port `80` in this first production pass.
- The app container does **not** auto-install Magento at boot anymore.
- `cron` only runs Magento cron jobs; clustering helpers were removed.
- `php.ini` lives in `docker-magento2-apache-php/php.ini` and is copied into the app image.

## Next step

The next iteration is to add proper TLS termination back into the new `nginx` container or move it to a host-level reverse proxy with real certificates.
