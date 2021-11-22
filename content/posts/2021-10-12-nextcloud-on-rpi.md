---
title: "Nextcloud on RPi4 with Arch Linux"
date: 2021-10-12
---

I experimented with [Nextcloud][] 22 on a Raspberry Pi 4 8 GB,
running the aarch64 build of Arch Linux ARM.
I found the setup too slow to be useful: even with caching,
switching between tabs on the web interface takes seconds,
while `top` shows `php-fpm` consuming a lot of CPU.
Either further tuning would be required, or this hardware is simply
too slim for this kind of workload, or more efficient software needs to be produced.

For the reference, I share my configuration below.
It is a summation of the [Arch Linux nextcloud documentation][arch-next],
and of the pages referenced by it, including the official documentation,
choosing one particular configuration, instead of presenting many alternatives:
PostgreSQL, nginx, php, php-fpm and nextcloud.

## PostgreSQL

Install the package, init a user and database:

    # pacman -Syu postgresql
    # su -l postgres
    [postgres]$ initdb --locale=en_US.UTF-8 -E UTF8 -D /var/lib/postgres/data
    [postgres]$ createuser -h localhost -P nextcloud
    [postgres]$ createdb -O nextcloud nextcloud

Make it listen on a local UNIX domain socket only:

    # vim /var/lib/postgres/data/postgresql.conf
    listen_addresses = ''

## nginx

Install the package and setup a site, according to the [nextcloud nginx docs][]:

    # pacman -Syu nginx
    $ cat /etc/nginx/nginx.conf
    user http;
    worker_processes  4;

    events {
        worker_connections  128;
    }

    http {
        include       mime.types;
        default_type  application/octet-stream;


        sendfile        on;
        keepalive_timeout  65;

        # https://wiki.archlinux.org/title/Nginx#Warning:_Could_not_build_optimal_types_hash
        types_hash_max_size 4096;
        server_names_hash_bucket_size 128;

        upstream php-handler {
            server unix:/run/nextcloud/nextcloud.sock;
        }

        server {
            listen       80;
            server_name  <your_hostname>;
            root /usr/share/webapps;

            location ^~ /.well-known {
              root /usr/share/webapps/nextcloud;

              # The rules in this block are an adaptation of the rules
              # in the Nextcloud `.htaccess` that concern `/.well-known`.
              location = /.well-known/carddav { return 301 /nextcloud/remote.php/dav/; }
              location = /.well-known/caldav  { return 301 /nextcloud/remote.php/dav/; }
              location /.well-known/acme-challenge    { try_files $uri $uri/ =404; }
              location /.well-known/pki-validation    { try_files $uri $uri/ =404; }
              # Let Nextcloud's API for `/.well-known` URIs handle all other
              # requests by passing them to the front-end controller.
              return 301 /nextcloud/index.php$request_uri;
            }

            location ^~ /nextcloud {
              # set max upload size
              client_max_body_size 10M;
              fastcgi_buffers 64 4K;

              # Enable gzip but do not remove ETag headers
              gzip on;
              gzip_vary on;
              gzip_comp_level 4;
              gzip_min_length 256;
              gzip_proxied expired no-cache no-store private no_last_modified no_etag auth;
              gzip_types application/atom+xml application/javascript application/json application/ld+json application/manifest+json application/rss+xml application/vnd.geo+json application/vnd.ms-fontobject application/x-font-ttf application/x-web-app-manifest+json application/xhtml+xml application/xml font/opentype image/bmp image/svg+xml image/x-icon text/cache-manifest text/css text/plain text/vcard text/vnd.rim.location.xloc text/vtt text/x-component text/x-cross-domain-policy;

              # Pagespeed is not supported by Nextcloud, so if your server is built
              # with the `ngx_pagespeed` module, uncomment this line to disable it.
              #pagespeed off;

              # HTTP response headers borrowed from Nextcloud `.htaccess`
              add_header Referrer-Policy                      "no-referrer"   always;
              add_header X-Content-Type-Options               "nosniff"       always;
              add_header X-Download-Options                   "noopen"        always;
              add_header X-Frame-Options                      "SAMEORIGIN"    always;
              add_header X-Permitted-Cross-Domain-Policies    "none"          always;
              add_header X-Robots-Tag                         "none"          always;
              add_header X-XSS-Protection                     "1; mode=block" always;

              # Remove X-Powered-By, which is an information leak
              fastcgi_hide_header X-Powered-By;

              # Specify how to handle directories -- specifying `/nextcloud/index.php$request_uri`
              # here as the fallback means that Nginx always exhibits the desired behaviour
              # when a client requests a path that corresponds to a directory that exists
              # on the server. In particular, if that directory contains an index.php file,
              # that file is correctly served; if it doesn't, then the request is passed to
              # the front-end controller. This consistent behaviour means that we don't need
              # to specify custom rules for certain paths (e.g. images and other assets,
              # `/updater`, `/ocm-provider`, `/ocs-provider`), and thus
              # `try_files $uri $uri/ /nextcloud/index.php$request_uri`
              # always provides the desired behaviour.
              index index.php index.html /nextcloud/index.php$request_uri;

              # Rule borrowed from `.htaccess` to handle Microsoft DAV clients
              location = /nextcloud {
                  if ( $http_user_agent ~ ^DavClnt ) {
                      return 302 /nextcloud/remote.php/webdav/$is_args$args;
                  }
              }

              # Rules borrowed from `.htaccess` to hide certain paths from clients
              location ~ ^/nextcloud/(?:build|tests|config|lib|3rdparty|templates|data)(?:$|/)    { return 404; }
              location ~ ^/nextcloud/(?:\.|autotest|occ|issue|indie|db_|console)                  { return 404; }

              # Ensure this block, which passes PHP files to the PHP process, is above the blocks
              # which handle static assets (as seen below). If this block is not declared first,
              # then Nginx will encounter an infinite rewriting loop when it prepends
              # `/nextcloud/index.php` to the URI, resulting in a HTTP 500 error response.
              location ~ \.php(?:$|/) {
                  fastcgi_split_path_info ^(.+?\.php)(/.*)$;
                  set $path_info $fastcgi_path_info;

                  try_files $fastcgi_script_name =404;

                  include fastcgi_params;
                  fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
                  fastcgi_param PATH_INFO $path_info;
                  #fastcgi_param HTTPS on;
                  fastcgi_param HTTPS off;

                  fastcgi_param modHeadersAvailable true;         # Avoid sending the security headers twice
                  fastcgi_param front_controller_active true;     # Enable pretty urls
                  fastcgi_pass  php-handler;

                  fastcgi_intercept_errors on;
                  fastcgi_request_buffering off;
              }

              location ~ \.(?:css|js|svg|gif|png|jpg|ico)$ {
                  try_files $uri /nextcloud/index.php$request_uri;
                  expires 6M;         # Cache-Control policy borrowed from `.htaccess`
                  access_log off;     # Optional: Don't log access to assets
              }

              location ~ \.woff2?$ {
                  try_files $uri /nextcloud/index.php$request_uri;
                  expires 7d;         # Cache-Control policy borrowed from `.htaccess`
                  access_log off;     # Optional: Don't log access to assets
              }

              # Rule borrowed from `.htaccess`
              location /nextcloud/remote {
                  return 301 /nextcloud/remote.php$request_uri;
              }

              location /nextcloud {
                  try_files $uri $uri/ /nextcloud/index.php$request_uri;
              }
            }

            # redirect server error pages to the static page /50x.html
            #
            error_page   500 502 503 504  /50x.html;
            location = /50x.html {
                root   /usr/share/nginx/html;
            }
        }
    }

