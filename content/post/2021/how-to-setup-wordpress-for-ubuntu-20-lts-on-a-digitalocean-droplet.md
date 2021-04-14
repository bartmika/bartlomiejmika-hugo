---
title: "How to Setup Wordpress for Ubuntu 20 LTS on a Digitalocean Droplet"
date: 2021-03-31T21:48:15-04:00
draft: false
author: "Bartlomiej Mika"
categories:
- "Web Development"
tags:
- "WordPress"
- "DevOps"
---

Do you setup WordPress sites often and you want instructions you can repeat quickly? The purpose of this HOWTO is to provide a series of copy and paste commands you can use over and over to setup WordPress on an Ubuntu 20 LTS server using DigitalOcean for your future clients.

<!--more-->

# Prerequisites

Before we begin, we'll need to establish a few items:

1. You need a **[DigitalOcean](https://m.do.co/c/2937038efac6)** account to follow along in this tutorial. If you don't have one, head on over there and sign up.
2. In this HOWTO I will be used `techops$`, `developer$` and `root#` in front of all my commands, this is to let you know the account you should be logged in as when executing the commands.
3. I assume you have generated a private public ssh keys and uploaded your public key to [DigitalOcean](https://m.do.co/c/2937038efac6).
4. I assume you know how to use ``vim`` in your linux console.
5. I assume you have purchased a domain name. For this HOWTO, I will be using *idiotbank.com*, so you'll need to replace all instances of this domain with your own. I will be using Namecheap as my domain name registrar.
6. This article was written in 2021 and if you are reading this at a different year, much will have changed!
7. I am assuming basic proficiency level with linux and if you don't understand a command you will look it up for more details.

# Instructions

## Setup Droplet
Log into your DigitalOcean dashboard and click the **Create > Droplet** button. The following options should be selected at *minimum*:

* **Choose an image:** Ubuntu 20.04 (LTS) x64
* **Choose a plan:** **(1)** Regular Intel with SSD **(2)** $10/mo
* **Choose a datacenter region:** Toronto
* **Select additional options:** IPv6, Monitoring
* **Authentication:** SSH Keys - Your account selected
* **Choose a hostname:** idiotbank-wordpress
* Finally click **Create Droplet** button when you are ready

Once the droplet has been created, please make a note of the IP addresses the droplet uses:

* **IPv4:** 159.203.12.17
* **IPv6:** 2604:a880:cad:d0::d92:3001

**Notes:**
* 1. Please be aware that through this tutorial we will use these IP addresses, you must replace these values with your own. If you don't do this then the rest of this tutorial will be very confusing!*
* 2. In the side sub-menu, please click **Snapshots** and be aware that throughout this tutorial you will periodically take snapshots of your droplet. Also this is where you will restore your droplet if you messed anything up while setting up your WordPress server.

## Setup DNS

While being logged in your DigitalOcean, click the **Networking** menu item on the side menu. In the tabs, please click the **Domains** item. Once on this page, add your domain name in the textfield, select your project and click **Add Domain** button. Once clicked please make the following changes by going to the appropriate record (A, AAA, CNAME, ETC) and fill out the fields as follows:

* **A Record**
  * **HOSTNAME:** idiotbank.com
  * **WILL DIRECT TO:** 159.203.12.17
  * **TTL (SECONDS):** 80
* **AAAA Record**
  * **HOSTNAME:** idiotbank.com
  * **WILL DIRECT TO:** 2604:a880:cad:d0::d92:3001
  * **TTL (SECONDS):** 80
* **CNAME Record**
  * **HOSTNAME:** *
  * **IS AN ALIAS OF:** @
  * **TTL (SECONDS):** 80

Please log into your domain name registrar and point your purchased domain to DigitalOcean nameservers. The following [article published by DigitalOcean](https://www.digitalocean.com/community/tutorials/how-to-point-to-digitalocean-nameservers-from-common-domain-registrars) provides various setup instructions you can follow. Please find your domain name registrar and follow their instructions. Once you have finished, please wait a few moments and then resume this tutorial.

## Connect to your Droplet

In your developers machine, please connect by running the command:

{{< highlight bash "linenos=false">}}
developer$ ssh -l root 159.203.12.17
{{</ highlight >}}

Once you are connected then congratulations, you are ready to begin!

## Setup administrators account

We will create an administrative account called ``techops`` so we don't use our ``root`` account - this is done on all Linux machines for security purposes, for more information please read [this article](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-20-04). You can name your administrative account whatever you like, for example ``bart``, however for this article we will refer to the ``techops`` user account as our administrators account.

Begin with the following:

{{< highlight bash "linenos=false">}}
root# adduser techops
root# usermod -aG sudo techops
{{</ highlight >}}

Change into our new account

```
root# su - techops
```

## Setup Basic Firewall
Configure a basic firewall which we will grow with in our system.

{{< highlight bash "linenos=false">}}
techops$ sudo ufw allow OpenSSH
techops$ sudo ufw enable
techops$ sudo ufw status
{{</ highlight >}}

## Basic Ubuntu Update
Update all the packages we have in our system. *Do not skip this step.*

{{< highlight bash "linenos=false">}}
techops$ sudo apt update
{{</ highlight >}}

## Setup Basic Nginx
Install, and configure our web-server

{{< highlight bash "linenos=false">}}
techops$ sudo apt install nginx
techops$ sudo ufw app list
techops$ sudo ufw allow 'Nginx HTTP'
techops$ sudo ufw status
{{</ highlight >}}

Check to see if ``nginx`` has been started and running in the background. *You should see no error messages.*
{{< highlight bash "linenos=false">}}
techops$ sudo systemctl status nginx
{{</ highlight >}}

In your browser check [http://159.203.12.17](159.203.12.17) and you should see an output.

![Welcome to Nginx on Ubuntu](/img/2021/03-31/welcome-to-nginx-on-Ubuntu.jpg)

*If you see a page load up with no errors then please take a snapshot in your DigitalOcean droplet so you can return to this latest working configuration later in the tutorial.*

## Setup Website Home Directory and Domain Name

Run the following:

{{< highlight bash "linenos=false">}}
techops$ sudo mkdir -p /var/www/idiotbank.com/html
techops$ sudo chown -R $USER:$USER /var/www/idiotbank.com/html
techops$ sudo chmod -R 755 /var/www/idiotbank.com
techops$ vi /var/www/idiotbank.com/html/index.html
{{</ highlight >}}

*Please note that you just ran the `vi` linux command. If you are not familiar with how to use `vi` then please watch [this quick youtube](https://youtu.be/pU2k776i2Zw) explaining how to use it. Once you understand proceed forward with this tutorial - do not proceed forward if you do not understand!*

Inside, add the following sample HTML and save the file:

{{< highlight html "linenos=false">}}
<!-- /var/www/idiotbank.com/html/index.html -->
<html>
    <head>
        <title>Welcome to idiotbank.com!</title>
    </head>
    <body>
        <h1>Success!  The idiotbank.com server block is working!</h1>
    </body>
</html>
{{</ highlight >}}

Next load up our virtual host configuration:

{{< highlight bash "linenos=false">}}
techops$ sudo vi /etc/nginx/sites-available/idiotbank.com
{{</ highlight >}}

Copy and paste the following code:

{{< highlight nginx "linenos=false">}}
server {
        listen 80;
        listen [::]:80;

        root /var/www/idiotbank.com/html;
        index index.html index.htm index.nginx-debian.html;

        server_name idiotbank.com www.idiotbank.com;

        location / {
                try_files $uri $uri/ =404;
        }
}
{{</ highlight >}}

Attach our new virtual host to our web-server:

{{< highlight bash "linenos=false">}}
techops$ sudo ln -s /etc/nginx/sites-available/idiotbank.com /etc/nginx/sites-enabled/
{{</ highlight >}}

Open up our web-server's configuration:

{{< highlight bash "linenos=false">}}
techops$ sudo vi /etc/nginx/nginx.conf
{{</ highlight >}}

Search through the file until you see the following code and uncomment it out and save the file:

{{< highlight nginx "linenos=false">}}
...
http {
    ...
    server_names_hash_bucket_size 64;
    ...
}
...
{{</ highlight >}}


Verify our web-server works

{{< highlight bash "linenos=false">}}
techops$ sudo nginx -t
{{</ highlight >}}

*Note: There should be no errors here, if there are please investigate before proceeding any further!*

Restart our web-server

{{< highlight bash "linenos=false">}}
techops$ sudo systemctl restart nginx
{{</ highlight >}}

Now go to [http://idiotbank.com](http://idiotbank.com) and you should see a page!

*If you see a page load up with no errors then please take a snapshot in your DigitalOcean droplet so you can return to this latest working configuration later in the tutorial.*

## Setup MariaDB Database
The database is an essential piece we cannot forget. While we have the option of using `MySQL`, in this tutorial we will use `MariaDB` instead.

Begin by running the following:

{{< highlight bash "linenos=false">}}
techops$ sudo apt update
techops$ sudo apt install mariadb-server
techops$ sudo mysql_secure_installation
{{</ highlight >}}

Access our database

{{< highlight bash "linenos=false">}}
techops$ sudo mariadb
{{</ highlight >}}

Creating an Administrative User that Employs Password Authentication.

{{< highlight sql "linenos=false">}}
GRANT ALL ON *.* TO 'admin'@'localhost' IDENTIFIED BY '123password' WITH GRANT OPTION;
FLUSH PRIVILEGES;
{{</ highlight >}}

*Note: This account is the administrative account which you can use to access **all** the databases in our system. Please be sure to use a more secure password!*

{{< highlight bash "linenos=false">}}
techops$ sudo systemctl status mariadb
{{</ highlight >}}

*You should get no errors after running the above command.*

Enable database on startup

{{< highlight bash "linenos=false">}}
techops$ sudo systemctl enable mariadb
{{</ highlight >}}

Finally check

{{< highlight bash "linenos=false">}}
techops$ mysqladmin -u admin -p version
{{</ highlight >}}

## Setup PHP

Let's install our server-side programming language. Start by running the following to install our dependencies:

{{< highlight bash "linenos=false">}}
techops$ sudo apt install php-fpm php-mysql
{{</ highlight >}}

Open up our virtual host configuration:

{{< highlight bash "linenos=false">}}
techops$ sudo vi /etc/nginx/sites-available/idiotbank.com
{{</ highlight >}}

And replace completely with the following copy and paste:

{{< highlight nginx "linenos=false">}}
server {
    listen 80;
    server_name idiotbank.com www.idiotbank.com;
    root /var/www/idiotbank.com;

    index index.html index.htm index.php;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php7.4-fpm.sock;
     }

    location ~ /\.ht {
        deny all;
    }

}
{{</ highlight >}}

Unlink the following:

{{< highlight bash "linenos=false">}}
techops$ sudo unlink /etc/nginx/sites-enabled/default
{{</ highlight >}}

Open up our index file

{{< highlight bash "linenos=false">}}
techops$ sudo vi /var/www/idiotbank.com/index.html
{{</ highlight >}}

Update the file with this copy and paste

{{< highlight html "linenos=false">}}
<html>
  <head>
    <title>idiotbank.com website</title>
  </head>
  <body>
    <h1>Hello World!</h1>

    <p>This is the landing page of <strong>idiotbank.com</strong>.</p>
  </body>
</html>
{{</ highlight >}}

Next create the following file

{{< highlight bash "linenos=false">}}
techops$ sudo vi /var/www/idiotbank.com/info.php
{{</ highlight >}}

And copy and paste the following into it

{{< highlight php "linenos=false">}}
<?php
phpinfo();
{{</ highlight >}}

Finally in your browser visit the [http://idiotbank.com/info.php](http://idiotbank.com/info.php) URL path and you should see a success message detailing the libraries used.

*If you see a page load up with no errors then please take a snapshot in your DigitalOcean droplet so you can return to this latest working configuration later in the tutorial.*

## Setup Wordpress

The next instructions will be pertaining to setting up WordPress on our server. This section is divided into further sub-sections.

### Setup WordPress Database

Enter our database

{{< highlight bash "linenos=false">}}
techops$ mysql -u admin -p
{{</ highlight >}}

And create our WordPress database

{{< highlight sql "linenos=false">}}
CREATE DATABASE wordpress DEFAULT CHARACTER SET utf8 COLLATE utf8_unicode_ci;
{{</ highlight >}}

Create our WordPress user

{{< highlight sql "linenos=false">}}
CREATE USER 'wordpressuser'@'localhost' IDENTIFIED BY '123password';

GRANT ALL ON wordpress.* TO 'wordpressuser'@'localhost';

EXIT;
{{</ highlight >}}

### Setup WordPress PHP Dependencies

Install our dependencies

{{< highlight bash "linenos=false">}}
techops$ sudo apt install php-curl php-gd php-intl php-mbstring php-soap php-xml php-xmlrpc php-zip
{{</ highlight >}}

Restart our ``php``

{{< highlight bash "linenos=false">}}
techops$ sudo systemctl restart php7.4-fpm
{{</ highlight >}}

### Setup Nginx for WordPress

Open virtual host configuration:

{{< highlight bash "linenos=false">}}
techops$ sudo vi /etc/nginx/sites-available/idiotbank.com
{{</ highlight >}}

Replace the contents of the file by copy and pasting the follwing:

{{< highlight nginx "linenos=false">}}
server {
    listen 80;
    server_name idiotbank.com www.idiotbank.com;
    root /var/www/idiotbank.com/wordpress;

    index index.html index.htm index.php;

    location / {
       # try_files $uri $uri/ =404;
       try_files $uri $uri/ /index.php$is_args$args;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php7.4-fpm.sock;
     }

    location ~ /\.ht {
        deny all;
    }

    location = /favicon.ico { log_not_found off; access_log off; }
    location = /robots.txt { log_not_found off; access_log off; allow all; }
    location ~* \.(css|gif|ico|jpeg|jpg|js|png)$ {
        expires max;
        log_not_found off;
    }
}
{{</ highlight >}}

Next check to see if any configuration issues occur

{{< highlight bash "linenos=false">}}
techops$ sudo nginx -t
{{</ highlight >}}

If no errors occur then please run the following to reload our web-server

{{< highlight bash "linenos=false">}}
techops$ sudo systemctl reload nginx
{{</ highlight >}}


### (Actually) Setup WordPress

Run the following which will download the latest WordPress files, and setup our folder structure for the project:

{{< highlight bash "linenos=false">}}
techops$ cd /tmp
techops$ curl -LO https://wordpress.org/latest.tar.gz
techops$ tar xzvf latest.tar.gz
techops$ cp /tmp/wordpress/wp-config-sample.php /tmp/wordpress/wp-config.php
techops$ sudo cp -a /tmp/wordpress/. /var/www/idiotbank.com/wordpress
techops$ sudo chown -R www-data:www-data /var/www/idiotbank.com/wordpress
{{</ highlight >}}

Run the following and copy the received content into your clipboard.

{{< highlight bash "linenos=false">}}
techops$ curl -s https://api.wordpress.org/secret-key/1.1/salt/
{{</ highlight >}}

Open up your WordPress configuration file.

{{< highlight bash "linenos=false">}}
techops$ sudo vi /var/www/idiotbank.com/wordpress/wp-config.php
{{</ highlight >}}

Please make sure the following tasks are complete with your open configuration file:
* a. Change the database name
* b. Change the database user
* c. Change the database password
* d. Paste in the clipboard for the dummy section
* e. Append ``define( 'FS_METHOD', 'direct' );`` to the end of the file

Save the file, now open up your browser and enter the [http://idiotbank.com/wordpress](http://idiotbank.com/wordpress) URL path. If everything works fine, you should see a WordPress configuration screen - Please fill out your form.

![WP First Page](/img/2021/03-31/wp-setup.png)

*If you see a page load up with no errors then please take a snapshot in your DigitalOcean droplet so you can return to this latest working configuration later in the tutorial.*

## Setup SSL

Install our dependencies:

{{< highlight bash "linenos=false">}}
techops$ sudo apt install certbot python3-certbot-nginx
{{</ highlight >}}

Reload our web-server:

{{< highlight bash "linenos=false">}}
techops$ sudo systemctl reload nginx
{{</ highlight >}}

Update our firewall to make it advanced:

{{< highlight bash "linenos=false">}}
techops$ sudo ufw status
techops$ sudo ufw allow 'Nginx Full'
techops$ sudo ufw delete allow 'Nginx HTTP'
{{</ highlight >}}

Install our SSL certificate and follow the instructions when prompted:

{{< highlight bash "linenos=false">}}
techops$ sudo certbot --nginx -d idiotbank.com -d www.idiotbank.com
{{</ highlight >}}

Now open up your browser and enter the [https://idiotbank.com](https://idiotbank.com) URL path. If everything works fine, you should see a WordPress page running without problems.

*If you see a page load up with no errors then please take a snapshot in your DigitalOcean droplet so you can return to this latest working configuration later in the tutorial.*

# Special Thanks:

The following articles were invaluable and more-then-inspired this article.

* https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-20-04
* https://www.digitalocean.com/community/tutorials/how-to-install-nginx-on-ubuntu-20-04
* https://www.digitalocean.com/community/tutorials/how-to-install-mariadb-on-ubuntu-20-04
* https://www.digitalocean.com/community/tutorials/how-to-install-linux-nginx-mysql-php-lemp-stack-on-ubuntu-20-04
* https://www.digitalocean.com/community/tutorials/how-to-install-wordpress-with-lemp-on-ubuntu-20-04
* https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-20-04
