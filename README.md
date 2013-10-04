Le but de cette étape est d'avoir un environnement permettant l'utilisation de plusieurs versions de PHP en  parallèle (le tout avec Apache).
The goal here is to be able to run Apache/PHP with multiple PHP installed on the machine, letting you a way to choose which version you want to run.

To be do that, we need to compile our own PHP.

First, be sure to grab all the dependencies
```
apt-get install build-essential libxml2-dev autoconf2.13 libpng12-* zlib1g gawk bison flex ^libxml2-* mcrypt libmcrypt-dev perl libcurl4-gnutls-dev libicu-dev libxslt1-dev
```

We also need the JPEG lib
```
apt-get install libjpeg62-dev
```

And the OpenSSL lib.

```
apt-get install libcurl4-openssl-dev
```

Then, we need  Apache and <a href="http://httpd.apache.org/docs/2.2/mod/worker.html">Apache MPM Worker</a>
```
apt-get install apache2 apache2-mpm-worker
```


We also need Apache's mod_fastcgi
If you try to run
```
apt-get install libapache2-mod-fastcgi
```
You'll get an error
```
E: Package 'libapache2-mod-fastcgi' has no installation candidate
```
This is perfectly normal since the fastcgi package is in the "non-free" repository of debian.

We need to add those one
```
vim /etc/apt/sources.list
```
Add those 2 lines : 
```
deb http://ftp.fr.debian.org/debian/ wheezy non-free
deb-src http://ftp.fr.debian.org/debian/ wheezy non-free
```

Save, quit

Don't forget to update !
```
apt-get update
```

Now, you can try again : 
```
apt-get install libapache2-mod-fastcgi
```

Then you'll need also those libs
```
apt-get install libreadline-dev libfreetype6-dev
```

It should now be good for Apache, at least for now.

Next, PHP !
As said before, we'll need to compile our PHP.
So, grab the sources : http://php.net/downloads.php
We'll start with version 5.5 (5.5.4 right now)


```
cd /opt
mkdir src
cd src
wget http://fr2.php.net/get/php-5.5.4.tar.gz/from/this/mirror
mv mirror php-5.5.4.tar.gz
tar xvfz mirror php-5.5.4.tar.gz 
cd php-5.5.4
```

Next, configure ! 
```
./configure \
--prefix=/opt/php5.5.4 \
--with-readline \
--enable-cli \
--with-mysql=mysqlnd \
--with-mysqli=mysqlnd \
--with-pdo-mysql=mysqlnd \
--with-gd \
--with-freetype-dir \
--with-jpeg-dir \
--with-png-dir \
--with-zlib-dir \
--enable-gd-native-ttf \
--with-curl \
--enable-bcmath \
--enable-mbstring \
--enable-exif \
--enable-pcntl \
--enable-soap \
--with-xsl \
--enable-zip \
--with-config-file-path=/opt/php5.5.4/conf \
--with-config-file-scan-dir=/opt/php5.5.4/conf.d \
--with-iconv-dir=/opt/local \
--enable-fpm \
--with-fpm-user=www-data \
--with-fpm-group=www-data
```
You need to be carfeful with the paths (/opt/php5.5.4/xxx) right here. It is used to be able to say PHP when it should be installed and where the configuration files are looked for.

(here, you can grab a little coffee)

Then, the make ! 

```
make
```

(Here, you can grab a big coffee)

Then, install it ! 
```
make install
```

Just check that everything is there. Just go :
```
cd /opt/php5.5.4/
```

You should find : 
```
ls /opt/php5.5.4
bin  etc  include  lib     php  sbin  var
```

Now, we need to make some changes in the php-fpm configuration file.

Thankfully, PHP have an example file we could use :
```
cp /opt/php5.5.4//etc/php-fpm.conf.default /opt/php5.5.4/etc/php-fpm.conf
```

Edit this file : 
```
vim /opt/php5.5.4/etc/php-fpm.conf  
```

Here are the changes you need to do : 
In [global], uncomment
```
pid = run/php-fpm.pid
```

And in the "Pool Definitions", you'll find the line : 
```
listen = 127.0.0.1:9000
```
Change this one to : 
```
listen = /var/run/php-fpm-5.5.4.sock
```
This will allow us not to "plug" PHP to an IP:PORT, but to an unix socket.
Save and quit.


