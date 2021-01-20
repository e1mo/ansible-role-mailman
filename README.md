# ansible-role-mailman

Ansible role to install and configure the [GNU Mailing List Manager](https://list.org/), also known as mailman.

## Requirements

- [Python](https://www.python.org/)3.6 or newer with [venv support](https://docs.python.org/3/library/venv.html)
- A working mail transfer agent (MTA)
  - This role can optionally create a _BASIC_ postfix configuration
  - [Mailman supports](https://docs.mailman3.org/projects/mailman/en/latest/src/mailman/docs/mta.html) postfix, exim, qmail and sendmail (strongly discouraged)
- A functioning web server
  - SSL is _highly recommended_
  - This role installs installs and configures gunicorn by default
  - This role has support for nginx
- Systemd: For services and sytemd timers
- A functioning database
  - Postgresql is recommended, this role only has been tested with postgresql
  - See geelingguys [ansible-role-postgresql](github.com/geerlingguy/ansible-role-postgresql) for a role to setup and configure postgresql
- Root access to the machine running debian or derivates
- At least one domain for the fronted and mail server

## Role variables

Available variables are listed below together with their default values as specified in `defaults/main.yml`.

### Generic environment configuration

```yml
mailman_user: "mailman"
mailman_group: "{{ mailman_user }}"
mailman_path: "/opt/mailman"
mailman_venv_path: "{{ mailman_path }}/venv"
```

Where to place the mailman files and configuration, who should be the owner and where to place the python virtual-env.

```yml
mailman_systemd_prefix: 'mailman-'
```

Prefix for the installed systemd services and timers (e.g. `mailman-gunicorn.service`, `mailman-notify.timer`)

```yml
mailman_apt_update: true
```

Update the apt-cache while installing mailman dependencies. Especially needed for older docker containers or outdated cloud images.

```yml
mailman_dependencies:
  - build-essential
  - sassc
  - memcached
  - python3-dev
  - python3-wheel
  - python3-setuptools
  - python3-virtualenv
  - memcached
  - libmemcached-dev
  - gettext
```

Apt packages that are required for mailman and the installation.

### Mailman core

```yml
mailman_siteowner: "user@example.com"
```

Address of the site owner, messages that _must_ reach a human are sent this way.

```yml
mailman_default_language: 'en'
```
Default language for created mailinglists


```yml
mailman_db_type: "sqlite"  # pgsql, mysql, sqlite (fallback for everything)
mailman_db_name: "mailman.sqlite3"
mailman_db_user: ""
mailman_db_pass: ""
mailman_db_host: ""
```

Database credentials for mailman core. SQLite is the default and fallback. In production, postgresql is recommended for performance reasons.

```yml
mailman_webservice_listen: localhost
mailman_webservice_port: 8001
mailman_webservice_use_https: false
mailman_webservice_show_tracebacks: false
mailman_webservice_admin_user: 'restadmin'
mailman_webservice_admin_pass: ''
mailman_webservice_workers: 2
```

Configuration for the mailman web service (internal REST-API). See <https://docs.mailman3.org/projects/mailman/en/latest/src/mailman/config/docs/config.html#webservice>.

```yml
mailman_mta_incomming: 'mailman.mta.postfix.LMTP'
mailman_mta_outgoing: 'mailman.mta.deliver.deliver'
```

Incoming and outgoing MTA. Directly mapps to <https://docs.mailman3.org/projects/mailman/en/latest/src/mailman/config/docs/config.html#mta>

```yml
mailman_mta_smtp_host: 'localhost'
mailman_mta_smtp_port: 25
mailman_mta_smtp_user: ''
mailman_mta_smtp_pass: ''
mailman_mta_smtp_secure_mode: 'smtp'
mailman_mta_smtp_verify_cert: true
mailman_mta_smtp_verify_hostname: true
```

Credentials and SSL-options for SMTP.

```yml
mailman_mta_lmtp_host: '127.0.0.1'
mailman_mta_lmtp_port: 8024
```

Configuration for mailmans own LMTP server.

```yml
mailman_mta_max_recipients: 500
mailman_mta_max_sessions_per_connection: 0
mailman_mta_max_delivery_threads: 0
mailman_mta_delivery_retry_period: '5d'
mailman_mta_max_autoresponses_per_day: 10
```

Tweaks to mailmans sending behaviour.

### Postorius

[Postorius](https://docs.mailman3.org/projects/postorius/en/latest/) is the [django](https://djangoproject.com/)-baed web frontend for managing lists and users. It also integrates with hyperkitty as a mail archiver.

```yml
mailman_postorius_base_url: 'http{{ "s" if mailman_nginx_ssl }}://{{ mailman_nginx_server_name }}'  # -> https://lists.example.org
```

The public URL to use in Templates and so on. Defaults to the nginx-configured values to save some typing but can be overwritten.

```yml
mailman_postorius_language_code: 'en-us'
```

Default language of the postorius fronted. Mapped to the `LANGUAGE_CODE` setting in Django. See [here](https://docs.djangoproject.com/en/2.2/ref/settings/#language-code) for more information.

```yml
mailman_postorius_secret_key: null
```

Secret key for [securing the application](https://docs.djangoproject.com/en/2.2/ref/settings/#std:setting-SECRET_KEY), _must_ be set.

```yml
mailman_postorius_debug: false
mailman_postorius_siteid: 1
```

Debug enables stack traces on errors, the site\_id specifies helps with some multi-domain setup and can normally stay 1. This stack-overflow has a good explanation what it's used for: <https://stackoverflow.com/a/25468782>

```yml
mailman_postorius_db_engine: 'sqlite3'  # mysql, sqlite3 or postgresql_psycopg2
mailman_postorius_db_name: 'postorius.sqlite3'
mailman_postorius_db_user: ''
mailman_postorius_db_pass: ''
mailman_postorius_db_host: ''
mailman_postorius_db_port: ''
```

Database connectivity for postorius, directly maps to the django settings: <https://docs.djangoproject.com/en/2.2/ref/settings/#std:setting-DATABASES>. Postgresql (`postgresql_psycopg2`) is, due to its superior performance, recommended in production. SQLite3 should only be used for testing purposes.

```yml
mailman_postorius_email_backend: 'django.core.mail.backends.smtp.EmailBackend'
mailman_postorius_email_host: '{{ mailman_mta_smtp_host }}'
mailman_postorius_email_port: '{{ mailman_mta_smtp_port }}'
mailman_postorius_email_host_user: '{{ mailman_mta_smtp_user }}'
mailman_postorius_email_host_pass: '{{ mailman_mta_smtp_pass }}'
mailman_postorius_email_default_from_email: '{{ mailman_postorius_email_host_user }}'
mailman_postorius_email_confirmation_from: '{{ mailman_postorius_email_default_from_email }}'
mailman_postorius_email_server_email: '{{ mailman_postorius_email_default_from_email }}'
mailman_postorius_email_use_ssl: true
```

Configuration on how postorius will send E-Mail. It uses the same MTA as mailman-core by default. See <https://docs.djangoproject.com/en/2.2/topics/email/#email-backends> for additional backends and how to configure them.

```yml
mailman_postorius_time_zone: '{{ ansible_date_time["tz"] }}'
```

The timezone used by postorius, defaults to the systems current timezone.

_Note for Regions with summer-/wintertime:_ It is advised to set the timezone manually since ansible does not provide the location of the timezone (e.g. `Europe/Berlin`) but the timezone according to the offset (CET with `+01:00` in the winter and CEST with `+02:00` in the summer). Therefore, you would have to re-run this role after each time change.

```yml
mailman_postorius_allowed_hosts:
  - 'localhost'
  - '127.0.0.1'
  - '::1'
  - '::ffff::127.0.0.1'
  - '{{ mailman_nginx_server_name }}'
```

Hosts and domain names django can serve content for, if a request redirected to django does not match this list, the request will be rejected.

```yml
mailman_postorius_admins:
  - name: Mailman Admin
    email: mailman@localhost
```

Errors in the Django-Application will be sent to the listed E-Mails: <https://docs.djangoproject.com/en/2.2/ref/settings/#admins>

```yml
mailman_postorius_archiver_from:
  - '127.0.0.1'
  - '::1'
  - '::ffff::127.0.0.1'
```

The archiver API can only be accessed from listed IPs. Since hyperkitty is running locally in this setup, it should be safe to leave unchanged.

```yml
mailman_postorius_cache_backend: 'django.core.cache.backends.memcached.PyLibMCCache'
mailman_postorius_cache_location: 'localhost'
```

The cache used for all sorts of django-caches. By default it usese memcached which is also installed by this role. See the django documentation for possible settings: <https://docs.djangoproject.com/en/2.2/ref/settings/#std:setting-CACHES>

### Hyperkitty (archiver)

```yml
mailman_archiver_key: 'changeme!'
```

Shared secret between hyperkitty and mailman for authentication. _Must_ be set.

```yml
mailman_hyperkitty_base_url: '{{ mailman_postorius_base_url }}'
```

Base URL for the hyperkitty installation. Normally, there is no need to change it. But in setups with external reverse proxy it may be necessary to change it to something like `127.0.0.1:{{ mailman_gunicorn_port }}`.

```yml
mailman_hyperkitty_serach_index: false
```

Rebuild hyperkitty search index after each update.

### Gunicorn

[Gunicorn](https://gunicorn.org/) is the WSGI HTTP server that runs postorius.

```yml
mailman_gunicorn_listen: '127.0.0.1'
mailman_gunicorn_port: 8000
```

IP and Port to bind to.

```yml
mailman_gunicorn_procname: '{{ mailman_systemd_prefix }}gunicorn'
```

Title of the process, useful for differentiating multiple gunicorn processes.

```yml
mailman_gunicorn_workers: 4
```

Amount of worker-processes to handle requests.

### 3rd Party Helpers

This role is able to do some basic configuration of nginx and postfix. Those configurations are _not especially secure_ nor complete at all. It just aims to help out a bit. Be sure you know what those services are doing with your specific configuration as you don't want to host an [open mail relay](https://en.wikipedia.org/wiki/Open_mail_relay), [really not](https://www.spamhaus.org/news/article/706/the-return-of-the-open-relays).

```yml
mailman_postfix_setup: false
mailman_postfix_smtp_port: "smtp"
```

Should this role install postfix and provide the necessary configuration for mailman to work with postfix? The `smtp` value of port translates to port `25` (SMTP) in postfix.

_Note_: This is really only recommended for testing purposes. Refer to <https://docs.mailman3.org/projects/mailman/en/latest/src/mailman/docs/mta.html#transport-maps> about the mailman specific configuration for postfix. The transport-maps are located at `{{ mailman_path }}/var/data/{postfix_lmtp,postfix_domains}`.

```yml
mailman_nginx_enabled: false
mailman_nginx_server_name: 'lists.example.org'
mailman_nginx_ssl: true
mailman_nginx_path: '/etc/nginx/sites-enabled/mailman.conf'
mailman_nginx_ssl_redirect: true
mailman_nginx_v6only: false
mailman_nginx_ssl_certificate: null
mailman_nginx_ssl_key: null
```

Set `mailman_nginx_enabled` to `true` in order to install a nginx configuration to the specified location and restart nginx afterwards.

_Note_: This role will neither install nginx nor generate SSL-Certificates from e.g. letsencrypt for you. That's on you. You can find some roles for that in the [related projects](#related-projects) section.

## Example Playbook

This is a small sample playbook utilizing most of the roles features. To run this playbook you need to have the postgresql database available. The lines ending in `# Change` contain secrets which should be changed in production. Same goes for domain names, E-Mail contacts and so on.

```yml
- hosts: mailman
  become: true
  vars_file:
    - vars/mailman.yml
  roles:
    - e1mo.mailman
```

_Inside `vars/mailman.yml`_

```yml
mailman_siteowner: 'postmaster@mydomain.com'

mailman_db_type: 'pgsql'
mailman_db_name: 'mailman'
mailman_db_user: 'mailman'
mailman_db_pass: 'totally_secret'  # Change

mailman_webservice_admin_pass: 'also_secret'  # Change

mailman_mta_smtp_port: 465
mailman_mta_smtp_user: 'mailman@mydomain.com'
mailman_mta_smtp_pass: '1337secret42'  # Change
mailman_mta_smtp_secure_mode: 'smtps'

mailman_postorius_secret_key: 'fooSecretBar'  # Change

mailman_postorius_db_engine: 'postgresql_psycopg2'
mailman_postorius_db_name: 'mailman_postorius'
mailman_postorius_db_user: 'mailman_postorius'
mailman_postorius_db_pass: 'no-less-secret'  # Change
mailman_postorius_db_host: 'localhost'

mailman_postorius_email_default_from_email: 'noreply@lists.mydomain.com'

mailman_postorius_time_zone: 'Europe/Berlin'

mailman_postorius_admins:
  - name: Fellow Hacker
    email: noc@hacker.org

mailman_archiver_key: 'not-leaking-that-one'  # Change

# This is **no especially secure** postfix setup.
# Running your own, properly configured, Mailserver is highly advised.
mailman_postfix_setup: true

# You need to have nginx installed before hand
mailman_nginx_enabled: true
mailman_nginx_path: '/etc/nginc/sites-enabled/lists.mydomain.com.conf'
mailman_nginx_server_name: 'lists.mydomain.com'

mailman_nginx_ssl_certificate: '/etc/ssl/certs/ssl-cert-snakeoil.pem'
mailman_nginx_ssl_key: '/etc/ssl/private/ssl-cert-snakeoil.key'
```

## License

GNU General Public License v3.0 or later. See [LICENSE](LICENSE) for the full license.

## Related projects

- The [mailman project](https://www.list.org/)
- The [ansible](https://github.com/ansible/ansible) project itself, without which this role could not exist.
- [ansible-role-postgresql](https://github.com/geerlingguy/ansible-role-postgresql) from [Jeff Geerling aka. geerlingguy](https://github.com/geerlingguy) which can install and configure postgresql easily.
- [ansible-nginx-base](https://github.com/rixx/ansible-nginx-base) and [ansible-dehydrated](https://github.com/rixx/ansible-dehydrated) (get letsencrypt certs) by [rixx](https://rixx.de/) are both really simple to work with and don't get in your way. I have patched and updated both roles a fair bit, just haven't submitted the patches: [ansible-nginx-base](https://github.com/e1mo/ansible-nginx-base) and [ansible-dehydrated](https://github.com/e1mo/ansible-dehydrated).

## Author information

Written by [Moritz 'e1mo' Fromm](https://e1mo.de).

The role is developed on [sourcehut](https://sr.ht) at <https://git.sr.ht/~e1mo/ansible-role-mailman>. To contribute send your patches to `~e1mo/ansible-role-mailman [at] lists.sr.ht` using [`git send-email`](https://git-send-email.io/) ([Mailing list etiquette](https://man.sr.ht/lists.sr.ht/etiquette.md)). The issue-tracker is located at <https://todo.sr.ht/~e1mo/ansible-role-mailman>, no account needed.

