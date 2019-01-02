---
title: Setup LNMP on Ubuntu 18.04
date: 2018-11-22 11:36:57
categories:
- study
tags:
- Linux
---

**Environment**

name   | version | remark
----   | ------- | ------
OS     | Ubuntu 18.04 | Bionic Beaver
Nginx  | 1.14.0  |
PHP    | 7.2     |
Mysql  | 8.0     | Client

<!--more-->


### Install Nginx

```
sudo apt install nginx

sudo systemctl start nginx.service		# or sudo service nginx start
nginx -v
nginx version: nginx/1.14.0 (Ubuntu)
```

### Install MySQL

```
# add mysql apt repo
# https://dev.mysql.com/downloads/repo/apt/

wget https://dev.mysql.com/get/mysql-apt-config_0.8.13-1_all.deb
sudo dpkg -i mysql-apt-config_0.8.13-1_all.deb

# Just install client
sudo apt install mysql-client

# Install server
sudo apt install mysql-server

mysql -V
mysql  Ver 8.0.13 for Linux on x86_64 (MySQL Community Server - GPL)
```

### Install PHP7 and Modules

```
sudo apt install php7.2 php7.2-mbstring php7.2-xml php7.2-fpm php7.2-mysql php7.2-curl

php -v
PHP 7.2.10-0ubuntu0.18.04.1 (cli) (built: Sep 13 2018 13:45:02) ( NTS )
Copyright (c) 1997-2018 The PHP Group
Zend Engine v3.2.0, Copyright (c) 1998-2018 Zend Technologies
    with Zend OPcache v7.2.10-0ubuntu0.18.04.1, Copyright (c) 1999-2018, by Zend Technologies
```

### Install Composer

Composer official site: https://getcomposer.org

Manually install

```
# download latest snapshot
wget https://getcomposer.org/composer.phar

# move to /usr/bin/ and make it executable
sudo mv composer.phar /usr/bin/composer
sudo chmod a+x /usr/bin/composer

# install zip and unzip
sudo apt install zip unzip
```

### Other Configurations

#### MySQL Import Tuning

At the beginning, login mysql , then run following commands:

```sql
SET autocommit=0;
SET unique_checks=0;
SET foreign_key_checks=0;
```
then import sql files, and at the end

```sql
COMMIT;
SET unique_checks=1;
SET foreign_key_checks=1;
```

#### SELinux Settings

As centos enable SELinux as default, and SELinux is preventing httpd from read access on the file and access db, we need to allow them.

**Note**: if cloud server disable the SELinux or allow relevant httpd services, the following commands doesn't need, so try to see SELinux status first.

```
# show se status
sestatus

# show selinux httpd config
getsebool -a | grep httpd

# allow httpd read user content
sudo setsebool -P httpd_read_user_content 1

# allow read & write of storage/
chcon -R -t httpd_sys_rw_content_t storage

# Allow HTTPD scripts and modules to network connect to databases.
sudo setsebool -P httpd_can_network_connect_db=1

# Allow HTTPD scripts and modules to connect to the network.
# elasticsearch & post check
sudo setsebool -P httpd_can_network_connect=1
```

#### Nginx Configuration Example
```
server {
    listen 80;

    server_name example.com;

    access_log /var/log/YOUR_PROJECT/access.log;
    error_log /var/log/YOUR_PROJECT/error.log;

    root /var/www/YOUR_PROJECT/public;
    index index.php index.html;

    # serve static files directly
    location ~* \.(jpg|jpeg|gif|css|png|js|ico|html)$ {
        access_log off;
        expires max;
        log_not_found off;
    }

    # removes trailing slashes (prevents SEO duplicate content issues)
    if (!-d $request_filename)
    {
        rewrite ^/(.+)/$ /$1 permanent;
    }

    # enforce NO www
    # if ($host ~* ^www\.(.*))
    # {
    #     set $host_without_www $1;
    #     rewrite ^/(.*)$ $scheme://$host_without_www/$1 permanent;
    # }

    # unless the request is for a valid file (image, js, css, etc.), send to bootstrap
    if (!-e $request_filename)
    {
        rewrite ^/(.*)$ /index.php?/$1 last;
        break;
    }

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~* \.php$ {
        try_files $uri = 404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        #fastcgi_pass 127.0.0.1:9000;
        fastcgi_pass unix:/var/run/php/php7.2-fpm.sock; # may also be: 127.0.0.1:9000;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.ht {
        deny all;
    }
}
```

**Google PageSpeed Module (optional)**
Official site: https://developers.google.com/speed/pagespeed/module/

#### Firewall Configuration

**UFW**: https://help.ubuntu.com/community/UFW

>UFW - Uncomplicated Firewall
The default firewall configuration tool for Ubuntu is ufw. Developed to ease iptables firewall configuration, ufw provides a user friendly way to create an IPv4 or IPv6 host-based firewall. By default UFW is >disabled.

```shell
sudo ufw status
sudo ufw enable
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw allow from 192.168.1.0/24
sudo ufw allow http
sudo ufw allow https
sudo ufw status numbered
```