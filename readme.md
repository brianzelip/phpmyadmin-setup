# Notes on setting up phpMyAdmin on Digital Ocean

first link: [How To Install and Secure phpMyAdmin on Ubuntu 18.04](https://www.digitalocean.com/community/tutorials/how-to-install-and-secure-phpmyadmin-on-ubuntu-18-04)

second link: [Initial Server Setup with Ubuntu 18.04](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-18-04)

third link: [How to Connect to Droplets with SSH](https://www.digitalocean.com/docs/droplets/how-to/connect-with-ssh/)

fourth link, first action taken, occurred while spinning up the droplet's initial configuration: [How to Upload SSH Public Keys to a DigitalOcean Account](https://www.digitalocean.com/docs/droplets/how-to/add-ssh-keys/to-account/)

## actual steps in practice

1. Add ssh key ('brian-mbair', this key is machine-dependent) via droplet config form, [How to Upload SSH Public Keys to a DigitalOcean Account](https://www.digitalocean.com/docs/droplets/how-to/add-ssh-keys/to-account/)

2. ssh into droplet - [Initial Server Setup with Ubuntu 18.04](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-18-04)

```sh
ssh root@$IP_ADDRESS
```

After accpeting the initial fingerprint warning, I was logged right in since I added the ssh key during init.

See also this link for what to do if no ssh keys added during init config: [How to Connect to Droplets with SSH](https://www.digitalocean.com/docs/droplets/how-to/connect-with-ssh/).

3. Set up an alternative user account with a reduced scope of influence for day-to-day work

```sh
# add user, then provide password (see .env for password)
adduser brian

# add user to sudo group
usermod -aG sudo brian
```

4. Set up basic firewall

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

5. Enable external access for regular user, where the root account uses SSH key authentication (as opposed to password authentication)

```sh
rsync --archive --chown=brian:brian ~/.ssh /home/brian
```
