---
title: Setup Supervisor and Python on Centos
date: 2016-10-24 15:40:57
categories:
- study
tags:
- Linux
- Python
- Supervisor
---

Supervisor is a client/server system that allows its users to control a number of processes on UNIX-like operating systems.

Official introduction: http://supervisord.org/introduction.html

<!--more-->

## Install python

Reference: [Python Source Releases](https://www.python.org/downloads/source/)

Version 2.7

## Install pip

Reference: [pip Installation](https://pip.pypa.io/en/latest/installing/)

```
wget 'https://bootstrap.pypa.io/get-pip.py'
sudo python get-pip.py
```

## Install Supervisor

Reference: [Supervisor Installing](http://supervisord.org/installing.html)

```
sudo pip install supervisor
```

## Config Supervisor

### Create default configure file

```
sudo mkdir -p /etc/supervisor/conf.d
echo_supervisord_conf > supervisord.conf
sudo cp  supervisord.conf /etc/supervisor/supervisord.conf
```

Change `supervisord.conf` `[include]` to :

```
[include]
files = /etc/supervisor/conf.d/*.conf
```

### Add worker configure file

Example `/etc/supervisor/conf.d/example.conf`

```
[program:program name]
process_name=%(program_name)s_%(process_num)02d
command=whoami
autostart=true
autorestart=true
startsecs=0

#configure thread number as needed
numprocs=4
redirect_stderr=true

#configure path as needed
stdout_logfile=/var/log/example.log
```

## Start Supervisor

Start supervisord
`supervisord -c /etc/supervisor/supervisord.conf`

When configure file have updated

```
supervisorctl reread
supervisorctl update
```

Basic operation

```
supervisorctl status

# start/stop a program
supervisorctl start [program name]
supervisorctl stop [program name]

# start/stop all programs
supervisorctl start all
supervisorctl stop all

```
## Running supervisord Automatically on Startup

>Reference:
>1. http://stackoverflow.com/questions/31157928/supervisord-on-linux-centos-7-only-works-when-run-with-root
>2. https://rayed.com/wordpress/?p=1496

### Setup init script.

Create `/etc/rc.d/init.d/supervisord` with the following content, configure line 19 [prefix] and line 20 [config_path] as needed (I don't know who's the owner of this script, but thanks all the same):

```
#!/bin/sh
#
# /etc/rc.d/init.d/supervisord
#
# Supervisor is a client/server system that
# allows its users to monitor and control a
# number of processes on UNIX-like operating
# systems.
#
# chkconfig: - 64 36
# description: Supervisor Server
# processname: supervisord

# Source init functions
. /etc/rc.d/init.d/functions

prog="supervisord"

prefix="/usr/local/"
exec_prefix="${prefix}"
config_path="/etc/supervisord.conf"
prog_bin="${exec_prefix}/bin/supervisord -c ${config_path}"
PIDFILE="/var/run/$prog.pid"

start()
{
       echo -n $"Starting $prog: "
       daemon $prog_bin --pidfile $PIDFILE
       sleep 1
       [ -f $PIDFILE ] && success $"$prog startup" || failure $"$prog startup"
       echo
}

stop()
{
       echo -n $"Shutting down $prog: "
       [ -f $PIDFILE ] && sleep 1 && killproc $prog || success $"$prog shutdown"
       echo
}

case "$1" in

 start)
   start
 ;;

 stop)
   stop
 ;;

 status)
       status $prog
 ;;

 restart)
   stop
   start
 ;;

 *)
   echo "Usage: $0 {start|stop|restart|status}"
 ;;

esac
```

Make the script executable and register it as a service.

```
sudo chmod +x /etc/rc.d/init.d/supervisord
sudo chkconfig --add supervisord
sudo chkconfig supervisord on

# Start the service
sudo service supervisord start

# Stop the service
sudo service supervisord stop

# Restart the service
sudo service supervisord restart

# Check service status
sudo service supervisord status
```