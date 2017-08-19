# PHP - install from source
[**Reference:** *https://blacksaildivision.com/php-install-from-source*](https://blacksaildivision.com/php-install-from-source)

#### Repositories
***CentOS/RHEL 7.x***
```ssh
rpm -Uvh https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
rpm -Uvh https://mirror.webtatic.com/yum/el7/webtatic-release.rpm
```

***CentOS/RHEL 6.x***
```sh
rpm -Uvh https://dl.fedoraproject.org/pub/epel/epel-release-latest-6.noarch.rpm
rpm -Uvh https://mirror.webtatic.com/yum/el6/latest.rpm
```

#### Prerequirements
First we need to install some libraries that are necessary to install PHP:
```sh
yum install bzip2-devel curl-devel libjpeg-devel libpng-devel freetype-devel libc-client-devel.i686 libc-client-devel libmcrypt-devel libxml2-devel openssl-devel -y
```

#### Download and unpack sources
Go to php.net download website and pick latest version of PHP. In our case it's 7.0.22, but this tutorial should work for any higher version.
```sh
cd ~/sources
wget -O php-7.0.22.tar.gz http://jp2.php.net/get/php-7.0.22.tar.gz/from/this/mirror
tar -zxvf php-7.0.22.tar.gz
cd php-7.0.22
```

#### Compile PHP from source
Now it's probably the hardest part of compiling PHP. You must provide the ./configure options and choose which modules do You want to install. For lots of needs the commands below will be sufficient, but if You need any particular library I suggest to check the PHP extensions list and find out installation options.

Commands below will enable required and basic extensions like curl, ftp, GD, IMAP, MySQLi, PDO, etc. Two important things for this tutorial are --enable-opcache and --enable-fpm. We will use PHP OPCache that comes with newer versions of PHP and will use FPM instead of Apache mod_php.
```sh
./configure --enable-bcmath --with-bz2 --enable-calendar --with-curl --enable-exif --enable-ftp --with-gd --with-jpeg-dir --with-png-dir --with-freetype-dir --enable-gd-native-ttf --with-imap --with-imap-ssl --with-kerberos --enable-mbstring --with-mcrypt --with-mhash --with-mysqli --with-openssl --with-pcre-regex --with-pdo-mysql --with-zlib-dir --enable-sysvsem --enable-sysvshm --enable-sysvmsg --enable-soap --enable-sockets --with-xmlrpc --enable-zip --with-zlib --enable-inline-optimization --enable-mbregex --enable-opcache --enable-fpm --with-libdir=lib64 --with-gettext --prefix=/usr/local/php
make
make install
```
Notice, that it might take lot of time. Much longer than Apache compilation. After make install You should find PHP installed in /usr/local/php directory.

## PHP Configuration
#### PHP-FPM setup
Before we will be able to run PHP from Apache we need to setup PHP-FPM worker. After installation there should be PHP-FPM default configuration file in installation directory. We will alter the file and then change it a bit.
```sh
cd /usr/local/php/etc
mkdir fpm.d
cp php-fpm.conf.default php-fpm.conf
vi php-fpm.conf
```
We need to  uncomment/change these lines:
```
include=etc/fpm.d/*.conf
pid = /var/run/php-fpm.pid
error_log = log/php-fpm.log
```
**COPY** EVERYTHING UNDER Pool Definitions TO CLIPBOARD AND REMOVE IT FROM php-fpm.conf FILE
```
;;;;;;;;;;;;;;;;;;;;
; Pool Definitions ;
;;;;;;;;;;;;;;;;;;;;
```

include=/etc/fpm.d/*.conf - by default there is one pool defined inside php-fpm.conf file. The best way to solve it is the same way as we solved Apache vhosts. We will include each pool in separate directory. In php-fpm.conf file one pool is already defined. We need to delete it from this file and put it inside fpm.d directory. We will have better control over the pools. The easiest way is just to Cut it from this file and paste it into new one.

Now let's create the file inside fpm.d for our example.com domain:
```sh
cd fpm.d
vi example.com.conf
```
PASTE TEXT FROM CLIPBOARD HERE AND CHANGE THESE LINES:
```
[www] -> [example_com] //Must be unique per file
user = apache
group = www
listen = 127.0.0.1:9000 //Port must be unique per file
catch_workers_output = yes
slowlog = /var/www/example.com/logs/php-fpm.slow.log
request_slowlog_timeout = 30s
php_flag[display_errors] = off
php_admin_value[error_log] = /var/www/example.com/logs/php-fpm.error.log
php_admin_flag[log_errors] = on
php_admin_value[memory_limit] = 64M
php_admin_value[open_basedir] = /var/www/example.com/htdocs
```

Each pool must have different name. So we need to change it from [www] to something else, for instance to domain name. It'll be easier to find the issues inside log files.

We set user and group to the same user as apache to have access to files.

Port will be different per pool. Standard way is to start from port 9000. Next will be 9001 etc.

We will catch errors and log them to file. In addition we set logging for  slow requests.

Nice part is that we can overwrite the settings from php.ini here. So we can overwrite error_log or memory_limit for instance. We should also set open_basedir so PHP will have access only to files inside our htdocs directory. Our server will be more secure with this setting.

#### php.ini and OPCache configuration
Second thing is php.ini file. After installation  php.ini file should located in /usr/local/php/lib. This is only the location. After compiling from source You won't anything there so we need to copy it from uncompressed sources.

```sh
cd /usr/local/php/lib
cp ~/sources/php-7.0.22/php.ini-development ./php.ini
vi php.ini
```
This is pretty large file with lot of configuration settings. Fortunately we only need to change some of the options:
```
short_open_tag = On
open_basedir = /var/www
disable_functions = exec,passthru,shell_exec,system,proc_open,popen
expose_php = Off
max_execution_time = 30
memory_limit = 64M
date.timezone = Europe/Warsaw
error_reporting = E_ALL & ~E_DEPRECATED & ~E_STRICT
display_errors = Off
display_startup_errors = Off
log_errors = On
post_max_size = 5M
upload_max_filesize = 4M

opcache.enable=1
opcache.memory_consumption=64
opcache.interned_strings_buffer=16
opcache.max_accelerated_files=7000
opcache.validate_timestamps=0 ;set this to 1 on production server
opcache.fast_shutdown=1
```

So we set few things here, enable <? tag, limit access to files from PHP level, disabled dangerous functions, adjusts timezone, security, max execution times, errors etc. In addition we have enable OPCache for PHP.

Each one of these options are well commented inside php.ini file. If You don't like the settings here or You need something else, feel free to change it for Your purposes.

### Useful shell scripts for PHP
#### /etc/init.d/php-fpm

As You probably remember during Apache setup we create script so we can use service command to start / stop Apache process. Now we will do the same for PHP-FPM

With PHP source code there comes ready script for that purpose.
```sh
cd /etc/init.d
cp ~/sources/php-7.0.22/sapi/fpm/init.d.php-fpm php-fpm
vi php-fpm
```
Now we need to setup configuration for the file:
```
prefix=/usr/local/php
exec_prefix=${prefix}

php_fpm_BIN=${exec_prefix}/sbin/php-fpm
php_fpm_CONF=${prefix}/etc/php-fpm.conf
php_fpm_PID=/var/run/php-fpm.pid
```

Save the file and add executable permission.

```sh
chmod +x php-fpm
servcie php-fpm status
service php-fpm start
servcie php-fpm status
```

After that we should have php-fpm process up and running!

#### Add PHP to $PATH
We can do one more thing to make our life easier:) Add PHP executable to PATH, so we'll be able to call php command from every directory.

```sh
echo 'pathmunge /usr/local/php/bin' > /etc/profile.d/php.sh
```

Execute such command, log out, log in and You'll be able to execute:
```sh
php -v
```

### Setup Apache for PHP-FPM
Now is the time to finally setup Apache for .php files. Let's edit one of the Virtual Hosts now.
```sh
vi /usr/local/apache2/conf/vhosts/example.com.conf
```
```
<VirtualHost *:80>
    ServerName example.com
    DocumentRoot "/var/www/html"

    <FilesMatch "\.(php*|phtm|phtml)$">
        SetHandler "proxy:fcgi://127.0.0.1:9000"
    </FilesMatch>

////Rest of the file below
```

So basically we need to proxy all files with .php extension to our PHP-FPM process.  Also we need to restart Apache and make sure PHP-FPM is running httpd server:

```sh
service php-fpm start
service httpd restart
```

### How to test if PHP is working?
We need to test if our PHP installation works. The easiest way to debug and check what's going on would be to create test.php file inside our /var/www directory.

```sh
vi /var/www/example.com/htdocs/test.php
```
and paste phpinfo() function there:

```php
<?php

phpinfo();
```

Save the file and open the file in Your browser, assuming that your vagrant setup is correct. For instance http://example.com/test.php or 192.168.99.99/test.php

If everything is OK you should get information about PHP installation. Well done!
