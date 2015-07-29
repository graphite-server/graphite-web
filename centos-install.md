Install dependencies
--------------------

-   Install development tools

``` bash
yum -v groupinstall "Development Tools"
```

-   Install python 2.7

``` bash
yum -v install bzip2-devel db4-devel gdbm-devel libpcap-devel ncurses-devel openssl-devel sqlite-devel readline-devel tk-devel xz-devel zlib-devel
mkdir -p ~/software/python
cd ~/software/python
wget https://www.python.org/ftp/python/2.7.10/Python-2.7.10.tgz
tar xvfz Python-2.7.10.tgz 
cd Python-2.7.10
./configure  --enable-shared 
make -j 4
make install
# Create a file name /etc/ld.so.conf.d/usrlocal.conf
vi /etc/ld.so.conf.d/usrlocal.conf # Add the following entry /usr/local/lib
ldconfig -v
```

-   Basic dependencies using yum

``` bash
yum -v install bitmap bitmap-fonts git libffi-devel httpd-devel memcached openldap-devel
```

-   Install pip

``` bash
curl https://raw.githubusercontent.com/pypa/pip/master/contrib/get-pip.py | python2.7 -
```

-   Intall mysqlclient

``` bash
ln -s /usr/lib64/libmysqlclient_r.so.16 /usr/lib64/libmysqlclient_r.so
pip install mysqlclient
```

