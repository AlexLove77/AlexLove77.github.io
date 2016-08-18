---
title: Setup LAMP on CentOS 7.2
date: 2016-08-18 11:36:57
categories:
- study
tags:
- Linux
---

**Environment**

name   | version | remark
----   | ------- | ------
OS     | CentOS 7.2 |
repo   | [Remi](http://rpms.famillecollet.com/) |
Apache | 2.4  |
PHP    | 7.0  | repo: remi-php70
Mysql  | 5.6  |
Java   | JDK8 |

<!--more-->

### Add Remi's RPM Repository

```bash
# if there is no epel repo
wget https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
sudo rpm -Uvh epel-release-latest-7*.rpm

# remi repo
wget http://rpms.famillecollet.com/enterprise/remi-release-7.rpm
sudo rpm -Uvh remi-release-7.rpm
```
### Install Apache (httpd)

```bash
sudo yum install httpd

sudo systemctl start httpd.service		# or sudo service httpd start
httpd -version
Server version: Apache/2.4.6 (CentOS)
Server built:   Jul 18 2016 15:30:14
```

### Install MySQL

```bash
# add mysql yum repo
wget http://dev.mysql.com/get/mysql57-community-release-el7-8.noarch.rpm
sudo rpm -Uvh mysql57-community-release-el7-8.noarch.rpm

# enable 5.6 and disable 5.7
sudo vi /etc/yum.repos.d/mysql-community.repo

# [1] if using cloud RDS, just install client
sudo yum install mysql

# [2] install mysql server
sudo yum install mysql-community-server

sudo systemctl start mysql.service		# or sudo service mysql start
```

```
mysql -V
mysql  Ver 14.14 Distrib 5.6.32, for Linux (x86_64) using  EditLine wrapper
```

### Install PHP7 and Modules

```bash
sudo yum --enablerepo=remi-php70 install php php-mbstring php-xml php-pdo php-mysql

php -v
PHP 7.0.9 (cli) (built: Jul 20 2016 17:58:41) ( NTS )
Copyright (c) 1997-2016 The PHP Group
Zend Engine v3.0.0, Copyright (c) 1998-2016 Zend Technologies
```

### Install Composer

Composer official site: https://getcomposer.org

Manually install

```bash
# download latest snapshot
wget https://getcomposer.org/composer.phar

# move to /usr/bin/ and make it executable
sudo mv composer.phar /usr/bin/composer
sudo chmod a+x /usr/bin/composer

# install zip and unzip
sudo yum install zip unzip
```

### Install Java8

According to the official guide, [Java (JVM) version](https://www.elastic.co/guide/en/elasticsearch/reference/current/setup.html#jvm-version)

>Only Oracle’s Java and the OpenJDK are supported. The same JVM version should be used on all Elasticsearch nodes and clients.

>We recommend installing the Java 8 update 20 or later, or Java 7 update 55 or later. Previous versions of Java 7 are known to have bugs that can cause index corruption and data loss. Elasticsearch will refuse to start if a known-bad version of Java is used.

>The version of Java to use can be configured by setting the JAVA_HOME environment variable.

```bash
# oracle jdk 8u102

# Note: If you would like to install a different release of Oracle Java 8 JDK,
# go to the Oracle Java 8 JDK Downloads Page, accept the license agreement,
# and copy the download link of the appropriate Linux .rpm package.
# Substitute the copied download link in place of the highlighted part of the wget command.

# Change to your home directory and download the Oracle Java 8 JDK RPM with these commands:
wget --no-cookies --no-check-certificate --header "Cookie: gpw_e24=http%3A%2F%2Fwww.oracle.com%2F; oraclelicense=accept-securebackup-cookie" http://download.oracle.com/otn-pub/java/jdk/8u102-b14/jdk-8u102-linux-x64.rpm

sudo rpm -Uvh jdk-8u102-linux-x64.rpm

java -version
java version "1.8.0_102"
Java(TM) SE Runtime Environment (build 1.8.0_102-b14)
Java HotSpot(TM) 64-Bit Server VM (build 25.102-b14, mixed mode)


# OpenJDK 1.8
sudo yum install java-1.8.0-openjdk

java -version
openjdk version "1.8.0_101"
OpenJDK Runtime Environment (build 1.8.0_101-b13)
OpenJDK 64-Bit Server VM (build 25.101-b13, mixed mode)
```

### Other Configurations

#### MySQL Import Tuning

At the beginning, login mysql bash, then run following commands:

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

```bash
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

#### Apache Configuration

**Apache modules**
Apache modules: mod_rewrite, mod_deflate, mod_filter

**Google PageSpeed Module (optional)**
Official site: https://developers.google.com/speed/pagespeed/module/

#### Apache Listen to Not Standard Port

>We may want a service such as Apache to be allowed to bind and listen for incoming connections on a non-standard port. By default, the SELinux policy will only allow services access to recognized ports associated with those services.

>A full list of ports that services are permitted access by SELinux can be obtained with:

```bash
sudo semanage port -l

# grep http ports
sudo semanage port -l | grep http
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
```

>If we wanted to allow Apache to listen on tcp port 82, we can add a rule to allow that using the 'semanage' command:

```
sudo semanage port -a -t http_port_t -p tcp 82
```

