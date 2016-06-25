---
title: "How to install Ghost blogging platform on Ubuntu"
slug: "how-to-install-ghost-blogging-platform-on-ubuntu"
category: Community
date: 2016-06-25 15:28 +05:45
tags: "ghost, nodejs, blog"
---

This tutorial explains how to setup Ghost blogging platform on a standard Ubuntu 16.04 template from the official Exoscale bare instance.

The barebones Exoscale template for Ubuntu is not configured with a normal non-root user. Let's add a new user and add this user to `sudoers`.

    adduser ghost
    usermod -aG sudo ghost

Now, login as this new user or just do:
    
    su ghost
    
All commands specified below should now be executed as this user.

## Refresh repository and upgrade system

    sudo apt-get update && apt-get upgrade -y

## Install Node.js, npm and nginx

Ghost requires Node.js to run. It depends on many Node.js packages which can be installed using `npm` package manager. We will serve the blog via nginx that is setup as a reverse proxy to the running Ghost instance.

    sudo apt-get install nodejs npm nodejs-legacy nginx

Node.js legacy package is required as `node` command is not available by default and many package will depend on `node` comamnd instead of `nodejs`.

## Step 3 : Install PM2 and Grunt
PM2 is a process manager for Node.js applications. We will use PM2 to make sure that our Ghost blog runs forever as a service. Install it using:
    
    sudo npm install pm2 -g


Install Grunt for building Ghost:

    sudo npm install -g grunt-cli
    
## Get the latest stable version of Ghost

    mkdir ~/webapps/
    cd ~/webapps/
    git clone -b stable git://github.com/tryghost/ghost.git

## Install Node.js packages required for Ghost

    cd ~/webapps/ghost
    npm install --production

If this process ends with `killed` in the last line, it is mainly because the memory was low and this process was killed. For this one-time installation task, we can temporarily create and use a swap space with:

    sudo fallocate -l 1G /swapfile
    sudo chmod 600 /swapfile
    sudo mkswap /swapfile
    sudo swapon /swapfile

Now, retry `npm install --production` after these commands.

## Build Ghost

To build Ghost for production, do this:
    
    grunt init && grunt prod

## Configuration

Copy the sample configuration to `config.js` and modify it as needed.

    cp config.example.js config.js

An important variable in this file is `url` in `production` section. Replace it with your url where you intend to make your blog available.

## Configure pm2 to start Ghost

Run this:

    NODE_ENV=production pm2 start index.js --name "Ghost"

Save this command in PM2's startup list with:

    pm2 save

To make this permanent and persist upon reboot, run:

    pm2 startup ubuntu

It will give a command to execute as root like `sudo su -c "env PATH=$PATH:/usr/bin pm2 startup ubuntu -u ghost --hp /home/ghost`. The exact command may vary. Run the supplied command and PM2 will reload Ghost on boot.

## Nginx configuration

Remove the default Nginx site configuration with:

    sudo rm /etc/nginx/sites-enabled/default
    
Create a new file, say `ghost`, inside `/etc/nginx/sites-available/` with the following contents:

    server {
        listen 0.0.0.0:80;
        server_name your-domain-name;
        access_log /var/log/nginx/your-domain-name.log;
    
        location / {
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header HOST $http_host;
            proxy_set_header X-NginX-Proxy true;
            proxy_pass http://127.0.0.1:2368;
            proxy_redirect off;
        }
    }
    
Be sure to replace `your-domain-name` with your domain name.

Add it to `sites-enabled` using:

    ln -s /etc/nginx/sites-available/ghost /etc/nginx/sites-enabled/ghost

Now restart the nginx process using:
    
    service nginx restart

Your site should now be live.

You many need to add a new `ingress` rule in your firewall for port 80 (TCP). You can learn more about it [here](https://community.exoscale.ch/documentation/compute/security-groups/).