-   Install [django](https://www.djangoproject.com/) and [django-tagging](http://code.google.com/p/django-tagging/)

``` bash
pip install cairocffi django-auth-ldap mod_wsgi pyparsing python-ldap python-memcached twisted pytz
pip install Django django-tagging
```

Install graphite
----------------

-   Install graphite server after cloning the code from github:

``` bash
git clone https://github.com/graphite-server/carbon.git
git clone https://github.com/graphite-server/ceres.git
git clone https://github.com/graphite-server/whisper.git
git clone https://github.com/graphite-server/graphite-web
```

Setup graphite
--------------

**Note:** The graphite webapp is written in python using django.

-   Creating a db user for django by running mysql client and execute:

``` sql
create database django character set utf8;
create user 'djangouser'@'localhost' identified by 'some_password';
grant all privileges on django.* to  'djangouser'@'localhost';
flush privileges;
```

-   Setup mysql to run with django.

``` bash
cat  > /opt/graphite/webapp/graphite/my.cnf << EOF
[client]
database = django
user = djangouser
password = some_password
default-character-set = utf8
EOF
```

-   Enable by copy the webapp local\_settings and manage.py:

``` bash
cp /opt/graphite/webapp/graphite/local_settings.py.example /opt/graphite/webapp/graphite/local_settings.py
cp {location of graphite source code}/graphite-web/webapp/manage.py /opt/graphite/webapp
```

-   Edit *local\_settings.py* and change the following option (this the current setting for a graphite cluster with four servers)

``` python
SECRET_KEY = 'Some random string'
....
TIME_ZONE = 'America/New_York'
....
MEMCACHE_HOSTS = ['127.0.0.1:11211']
....
## LDAP using django-auth-ldap
import ldap
from django_auth_ldap.config import LDAPSearch,LDAPSearchUnion
USE_LDAP_AUTH = True
AUTH_LDAP_SERVER_URI = "ldap://ldap.company.com:389"
AUTH_LDAP_BIND_DN = ""
AUTH_LDAP_BIND_PASSWORD = ""
AUTH_LDAP_USER_SEARCH = LDAPSearchUnion(
        LDAPSearch("ou=ah,dc=ad,dc=company,dc=com", ldap.SCOPE_SUBTREE, "(uid=%(user)s)"),
        LDAPSearch("ou=mail,dc=ad,dc=company,dc=com", ldap.SCOPE_SUBTREE, "(uid=%(user)s)")
)
....
DASHBOARD_REQUIRE_AUTHENTICATION = True
DASHBOARD_REQUIRE_PERMISSIONS = True
....
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'OPTIONS': {
                'read_default_file': '/opt/graphite/webapp/graphite/my.cnf'
        }
    }
}
...
```

-   Edit */opt/graphite/webapp/graphite/local\_settings.py* and fill the variable SECRET\_KEY with some random hash. This needs to be unique and secret for each graphite server.

<!-- -->

-   Sync the database configuration (this need to be done only once)

``` bash
python /opt/graphite/webapp/manage.py syncdb
```

-   Collect all the static content from django contributions.

``` bash
python /opt/graphite/webapp/manage.py collectstatic
```

-   Setup memcached and start the service

``` bash
cat  > /etc/sysconfig/memcached << EOF
PORT="11211"
USER="memcached"
MAXCONN="1024"
CACHESIZE="64"
OPTIONS="-l 127.0.0.1"
EOF
chkconfig memcached on
service memcached start
```

-   Copy the example file as the actual conf files

``` bash
cp /opt/graphite/conf/carbon.conf.example /opt/graphite/conf/carbon.conf
cp /opt/graphite/conf/storage-schemas.conf.example /opt/graphite/conf/storage-schemas.conf
cp /opt/graphite/conf/aggregation-rules.conf.example /opt/graphite/conf/aggregation-rules.conf
cp /opt/graphite/conf/storage-aggregation.conf.example /opt/graphite/conf/storage-aggregation.conf
```

-   Edit *storage-schemas.conf* and set the retention rules for each metric

``` bash
[carbon]
pattern = ^carbon\.
retentions = 60:90d

[collectl]
pattern = ^collectl\.
retentions = 60:90d

[default]
pattern = .*
retentions = 1m:24h,5m:60h,30m:15d,120m:62d,1440m:762d
```

-   Edit *carbon.conf* and change the following parameters (you can read the definition of the parameters in the config file)

``` bash
...
MAX_UPDATES_PER_SECOND_ON_SHUTDOWN = 2000
...
MAX_CREATES_PER_MINUTE = inf
```

-   Set new limits for current and future sessions:

``` bash
# Check if the number is large (suggestion 65535)
ulimit -n
# If then add line to /etc/security/limits.conf (this is after reboot or next session)
..
root             -       nofile          65535
..
# Run the following to increase the limit for the current session
ulimit -n 65535
```

-   Set the right permit ownership for storage directory

``` bash
chown -R apache:apache /opt/graphite/storage
```

-   Test mod\_wsgi express (service at localhost:8000)

``` bash
python manage.py runmodwsgi --user apache --group apache
```

Install and setup supervisor
----------------------------

-   Install supervisord

``` bash
pip install supervisor
```

-   Create a init.d script for supervisord

``` bash
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

prefix="/usr/local"
exec_prefix="${prefix}"
prog_bin="${exec_prefix}/bin/supervisord -c /etc/supervisord.conf"
PIDFILE="/var/run/$prog.pid"

export PATH="/usr/local/bin:$PATH"

start()
{
        echo -n $"Starting $prog: "
        daemon $prog_bin --pidfile $PIDFILE
        [ -f $PIDFILE ] && success $"$prog startup" || failure $"$prog startup"
        echo
}

stop()
{
        echo -n $"Shutting down $prog: "
        [ -f $PIDFILE ] && killproc $prog || success $"$prog shutdown"
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

-   Create a configuration file for supervisord

``` bash
cat > /etc/supervisord.conf << EOF
; Supervisor configuration for carbon-cache.
 
[unix_http_server]
file=/tmp/supervisor.sock   ; (the path to the socket file)
 
[supervisord]
logfile=/tmp/supervisord.log ; (main log file;default $CWD/supervisord.log)
logfile_maxbytes=50MB        ; (max main logfile bytes b4 rotation;default 50MB)
logfile_backups=10           ; (num of main logfile rotation backups;default 10)
loglevel=info                ; (log level;default info; others: debug,warn,trace)
pidfile=/tmp/supervisord.pid ; (supervisord pidfile;default supervisord.pid)
nodaemon=false               ; (start in foreground if true;default false)
minfds=1024                  ; (min. avail startup file descriptors;default 1024)
minprocs=200                 ; (min. avail process descriptors;default 200)
 
[rpcinterface:supervisor]
supervisor.rpcinterface_factory = supervisor.rpcinterface:make_main_rpcinterface
 
[supervisorctl]
serverurl=unix:///tmp/supervisor.sock ; use a unix:// URL  for a unix socket
 
[program:carbon-cache]
command=/opt/graphite/bin/carbon-cache.py start --debug
autostart=true
autorestart=true
redirect_stderr=true
stdout_logfile=/opt/graphite/storage/log/carbon/cache.log
stdout_logfile_maxbytes=1MB

[program:graphite-webapp]
command=python /opt/graphite/webapp/manage.py runmodwsgi --user apache --group apache --port=80 --access-log --log-to-terminal --server-root=/opt/graphite/storage/modwsgi
autostart=true
autorestart=true
redirect_stderr=true
stdout_logfile=/opt/graphite/storage/log/webapp/modwsgi.log
stdout_logfile_maxbytes=1MB
EOF
```

-   Start the service

``` bash
chmod +x /etc/init.d/supervisord
service supervisord start
chkconfig supervisord on
supervisorctl status
```

Install and setup collectl monitors
-----------------------------------

-   Install from source

``` bash
wget http://sourceforge.net/projects/collectl/files/latest/download?source=files
tar xvfz collectl-X.Y.Z.src.tar.gz
cd collectl-X.Y.Z
./INSTALL
```

-   Configure '/etc/collectl.conf' and start service

``` bash
chmod u+w /etc/collectl.conf
edit /etc/collectl.conf
...
DaemonCommands = -i 60 -s cDmn --export graphite,{hostname},b=collectl.
...
```

-   Start collectl service

``` bash
service collectl start
chkconfig collectl on
```
