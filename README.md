# iblack.sexy
Shortcut to [NTNU Blackboard](https://innsida.ntnu.no/blackboard)

## Purpose
The purpose of iblack.sexy is to provide a convenient way to access NTNU's Blackboard portal.
Blackboard is an online tool to manage courses at educational insitutions.

# Setup

## Resources
The guide is partly written by following these guides from [Digital Ocean](https://www.digitalocean.com/):
- https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-18-04
- https://www.digitalocean.com/community/tutorials/how-to-install-nginx-on-ubuntu-18-04
- https://www.digitalocean.com/community/tutorials/how-to-create-temporary-and-permanent-redirects-with-nginx
- https://www.digitalocean.com/community/tutorials/how-to-configure-logging-and-log-rotation-in-nginx-on-an-ubuntu-vps

## Initial work
Create a droplet on digitalocean.com. The cheapest instance will be enough to handle the load of our service. This guide assumes the droplet is running Ubuntu 18.04.

Create a private/public key pair, to be used to access the server. Copy the public key to the droplet:
```bash
ssh-copy-id -i iblack.sexy.pub root@your_server_ip
```

## Create a non-root user with sudo access
In this section we assume you are accessing the droplet through `ssh`, and using the default `root` user.

Create local user and follow the wizard for creating a new user called `iblack`:
```shell
# adduser iblack
```

Add the user to the sudo group:
```shell
# usermod -aG sudo iblack
```

## Configure firewall
Allow `ssh` connections through the firewall:
```shell
# ufw allow OpenSSH
# ufw enable
```

We want to be able to use the same private key we used to access `root` to access the user `iblack`.
Copy the `authorized_keys` file for `root` into the `~/.ssh` folder of `iblack`:
```shell
# rsync --archive --chown=iblack:iblack ~/.ssh /home/iblack
```

You can now login as `iblack` with the previously generated private key.
```shell
ssh iblack@your_server_ip
```

## Install nginx

From now on, we assume that you are logged in with the user `iblack`, and not as the `root` user.

Install `nginx`:
```shell
sudo apt update
sudo apt upgrade
sudo apt install nginx
```

We now need to adjust the firewall to accept incoming traffic. We only want to let traffic through on port 80 (HTTP). We do not need to open port 443 (HTTPS).

Check that the presets for nginx are installed:
```shell
sudo ufw app list
``` 

It should yield the following:
```
Available applications:
  Nginx Full
  Nginx HTTP
  Nginx HTTPS
  OpenSSH
```

Allow "Nginx HTTP":
```shell
sudo ufw allow 'Nginx HTTP'
```

`nginx` should now be serving the standard splash page at your server's IP, and be configured to start on reboot.

To check the status of `nginx` you can issue the following command:
```shell
systemctl nginx status
```

You can test if the `nginx` service start on boot by rebooting...
```shell
sudo reboot
```

... and then logging in again and checking the status of `nginx`:
```shell
systemctl nginx status
```

If `nginx` doesn't start on boot, you can enable it with:
```shell
sudo systemctl enable nginx
```

## Configure nginx

Create an `nginx` server block:
```shell
sudo vim /etc/nginx/sites-available/iblack.sexy
```

Enter the following config:
```
server {
        listen 80 default_server;
        listen [::]:80 default_server;

        server_name .iblack.sexy;

        rewrite ^/(.*)$ https://innsida.ntnu.no/blackboard redirect;
}
```

Create a symlink to enable the site:
```shell
sudo ln -s /etc/nginx/sites-available/iblack.sexy /etc/nginx/sites-enabled/iblack.sexy
```

Remove the default site symlink:
```shell
sudo rm /etc/nginx/sites-enabled/default
```

Restart nginx to enable the new config:
```shell
sudo systemctl restart nginx
```

## Adding Host to nginx access log

If more than one domain is hosted on the same server, it is beneficial to be able to distinguish between the requests in the access log. In order to add host to the logs, add the following config in the `html` block in the nginx config at `/etc/nginx/nginx.conf`:
```shell
log_format with_host '$remote_addr - $remote_user [$time_local] "$host" '
                           '"$request" $status $body_bytes_sent '
                           '"$http_referer" "$http_user_agent"';
```

Then modify the default logging settings by specifying the new log format:
```shell
access_log /var/log/nginx/access.log with_host;
error_log /var/log/nginx/error.log with_host;
```

## Configure logging with logrotate

Edit the nginx logrotate config file:
```shell
sudo vim /etc/logrotate.d/nginx
```

Enter the following config:
```
/var/log/nginx/*.log {
        monthly
        missingok
        rotate 24
        compress
        delaycompress
        notifempty
        create 0640 www-data adm
        sharedscripts
        prerotate
                if [ -d /etc/logrotate.d/httpd-prerotate ]; then \
                        run-parts /etc/logrotate.d/httpd-prerotate; \
                fi \
        endscript
        postrotate
                invoke-rc.d nginx rotate >/dev/null 2>&1
        endscript
}
```

Logs are now rotated monthly, and stored at `/var/log/nginx/`.
To get some interesting stats you can try the following regex's:
```
sudo cat /var/log/nginx/access.log | grep -o '[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}.*01\/Apr' | uniq | wc -l
sudo cat /var/log/nginx/access.log | grep -o '[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}.*[0-9]\{2\}\/May' | uniq | wc -l
```
