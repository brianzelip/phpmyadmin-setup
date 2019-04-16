# Notes on setting up phpMyAdmin on Digital Ocean

first link: [How To Install and Secure phpMyAdmin on Ubuntu 18.04](https://www.digitalocean.com/community/tutorials/how-to-install-and-secure-phpmyadmin-on-ubuntu-18-04)

second link: [Initial Server Setup with Ubuntu 18.04](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-18-04)

third link: [How to Connect to Droplets with SSH](https://www.digitalocean.com/docs/droplets/how-to/connect-with-ssh/)

fourth link, first action taken, occurred while spinning up the droplet's initial configuration: [How to Upload SSH Public Keys to a DigitalOcean Account](https://www.digitalocean.com/docs/droplets/how-to/add-ssh-keys/to-account/)

fifth link: [How To Install Linux, Apache, MySQL, PHP (LAMP) stack on Ubuntu 18.04](https://www.digitalocean.com/community/tutorials/how-to-install-linux-apache-mysql-php-lamp-stack-ubuntu-18-04)

## actual steps in practice

### 1. Initial server setup

1.1. Add ssh key ('brian-mbair', this key is machine-dependent) via droplet config form, [How to Upload SSH Public Keys to a DigitalOcean Account](https://www.digitalocean.com/docs/droplets/how-to/add-ssh-keys/to-account/)

1.2. ssh into droplet - [Initial Server Setup with Ubuntu 18.04](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-18-04)

```sh
ssh root@$IP_ADDRESS
```

After accpeting the initial fingerprint warning, I was logged right in since I added the ssh key during init.

See also this link for what to do if no ssh keys added during init config: [How to Connect to Droplets with SSH](https://www.digitalocean.com/docs/droplets/how-to/connect-with-ssh/).

1.3. Set up an alternative user account with a reduced scope of influence for day-to-day work

```sh
# add user, then provide password (see .env for password)
adduser brian

# add user to sudo group
usermod -aG sudo brian
```

1.4. Set up basic firewall

```sh
# see that OpenSSH is registered with UFW (Uncomplicated Firewall)
ufw app list

# make sure that the firewall allows SSH connections
# so that we can log back in next time
ufw allow OpenSSH

# enable the firewall
ufw enable

# see that SSH connections are still allowed
ufw status
```

> As **the firewall is currently blocking all connections except for SSH**, if you install and configure additional services, you will need to adjust the firewall settings to allow acceptable traffic in. You can learn some common UFW operations in [this guide](https://www.digitalocean.com/community/tutorials/ufw-essentials-common-firewall-rules-and-commands).

1.5. Enable external access for regular user, where the root account uses SSH key authentication (as opposed to password authentication)

```sh
rsync --archive --chown=brian:brian ~/.ssh /home/brian
```

### 2. Install LAMP stack

[How To Install Linux, Apache, MySQL, PHP (LAMP) stack on Ubuntu 18.04](https://www.digitalocean.com/community/tutorials/how-to-install-linux-apache-mysql-php-lamp-stack-ubuntu-18-04)

2.1. Install Apache

```sh
sudo apt update

sudo apt install apache2

# the following was auto suggested via CLI output, not the docs,
# after running sudo apt update
sudo apt autoremove

# make sure that your firewall allows HTTP and HTTPS traffic
# by checking that UFW has an application profile for Apache
sudo ufw app list

# another form of confirmation like the above command
sudo ufw app info "Apache Full"

# allow incoming HTTP and HTTPS traffic for this profile
sudo ufw allow in "Apache Full"
```

SUCCESS! The "Apache Ubuntu Default Page" is now showing at \$IP_ADDRESS.

2.2. Install MySQL

```sh
# install MySQL
sudo apt install mysql-server

# run a simple security script that comes pre-installed with MySQL
# which will remove some dangerous defaults and lock down access to your
# database system. Start the interactive script by running the following
# command:
#     choices made during the mysql_secure_installation interactive script:
#     1. enable the validate-password plugin for good passwords
#     2. root password - see .env
#     3. remove anonymous user that comes w/ by default for testing
#     4. disallow root login remotely, only from localhost
#     5. remove test database and access to it
#     6. reload privileges table (make MySQL immediately respect above changes)
sudo mysql_secure_installation

# switch MySQL authentication method when connecting as root,
# from `auth_socket` to password authentication. This is useful for
# allowing an external program (e.g., phpMyAdmin) to access the user.
sudo mysql
```

The final command above leads to a mysql session:

```sql
SELECT user,authentication_string,plugin,host FROM mysql.user;

-- output:
/*
+------------------+-------------------------------------------+-----------------------+-----------+
| user             | authentication_string                     | plugin                | host      |
+------------------+-------------------------------------------+-----------------------+-----------+
| root             |                                           | auth_socket           | localhost |
| mysql.session    | *THISISNOTAVALIDPASSWORDTHATCANBEUSEDHERE | mysql_native_password | localhost |
| mysql.sys        | *THISISNOTAVALIDPASSWORDTHATCANBEUSEDHERE | mysql_native_password | localhost |
| debian-sys-maint | *707265A2C4441B07A58963D5C24F737A8DBD7135 | mysql_native_password | localhost |
+------------------+-------------------------------------------+-----------------------+-----------+
*/

/* configure the root account to authenticate with a password,
   run the following ALTER USER command
*/
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY $MYSQL_ALTER_USER_PASSWORD;

-- flush privileges
FLUSH PRIVILEGES;

-- confirm root's new authentication method (via password)
SELECT user,authentication_string,plugin,host FROM mysql.user;

exit
```

2.3. Install PHP

```sh
# Install PHP and related packages
sudo apt install php libapache2-mod-php php-mysql

# tell the web server to prefer PHP files over others,
# so make Apache look for an index.php file first.
# Do this by changing the order of the file names listed in the following file
sudo nano /etc/apache2/mods-enabled/dir.conf

# restart Apache for above change to take effect
sudo systemctl restart apache2
```

LAMP stack is now installed!

See the above tutorial section that includes the `apt search php- | less`. This section provides info about installing more php-related packages, which might be useful in the future.

2.3.1 Test PHP

```sh
# In order to test that your system is configured properly for PHP,
# create a very basic PHP script called info.php. In order for Apache
# to find this file and serve it correctly, it must be saved to a very specific directory, which is called the "web root".
sudo nano /var/www/html/info.php
```

Write the following string to info.php:

```php
<?php
phpinfo();
?>
```

info.php is now accessible via http://45.55.39.12/info.php.

Now delete info.php to deny this information to crackers!

```sh
sudo rm /var/www/html/info.php
```

### 3. Set up SSL certificate via Let's Encrypt

Main tutorial: [How To Secure Apache with Let's Encrypt on Ubuntu 18.04](https://www.digitalocean.com/community/tutorials/how-to-secure-apache-with-let-s-encrypt-on-ubuntu-18-04)

Secondary tutorial: [Step 5 - Setting Up Virtual Hosts](https://www.digitalocean.com/community/tutorials/how-to-install-the-apache-web-server-on-ubuntu-18-04#step-5-—-setting-up-virtual-hosts-recommended)

I will use bmoregoods.com for this project.

3.1. Set up virtual hosts file

```sh
# Create the directory for bmoregoods.com as follows,
# using the -p flag to create any necessary parent directories:
sudo mkdir -p /var/www/bmoregoods.com/html

# Next, assign ownership of the directory with the
# $USER environmental variable:
sudo chown -R $USER:$USER /var/www/bmoregoods.com/html

# make sure web roots persmissions are correct, even though they still should be
sudo chmod -R 755 /var/www/bmoregoods.com/

# create simple index.html page
nano /var/www/bmoregoods.com/html/index.html




```
