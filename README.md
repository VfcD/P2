# Nextcloud 12, Ubuntu and LEMP Tutorial
##### flavoured by: https://www.linuxbabe.com/ubuntu/install-nextcloud-11-ubuntu-16-04-nginx-config

## against password acrobatics
	sudo su
## or
  su


# prepare MariaDB
## create a database for Nextcloud and edit 'strong_password'
	mysql -uroot -p
	CREATE DATABASE nextcloud;
	GRANT ALL PRIVILEGES ON nextcloud.* TO 'nextcloud'@'localhost' IDENTIFIED BY 'strong_password';
	FLUSH PRIVILEGES;
	\q

## enable binary loggin in MariaDB
	nano /etc/mysql/mariadb.conf.d/50-server.cnf

## add following block in [mysql] section
	log-bin        = /var/log/mysql/mariadb-bin
	log-bin-index  = /var/log/mysql/mariadb-bin.index
	binlog_format  = mixed

## reload mysql
	systemctl reload mysql


# prepare fpm
## uncomment following environment variables from /etc/php/7.0/fpm/pool.d/www.conf
	nano /etc/php/7.0/fpm/pool.d/www.conf
	env[HOSTNAME] = $HOSTNAME
	env[PATH] = /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
	env[TMP] = /tmp
	env[TMPDIR] = /tmp
	env[TEMP] = /tmp

## restart fpm
	service php7.0-fpm restart

## download nextcloud
	wget https://download.nextcloud.com/server/releases/latest.zip

## unzip it and move it to nginx landing
	unzip latest.zip -d /usr/share/nginx/

## make www-data owner for the nextcloud dir
	chown -R www-data:www-data  /usr/share/nginx/

## create nextcloud.conf in /etc/nginx/conf.d/ dir
	nano /etc/nginx/conf.d/nextcloud.conf

## add following blockedit and edit server_name,

        server {
            listen 80;
            server_name <IP> OR DOMAIN <IP>;

            # Add headers to serve security related headers
            add_header X-Content-Type-Options nosniff;
           #uncomment following line. looks like its now standard in nginx
           # add_header X-Frame-Options "SAMEORIGIN";
            add_header X-XSS-Protection "1; mode=block";
            add_header X-Robots-Tag none;
            add_header X-Download-Options noopen;
            add_header X-Permitted-Cross-Domain-Policies none;

            # Path to the root of your installation
            root /usr/share/nginx/nextcloud/;

            location = /robots.txt {
                allow all;
                log_not_found off;
                access_log off;
            }

            # The following 2 rules are only needed for the user_webfinger app.
            # Uncomment it if you're planning to use this app.
            #rewrite ^/.well-known/host-meta /public.php?service=host-meta last;
            #rewrite ^/.well-known/host-meta.json /public.php?service=host-meta-json
            # last;

            location = /.well-known/carddav {
                return 301 $scheme://$host/remote.php/dav;
            }
            location = /.well-known/caldav {
               return 301 $scheme://$host/remote.php/dav;
            }

            location ~ /.well-known/acme-challenge {
              allow all;
            }

            # set max upload size
            client_max_body_size 512M;
            fastcgi_buffers 64 4K;

            # Disable gzip to avoid the removal of the ETag header
            gzip off;

            # Uncomment if your server is build with the ngx_pagespeed module
            # This module is currently not supported.
            #pagespeed off;

            error_page 403 /core/templates/403.php;
            error_page 404 /core/templates/404.php;

            location / {
               rewrite ^ /index.php$uri;
            }

            location ~ ^/(?:build|tests|config|lib|3rdparty|templates|data)/ {
               deny all;
            }
            location ~ ^/(?:\.|autotest|occ|issue|indie|db_|console) {
               deny all;
             }

            location ~ ^/(?:index|remote|public|cron|core/ajax/update|status|ocs/v[12]|updater/.+|ocs-provider/.+|core/templates/40[34])\.php(?:$|/) {
               include fastcgi_params;
               fastcgi_split_path_info ^(.+\.php)(/.*)$;
               fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
               fastcgi_param PATH_INFO $fastcgi_path_info;
               #Avoid sending the security headers twice
               fastcgi_param modHeadersAvailable true;
               fastcgi_param front_controller_active true;
               fastcgi_pass unix:/run/php/php7.0-fpm.sock;
               fastcgi_intercept_errors on;
               fastcgi_request_buffering off;
            }

            location ~ ^/(?:updater|ocs-provider)(?:$|/) {
               try_files $uri/ =404;
               index index.php;
            }

            # Adding the cache control header for js and css files
            # Make sure it is BELOW the PHP block
            location ~* \.(?:css|js)$ {
                try_files $uri /index.php$uri$is_args$args;
                add_header Cache-Control "public, max-age=7200";
                # Add headers to serve security related headers (It is intended to
                # have those duplicated to the ones above)
                add_header X-Content-Type-Options nosniff;
                add_header X-Frame-Options "SAMEORIGIN";
                add_header X-XSS-Protection "1; mode=block";
                add_header X-Robots-Tag none;
                add_header X-Download-Options noopen;
                add_header X-Permitted-Cross-Domain-Policies none;
                # Optional: Don't log access to assets
                access_log off;
           }

           location ~* \.(?:svg|gif|png|html|ttf|woff|ico|jpg|jpeg)$ {
                try_files $uri /index.php$uri$is_args$args;
                # Optional: Don't log access to other assets
                access_log off;
           }
        }


