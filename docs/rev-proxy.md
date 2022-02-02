---
id: reverse-proxy
title:  Putting peppermint behind a reverse proxy
slug: /proxy
---

### Why

Sometimes, individuals & organisations like to put applications behind a reverse proxy, the main reason for this is to allow applications to be accessed on specific domains.

An example of this would be a helpdesk company running peppermint having it accessible to employee's through their domain name.

```
https://support.example.com
```

Behind the scenes you'll most likely use nginx, trafik or haproxy to achieve this goal. In this example we will be using nginx on debian.

## Setting up the box & nginx

In this example, I'm going to be using a node provided by linode pre-installed with docker, but any vm should be able to achieve this provided they have a static i.p address. 
I wont be going into any detail regarding secuirty or best practices for setting up a linux machine, or we'll be here all day :) 
This is going to be a straight forward, down to business guide.

With docker pre installed, we need to get the second piece of the puzzle installed, which is nginx.

Since this our first interaction which the machine, we'll need to update the package manager to get the latest listings:

```
sudo apt update
```

After this we can installed nginx

```
sudo apt install nginx
```

Before testing Nginx, the firewall software needs to be adjusted to allow access to the service.

List the application configurations that ufw knows how to work with by typing:

```
sudo ufw app list
```

This should show the list of application profiles
```
Output
Available applications:
...
  Nginx Full
  Nginx HTTP
  Nginx HTTPS
...
```

As you can see, there are three profiles available for Nginx:

 - Nginx Full: This profile opens both port 80 (normal, unencrypted web traffic) and port 443 (TLS/SSL encrypted traffic)
 - Nginx HTTP: This profile opens only port 80 (normal, unencrypted web traffic)
 - Nginx HTTPS: This profile opens only port 443 (TLS/SSL encrypted traffic)

It is recommended that you enable the most restrictive profile that will still allow the traffic you’ve configured. Since we haven’t configured SSL for our server yet in this guide, we will only need to allow traffic for HTTP on port 80.

Lets enable that now

```
sudo ufw allow 'Nginx HTTP'
```

### Checking nginx 

To make sure nginx is running okay when can run the command:

```
systemctl status nginx
```

Which should display 

```
Output
● nginx.service - A high performance web server and a reverse proxy server
   Loaded: loaded (/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
   Active: active (running) since Wed 2019-07-03 12:52:54 UTC; 4min 23s ago
     Docs: man:nginx(8)
 Main PID: 3942 (nginx)
    Tasks: 3 (limit: 4719)
   Memory: 6.1M
   CGroup: /system.slice/nginx.service
           ├─3942 nginx: master process /usr/sbin/nginx -g daemon on; master_process on;
           ├─3943 nginx: worker process
           └─3944 nginx: worker process
```

Another way you can test this is working is by going to:

```
http://your_server_ip
```

And if everything is good you should see the welcome to nginx sign.

## Setting up peppermint 

Now nginx is out the way and configured, we can now simply download the docker compose file from github:

```
wget https://raw.githubusercontent.com/Peppermint-Lab/Peppermint/master/docker-compose.yml
```

After this we should be able to run:
```
nano docker-compose.yml
```

and we'll be presented with an output like so:
```
version: "3.1"

services:
  postgres:
    container_name: postgres
    image: postgres:latest
    restart: always
    volumes:
      - ./peppermint/db:/data/db
    environment: 
      POSTGRES_USER: peppermint
      POSTGRES_PASSWORD: 1234
      POSTGRES_DB: peppermint

  client:
    container_name: peppermint
    image: pepperlabs/peppermint:latest
    ports:
      - 5001:5001
    restart: on-failure
    depends_on:
      - postgres
    environment:
      PORT: 5001
      DB_USERNAME: "peppermint"
      DB_PASSWORD: "1234"
      DB_HOST: "postgres"
      BASE_URL: "http://support.example.com"
```

Now for your base url: this is where your subdomain url is going to, if youre using nginx https with SSL, then make sure you change it accordingly. 

After you enter the correct base url hit ```ctrl + x``` and then ```y``` to save. When this is complete run the command: 

```
docker-compose up -d
```

This will pull the peppermint image & postgres and start the process of both containers. Once up, we have one finally more nginx config to take care of.


### Nginx config

Now that nginx is set up & both are containers are working, we can now implement the config file which is going to route our proxy to our subdomain. 

I like to save this in the conf.d folder of nginx, it works for me and i never run into issues.

Start off by running:

```
nano /etc/nginx/conf.d
```

This will bring an editor up, in which you will paste

```
server {
    listen 80;
    listen [::]:80;
    server_name support.example.com;
    add_header Strict-Transport-Security "max-age=15552000; includeSubDomains" always;

    location / {
        proxy_pass http://127.0.0.1:5001;
        proxy_next_upstream error timeout invalid_header http_500 http_502 http_503;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forward-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_redirect off;
        proxy_read_timeout 5m;
    }
    client_max_body_size 10M;
}
```

Replace the server name with your url of choice, including subdomain and procced to save the file as

```
peppermint.conf
```

### Restarting nginx

The final touch to the tale is you need to restart nginx for this to take full effect.

This can be achieved by running:

```
systemctl restart nginx
```

You should now be able to see peppermint running on your choosen subdomain.

I hope you found this guide usual :) 