# php7-from-scratch
Instructions on how to compile PHP 7.2.1 from source on Ubuntu 17.10 machines. This
will also include instructions on how to configure the machine for Nginx.

These instructions are ideal for those who want to run PHP in production. This version of PHP lacks intrusive packages like Suhosin. Also, you don't have install any packages like php-json, php-mysql, or php-common/php-cli.

Instead, you have 1 PHP binary, and a PHP FPM binary.

## Create directory for the PHP Binaries
The PHP binaries will be installed in a `bin` directory off of the `~/` directory.

    mkdir -p ~/bin/

## Add the bin directory to your Path
This will allow you to use type `php` anywhere and the version of PHP we're compiling
will be first on the list - and the first one to be run

Open the file `~/.bashrc` and add the following to the end of it

    cat >> ~/.bashrc

Now, paste the following into your terminal

    # add local bin directory
    if [ -d "$HOME/bin" ] ; then
      PATH="$HOME/bin:$PATH"
    fi


## Install packages to prepare our system
Install the following packages. These are needed for PHP to communicate with
MySQL, Postgres, composer, and Nginx.


    sudo apt-get install autoconf build-essential curl libtool \
      libssl-dev libcurl4-openssl-dev libxml2-dev libreadline7 \
      libreadline-dev libzip-dev libzip4 nginx openssl \
      pkg-config zlib1g-dev

## Install MySQL
If you do not plan on installing MySQL, then skip this step

    sudo apt-get install mysql-server mysql-client

## install Postgres
If you don't plan on installing PostgreSQL then skip this step.

    # I like to add a postgres user (but, this is optional)
    sudo adduser postgres

    sudo apt-get install postgresql-9.6 postgresql-client-9.6 postgresql-contrib-9.6 libpq-dev

## Fix easy.h error __IF__ you do not install `pkg-config`
For some reason, Debian based distros don't report the correct location of openSSL headers. __IF__ you do not install the package `pkg-config` PHP looks in the wrong location for the headers. This symlink fixes that.

    sudo ln -s /usr/include/x86_64-linux-gnu/curl /usr/include/curl

## Download Latest PHP 7.2.1
As of the time of this, PHP 7.2.1 was the latest PHP available.

__NOTE: this installs PHP in the user's local bin directory, not
globally!__

    # create the Download directory
    mkdir -p ~/Downloads/

    # change to the Downloads directory and fetch the PHP 7.2 tarball
    cd ~/Downloads;
    wget http://us3.php.net/get/php-7.2.1.tar.xz/from/this/mirror -O php-7.2.1.tar.xz
    tar -xf php-7.2.1.tar.xz
    cd php-7.2.1


__IF YOU DID NOT INSTALL MYSQL, REMOVE THE FOLLOWING LINES FROM THIS SCRIPT__

    --enable-mysqlnd \
    --with-pdo-mysql \
    --with-pdo-mysql=mysqlnd \

__IF YOU DID NOT INSTALL POSTGRES, REMOVE THE FOLLOWING LINES FROM THIS SCRIPT__

    --with-pdo-pgsql=/usr/bin/pg_config \

Paste the following command into the terminal, hit enter

    cat > build_php.sh

Then paste this

    #!/bin/sh

    ./configure --prefix=$HOME/bin/php7 \
        --enable-mysqlnd \
        --with-pdo-mysql \
        --with-pdo-mysql=mysqlnd \
        --with-pdo-pgsql=/usr/bin/pg_config \
        --enable-bcmath \
        --enable-fpm \
        --with-fpm-user=www-data \
        --with-fpm-group=www-data \
        --enable-mbstring \
        --enable-phpdbg \
        --enable-shmop \
        --enable-sockets \
        --enable-sysvmsg \
        --enable-sysvsem \
        --enable-sysvshm \
        --enable-zip \
        --with-libzip=/usr/lib/x86_64-linux-gnu \
        --with-zlib \
        --with-curl \
        --with-pear \
        --with-openssl \
        --enable-pcntl \
        --with-readline

    # press ctrl+c to exit out of cat

#### compile and install
Remember, PHP will be installed in the home directory.

    sh build_php.sh
    make
    make install