We'll also need a basic php.ini file. Once again, PHP have one (in the souces) : 
```
cd /opt/php5.5.4
mkdir conf
cp /opt/src/php-5.5.4/php.ini-development /opt/php5.5.4/conf/ 
```
We'll also need a startup script.

```
cp /opt/src/php-5.5.4/sapi/fpm/init.d.php-fpm /etc/init.d/php-fpm5.5.4
```
Be sure the script is executable
```
chmod +x  /etc/init.d/php-fpm5.5.4
```
Next, we'll go back to Apache configuration

Be sure that the "actions" and "fastcgi" extensions are enable
```
a2enmod actions fastcgi 
```

Now, we need to change the mod_fastcgi configuration to fits our needs
```
cd /etc/apache2/mods-enabled/
vim fastcgi.conf
```

This file needs to be changes to that : 
```
<IfModule mod_fastcgi.c>
 AddHandler fastcgi-script .fcgi
 FastCgiIpcDir /var/lib/apache2/fastcgi
 FastCgiConfig -autoUpdate -singleThreshold 100 -killInterval 300 -maxProcesses 10 -maxClassProcesses 1
 AddHandler php5-fcgi .php
 AddType application/x-httpd-php .php
 ScriptAlias /fastcgi/ /var/www/fastcgi/

 <Directory "/var/www/fastcgi">
   Options +ExecCGI
   SetHandler fastcgi-script
   Order Deny,Allow
   Allow from env=REDIRECT_STATUS
 </Directory>
 <Location "/fastcgi/php5.fpm">

     Order Deny,Allow
     Deny from all
     Allow from env=REDIRECT_STATUS
 </Location>
 FastCgiExternalServer /var/www/fastcgi/php5.fpm -socket /var/run/php-fpm-5.5.4.sock
 Action php5-fcgi /fastcgi/php5.fpm

</IfModule>
```

The important part here is ```FastCgiExternalServer /var/www/fastcgi/php5.fpm -socket /var/run/php-fpm-5.5.4.sock```
The last part (```-socket /var/run/php-fpm-5.5.4.sock```) is the same that we have changed before in the "/opt/php5.5.4/etc/php-fpm.conf" file.
 

About the ```/var/www/fastcgi/php5.fpm<```, the directory "fastcgi" needs to exist. It can be empty, but it really need to exist. So, let's create it :
```
mkdir /var/www/fastcgi/
```

Et voilà, let's test it now ! 

Add a php file in /var/www
```
vim /var/www/index.php
```

With just a simple phpinfo() in it
```
<?php phpinfo();
```


Start Apache and php-fpm
```
/etc/init.d/apache2 restart
/etc/init.d/php-fpm5.5.4 restart  
```

Now, with your broswer, go check that everything works.

http://<IP>/index.php
You should see "PHP Version 5.5.4"

We now have our Apache/PHP working. But what about adding a PHP 5.3 now ?

Compile it !
```
cd /opt/src/
wget http://fr2.php.net/get/php-5.3.27.tar.gz/from/this/mirror
mv mirror php-5.3.27.tar.gz
tar xvfz php-5.3.27.tar.gz 
cd php-5.3.27
```

Don't forget to change the paths 
```
./configure \
--prefix=/opt/php5.3.27 \
--with-readline \
--enable-cli \
--with-mysql=mysqlnd \
--with-mysqli=mysqlnd \
--with-pdo-mysql=mysqlnd \
--with-gd \
--with-freetype-dir \
--with-jpeg-dir \
--with-png-dir \
--with-zlib-dir \
--enable-gd-native-ttf \
--with-curl \
--enable-bcmath \
--enable-mbstring \
--enable-exif \
--enable-pcntl \
--enable-soap \
--with-xsl \
--enable-zip \
--with-config-file-path=/opt/php5.3.27/conf \
--with-config-file-scan-dir=/opt/php5.3.27/conf.d \
--with-iconv-dir=/opt/local \
--enable-fpm \
--with-fpm-user=www-data \
--with-fpm-group=www-data
```

```
make
```
(coffee)
```
make install
```