## php

Install php, enable the required modules and enable caching:

    # pacman -Syu php php-apcu php-gd php-imap php-intl php-pgsql php-sqlite

    $ snip /etc/php/php.ini
    memory_limit = 1024M
    extension=bz2
    extension=gd
    extension=gmp
    extension=imap
    extension=intl
    extension=pdo_pgsql
    extension=pgsql

    $ cat /etc/php/conf.d/apcu.ini
    extension=apcu.so
    apc.ttl=7200
    apc.enable_cli=1

Opcache is enabled by default.

## php-fpm

This is a daemon that takes requests from nginx and executes them with php. Install and configure:

    # pacman -Syu php-fpm

    $ snip /etc/php/php-fpm.conf
    systemd_interval = 1h

    $ cat /etc/php/php-fpm.d/nextcloud.conf
    [nextcloud]
    user = nextcloud
    group = nextcloud
    listen = /run/nextcloud/nextcloud.sock
    env[PATH] = /usr/local/bin:/usr/bin:/bin
    env[TMP] = /tmp

    ; should be accessible by your web server
    listen.owner = http
    listen.group = http

    pm = dynamic
    pm.max_children = 15
    pm.start_servers = 2
    pm.min_spare_servers = 1
    pm.max_spare_servers = 3

    $ cat /etc/systemd/system/php-fpm.service.d/override.conf
    [Service]
    ReadWritePaths=/var/lib/nextcloud/data

## nextcloud

Install the php files, setup the database:

    # pacman -Syu nextcloud
    # /usr/bin/occ maintenance:install --database pgsql --database-name nextcloud --database-host /run/postgresql --database-user nextcloud --database-pass nextcloudpw --data-dir /var/lib/nextcloud/data -vvv

Enable APCu:

    $ snip /usr/share/webapps/nextcloud/config/config.php
    'memcache.local' => '\OC\Memcache\APCu',

Start the required services and check their status:

    # systemctl start postgresql nginx php-fpm
    # systemctl status postgresql nginx php-fpm

The nextcloud web UI should be accessible at http://<your_hostname>/nextcloud.

## Appendix

To see the difference between a current file and the package maintainers original version,
use ([source][diff-file]):

    function qkkdiff-file {
      if [[ "$1" == "" ]];
      then
          echo "Example usage: qkkdiff-file /etc/pacman.conf"
      elif [[ -f "$1" ]];
      then
          pkg="$(pacman -Qo $1 | awk '//{printf "%s-%s", $(NF-1), $NF;}')"
          bsdtar -xOf /var/cache/pacman/pkg/${pkg}-$(uname -m).pkg.tar.xz "${1/\//}" | diff - "$1"
          return 0
      else
          echo "The provided file ${1} does not exist."
          return 1
      fi
    }

[Nextcloud]: https://nextcloud.com/
[arch-next]: https://wiki.archlinux.org/title/Nextcloud
[nextcloud nginx docs]: https://docs.nextcloud.com/server/latest/admin_manual/installation/nginx.html
[diff-file]: https://bbs.archlinux.org/viewtopic.php?id=177570
