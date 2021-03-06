# Notes on setting up phpMyAdmin on Digital Ocean

## Resources via initial web search, "install phpmyadmin on digital ocean"

- first link: [How To Install and Secure phpMyAdmin on Ubuntu 18.04](https://www.digitalocean.com/community/tutorials/how-to-install-and-secure-phpmyadmin-on-ubuntu-18-04)

- second link: [Initial Server Setup with Ubuntu 18.04](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-18-04)

- third link: [How to Connect to Droplets with SSH](https://www.digitalocean.com/docs/droplets/how-to/connect-with-ssh/)

- fourth link, first action taken, occurred while spinning up the droplet's initial configuration: [How to Upload SSH Public Keys to a DigitalOcean Account](https://www.digitalocean.com/docs/droplets/how-to/add-ssh-keys/to-account/)

- fifth link: [How To Install Linux, Apache, MySQL, PHP (LAMP) stack on Ubuntu 18.04](https://www.digitalocean.com/community/tutorials/how-to-install-linux-apache-mysql-php-lamp-stack-ubuntu-18-04)

## Actual steps in practice

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
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY $MYSQL_ALTER_ROOT_USER_PASSWORD;

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

info.php is now accessible via [http://45.55.39.12/info.php](http://45.55.39.12/info.php).

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

At this point, I cirucumvented the tutorial's chronology, and configured DNS settings to point bmoregoods.com and www.bmoregoods.com to \$IP_ADDRESS. So onto section 3.1.1...

3.1.1. Configure DNS settings for bmoregoods.com

Useful links for this section:

- [How To Point to DigitalOcean Nameservers From Common Domain Registrars](https://www.digitalocean.com/community/tutorials/how-to-point-to-digitalocean-nameservers-from-common-domain-registrars)

- [How to Create DNS Records](https://www.digitalocean.com/docs/networking/dns/how-to/manage-records/) (_very useful for understanding DNS records in general_!)

I set up two A records via the droplet dashboard, instead of an A record and a CNAME record like GoDaddy does it. This is becuase the tutorial "How To Secure Apache with Let's Encrypt on Ubuntu 18.04" lists a preqrequisite of:

- An A record with example.com pointing to your server's public IP address.
- An A record with www.example.com pointing to your server's public IP address.

Here were the DNS steps I took:

1. On GD: disabled the forwarding to bmoregoods.org
2. On GD: updated the name servers for this domain (from GD to DO name servers)
3. Added 2 new A records to the droplet via the droplet dashboard

3.1.2 Now back to configuring the new virtual hosts file

```sh
# Instead of modifying the default configuration, create a new
# virtual host file with the correct directives.
# (see tutorial for virtual host file content)
sudo nano /etc/apache2/sites-available/bmoregoods.com.conf

# Let's enable the file with the a2ensite tool:
sudo a2ensite bmoregoods.com.conf

# Disable the default site defined in 000-default.conf:
sudo a2dissite 000-default.conf

# Next, let's test for configuration errors:
sudo apache2ctl configtest

# Restart Apache to implement your changes:
sudo systemctl restart apache2
```

**UPDATE**: SHIT WORKS!

![screenshot of initial bmoregoods.com rendered as part of this work](https://i.imgur.com/jiPs5c2.png)

The virtual hosts file tutorial lists a section at the end that is useful for general learning about Apache. Including it in this repo for later reading, see about-apache.md herein.

3.2. Install and configure certbot

Now, after the virtual host file and DNS pivots above, we're back on track for setting up https, so back to the "How To Secure Apache with Let's Encrypt on Ubuntu 18.04" tutorial.

```sh
# add certbot repo specifically made for Ubuntu, via certbot maintainers
sudo add-apt-repository ppa:certbot/certbot

# Install Certbot's Apache package with apt:
sudo apt install python-certbot-apache

# obtain SSL cert
sudo certbot --apache -d bmoregoods.com -d www.bmoregoods.com
```

On the interactive script that resulted after the last command above, I chose option 2, which allows redirects from http to https.

It worked, I have now conigured Let's Encrypt 🎉

Here is the confirmation message, which includes an import note about renewal!

> IMPORTANT NOTES:
>
> - Congratulations! Your certificate and chain have been saved at:
>   /etc/letsencrypt/live/bmoregoods.com/fullchain.pem
>   Your key file has been saved at:
>   /etc/letsencrypt/live/bmoregoods.com/privkey.pem
>   **Your cert will expire on 2019-07-15**. To obtain a new or tweaked
>   version of this certificate in the future, simply run certbot again
>   with the "certonly" option. To non-interactively renew _all_ of
>   your certificates, run "certbot renew"
>
> - Your account credentials have been saved in your Certbot
>   configuration directory at /etc/letsencrypt. You should make a
>   secure backup of this folder now. This configuration directory will
>   also contain certificates and private keys obtained by Certbot so
>   making regular backups of this folder is ideal.

**UPDATE on SSL**: ALAS! The next section in the tutorial covers certbot auto renewal!

3.2.1. Verify Certbot auto-renewal

```sh
# verify certbot auto-renewal via --dry-run
sudo certbot renew --dry-run
```

Everything relating to renewing the SSL cert should be set up. Here is the tutorial's final about this:

> If you see no errors, you're all set. When necessary, Certbot will renew your certificates and reload Apache to pick up the changes. If the automated renewal process ever fails, Let’s Encrypt will send a message to the email you specified, warning you when your certificate is about to expire.

ps - bmoregoods.com received an A grade form [SSL Labs Server Test](https://www.ssllabs.com/ssltest/)!

### 4. Install phpMyAdmin

Main tutorial: [How To Install and Secure phpMyAdmin on Ubuntu 18.04](https://www.digitalocean.com/community/tutorials/how-to-install-and-secure-phpmyadmin-on-ubuntu-18-04)

4.1. Install packages

```sh
# install phpmyadmin related packages
#   see $MYSQL_APPLICATION_PASSWORD_FOR_PHPMYADMIN
sudo apt install phpmyadmin php-mbstring php-gettext

# explicitly enable the mbstring PHP extension
sudo phpenmod mbstring

# restart Apache to init changes above
sudo systemctl restart apache2
```

4.2. Adjust User Authentication and Privileges

FYI: the ALTER_USER from installing MySQL in section 2.2. above is the user that \*I\* log in with to create new users, so use \$MYSQL_ALTER_ROOT_USER_PASSWORD.

```sql
# Add user abbie
CREATE USER 'abbie'@'localhost' IDENTIFIED BY $MYSQL_ABBIE_USER_PASSWORD;

# grant abbie privilege to all tables within the database, as well as the power to add, change, and remove user privileges, with this command
GRANT ALL PRIVILEGES ON *.* TO 'abbie'@'localhost' WITH GRANT OPTION;

exit
```

### 5. LOG IN! 🎉

[bmoregoods.com/phpmyadmin](https://bmoregoods.com/phpmyadmin/) 🚢