```
cd /opt/php5.3.27/
cp /opt/php5.3.27/etc/php-fpm.conf.default /opt/php5.3.27/etc/php-fpm.conf
vim /opt/php5.3.27/etc/php-fpm.conf
```

As before, change the values : 
```
pid = run/php-fpm.pid
```
And of course, change the socket (use an other one)
```
 listen = /var/run/php-fpm-5.3.27.sock
```

php.ini and startup script : 
```
cd /opt/php5.3.27
mkdir conf
cp /opt/src/php-5.3.27/php.ini-development /opt/php5.3.27/conf/ 
cp /opt/src/php-5.3.27/sapi/fpm/init.d.php-fpm /etc/init.d/php-fpm5.3.27
chmod +x  /etc/init.d/php-fpm5.3.27
```

We also need to make an other change in the fastcgi.conf file : 
```
vim /etc/apache2/mods-enabled/fastcgi.conf
```

See the line
```
FastCgiExternalServer /var/www/fastcgi/php5.fpm -socket /var/run/php-fpm-5.5.4.sock
```
Add an other one, just below
```
FastCgiExternalServer /var/www/fastcgi/php5.3.fpm -socket /var/run/php-fpm-5.3.27.sock 
```

We should have : 
```
<IfModule mod_fastcgi.c>
 AddHandler fastcgi-script .fcgi
 FastCgiIpcDir /var/lib/apache2/fastcgi
 FastCgiConfig -autoUpdate -singleThreshold 100 -killInterval 300 -maxProcesses 10 -maxClassProcesses 1
 AddHandler php5-fcgi .php
 AddType application/x-httpd-php .php
 ScriptAlias /fastcgi/ /var/www/fastcgi/

 <Directory "/var/www/fastcgi">
   Options +ExecCGI
   SetHandler fastcgi-script
   Order Deny,Allow
   Allow from env=REDIRECT_STATUS
 </Directory>
 <Location "/fastcgi/php5.fpm">

     Order Deny,Allow
     Deny from all
     Allow from env=REDIRECT_STATUS
 </Location>
 FastCgiExternalServer /var/www/fastcgi/php5.fpm -socket /var/run/php-fpm-5.5.4.sock
 FastCgiExternalServer /var/www/fastcgi/php5.3.fpm -socket /var/run/php-fpm-5.3.27.sock
 Action php5-fcgi /fastcgi/php5.fpm
</IfModule>
```

That it !
How to choose which PHP version you want to use ?
Easy, just go in your Apache Virtualhost, and change the "Action php5-fcgi /fastcgi/php5.fpm" to the right fastcgi server.
Example with the basic virtualhost
```
  vim /etc/apache2/sites-enabled/000-default
```

Before </VirtualHost>, add 
```
Action php5-fcgi /fastcgi/php5.3.fpm
```

So we have : 
```
<VirtualHost *:80>
        ServerAdmin webmaster@localhost

        DocumentRoot /var/www
        <Directory />
                Options FollowSymLinks
                AllowOverride None
        </Directory>
        <Directory /var/www/>
                Options Indexes FollowSymLinks MultiViews
                AllowOverride None
                Order allow,deny
                allow from all
        </Directory>

        ScriptAlias /cgi-bin/ /usr/lib/cgi-bin/
        <Directory "/usr/lib/cgi-bin">
                AllowOverride None
                Options +ExecCGI -MultiViews +SymLinksIfOwnerMatch
                Order allow,deny
                Allow from all
        </Directory>

        ErrorLog ${APACHE_LOG_DIR}/error.log

        # Possible values include: debug, info, notice, warn, error, crit,
        # alert, emerg.
        LogLevel warn

        CustomLog ${APACHE_LOG_DIR}/access.log combined
        Action php5-fcgi /fastcgi/php5.3.fpm

</VirtualHost>
```
Save and quit.

Launch all you php-fpm and Apache

```
/etc/init.d/php-fpm5.5.4 restart 
/etc/init.d/php-fpm5.3.27 restart 
/etc/init.d/apache2 restart 
```

Now, if you go on http://<IP>/index.php, you should see  "PHP Version 5.3.27"

Et voilà !

Prochaine étape, la connexion avec MySQL et les quelques extensions indispensables manquantes à l'heure actuelle !