## check if you did good
	nginx -t
	systemctl reload nginx

## remove dir /etc/nginx/sites-enabled to avoid confict with nextcloud landing page
	rm -r /etc/nginx/sites-enabled

## remove default.conf in /etc/nginx/conf.d - u just keep the nextcloud.conf
	rm /etc/nginx/conf.d/default.conf

## again, check if you did good
	nginx -t
	systemctl reload nginx


# install php modules for Nextcloud
	apt install php7.0-zip php7.0-intl php7.0-mcrypt php-imagick

# enable caching
##### flavoured by https://bayton.org/docs/nextcloud/installing-nextcloud-on-ubuntu-16-04-lts-with-redis-apcu-ssl-apache/
	apt install php-apcu redis-server php-redis

## restart fpm
	service php7.0-fpm restart

## open redis conf
	nano /etc/redis/redis.conf

## change port
	port 6379 -> port 0

## uncomment and edit permissions
	unixsocket /var/run/redis/redis.sock
	unixsocketperm 770

## add the Redis user redis to the www-data group
	sudo usermod -a -G redis www-data

now you can visit your nextcloud landing page via browser at `<IP> OR DOMAIN <IP>`

## create admin and pw for nextcloud, log in with MariaDB Nextcloud credentials we created before:

	admin account
	alf_admin1337
	really_strong_password

	Data folder
	/usr/share/nginx/nextcloud/data/

	Configure the database
	nextcloud
	strong_password
	nextcloud
	localhost

## I got a timeout here. 504 bad gateway. reload page. reload the page, log in and wait...

## edit Nextcloud config file
	nano /usr/share/nginx/nextcloud/config/config.php

## paste the following block at the end of file - normally below 'installed' => true,

	'memcache.local' => '\OC\Memcache\APCu',
	'memcache.locking' => '\\OC\\Memcache\\Redis',
	'filelocking.enabled' => 'true',
	'redis' =>
	array (
		'host' => '/var/run/redis/redis.sock',
		'port' => 0,
		'timeout' => 0.0,
	),

## optimize /etc/php/7.0/fpm/php.ini
	nano /etc/php/7.0/fpm/php.ini

## find vars, uncomment and change to
	opcache.enable=1
	opcache.enable_cli=1
	opcache.memory_consumption=128
	opcache.interned_strings_buffer=8
	opcache.max_accelerated_files=10000
	opcache.revalidate_freq=1
	opcache.save_comments=1

## restart fpm
	systemctl reload php7.0-fpm

#use Cron instead Ajax

##### flavoured by https://docs.nextcloud.com/server/12/admin_manual/configuration_server/background_jobs_configuration.html

## check radiobutton on Nextcloud admin page Cron

## write a crontab like
	crontab -u www-data -e

## chose editor and add at the end of file
```
	*/15  *  *  *  * php -f /usr/share/nginx/nextcloud/cron.php
```

## verify if you did good
	crontab -u www-data -l

## Note: On some systems it might be required to call php-cli instead of php


# extern HDD connection
## prepare mounting point
	mkdir /home/Nextcloud_extern

## check usb devices
	lsblk

## select usb device, normally sda1
	mount -t auto /dev/sda1 /home/Nextcloud_extern

## giving priveleges for nginx
	chown -R www-data:www-data /home/Nextcloud_extern

## creating automount
	nano /etc/fstab
	/dev/sda1 /home/Nextcloud_extern auto noatime 0 0

## install smbclient
	apt install smbclient
	systemctl reload php7.0-fpm
	systemctl reload nginx

## settings // Apps // deavtivated Apps
	activate  External storage support

## go to admin panel
	External storage


# nextcloud logging file
	nano /usr/share/nginx/nextcloud/data/nextcloud.log

> reach me via derbarti gmail com