#### Edit the PHP.ini file
Copy the *php-7.1.11/php.ini-development* file from the source directory to the *~/bin/php7/lib/* directory, and rename
the file to *php.ini*

    cp php.ini-development ~/bin/php7/lib/php.ini
    cd ~/bin/php7/lib;

You'll need to change a few settings

Change your timezone

    [Date]
    ; Defines the default timezone used by the date functions
    ; http://php.net/date.timezone
    date.timezone = America/Chicago

The MySQL socket PDO connects to

    ; Default socket name for local MySQL connects.  If empty, uses the built-in
    ; MySQL defaults.
    ; http://php.net/pdo_mysql.default-socket
    pdo_mysql.default_socket=/var/run/mysqld/mysqld.sock

PHP-FPM fix (You will most likely have to add this to the file)

    [php-fpm]
    cgi.fix_pathinfo=0:

You need to create a *php-fpm.conf* file to use Nginx.

    cd ~/bin/php7/etc/; mv php-fpm.conf.default php-fpm.conf
    cd ~/bin/php7/etc/php-fpm.d/; mv www.conf.default www.conf

In your favorite editor, open the www.conf file, and change the user and group
from nobody to www-data (the user for nginx). **THIS SHOULD BE DONE FOR YOU
JUST DOUBLE CHECK TO MAKE SURE IT'S SET**

    ...
    user = www-data
    group = www-data
    ...

__ADD PHP TO YOUR PATH__

Assuming you'll be logged into your server not as root, add the PHP binary to
your .bashrc profile

    cd ~/bin
    ln -s php7/bin/php php
    ln -s php7/bin/php-cgi php-cgi
    ln -s php7/bin/php-config php-config
    ln -s php7/bin/phpize phpize
    ln -s php7/bin/phar.phar phar
    ln -s php7/bin/pear pear
    ln -s php7/bin/phpdbg phpdbg
    ln -s php7/sbin/php-fpm php-fpm

### Start PHP-FPM
You'll have to start *php-fpm* manually to get it working with NginX

    sudo ~/bin/php7/sbin/php-fpm

## WEBSERVERS

__Laravel 5.5__
If you're using Laravel 5.5 this is how you start the built in PHP server

    php artisan serve --host=your.ip.of.vm --port=8000

__PHP CLI SERVER__
If you want to use the built in PHP web-server

    php -S ip.of.your.machine:port_number

__Nginx__ If you're using nginx

    cd /etc/nginx/sites-available

Open *default* and change the root directory to where your code is

    root /path/to/your_PHP_project;

Add index.php to the index list

    index index.php;

Remove the *location* section and make the following changes in the
FastCGI section

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;

        # With php7.0-cgi alone:
        fastcgi_pass 127.0.0.1:9000;
    }

So, your conf file should look like this:

    server {
        listen 80 default_server;
        listen [::]:80 default_server;

        # path to code (index.php should be in this directory)
        root /path/to/PHPCODE;

        index index.php;

        server_name virtualboxphpdev;

        location ~ \.php$ {
            include snippets/fastcgi-php.conf;

            # With php7.0-cgi alone:
            fastcgi_pass 127.0.0.1:9000;
        }
    }

Nginx should now be serving your files

#### Nginx Virtualbox Bug
You'll need to edit `/etc/nginx/nginx.conf` and change the line `sendfile on;` to `sendfile off;`

What will happen is changes to static files (CSS and Javascript) files won't be updated. Turning off _sendfile_ will cause Nginx to serve the file via a different method and the new file's changes will be displayed immediately.

## Development on Virtual Machines
If you are developing on a virtual machine, like VirtualBox or Parallels and want to use this setup for development, heres how to set up the virtual machine.

Again, this will be for an Ubuntu server. We will mount a directory on your harddrive and sync it to a directory on the virtual machine.

# VIRTUALBOX
### Step 1: mount guest additions
Insert the guest additions CD image, then run this on the Linux VM

    sudo mount /dev/cdrom /media/cdrom

### Step 2: install guest additions:

   sudo /media/cdrom/VBoxLinuxAdditions.run

### Step 3: Add your user to the vboxsf

    sudo adduser PUT_YOUR_USERNAME_HERE vboxsf
    sudo adduser var-www vboxsf
