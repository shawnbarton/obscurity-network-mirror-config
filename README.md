# Basic Mirror Configuration

This document serves as a guide to configure a basic nginx HTTP(S) file mirror, for example, https://obscurity.network/).

Wildcard HTTPS certificates, basics such as securing the server, and networking are out of scope.

Base OS: Ubuntu 20.4.2

Additional domain name configuration: Ensure the (sub-)domain has a CAA Record.


1) Install nginx, ssl-cert, rsync

```
sudo apt install nginx ssl-cert rsync
```

2) Install certbot snap package

```
sudo snap install core; sudo snap refresh core
```
```
sudo snap install --classic certbot
```

3) Create our working directories

```
sudo mkdir -p /srv/www-data/
sudo mkdir -p /srv/mirrors/obscurity.network/obscurity/
sudo chown -R www-data: /srv/www-data/
sudo chown -R www-data: /srv/mirrors/
```

4) Configure nginx:
```
sudo vi /etc/nginx/nginx.conf
```

```
Change:
        sendfile on;
To:
        sendfile on;
        sendfile_max_chunk 512k;
```

```
Change:
        sendfile on;
To:
        sendfile on;
        sendfile_max_chunk 512k;
```

```
Replace SSL Settings section:
        ##
        # SSL Settings
        ##

        ssl_protocols TLSv1.2 TLSv1.3; # Only modern TLS
        ssl_prefer_server_ciphers on;
        ssl_stapling on;
        ssl_stapling_verify on;
        ssl_session_cache shared:SSL:60m;
        ssl_session_timeout 1d;
        ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
```


5) Configure nginx sites:

```
sudo vi /etc/nginx/sites-available/generics_unavailable
```
```
server {
        listen 80;
        server_name replace.me.with.wan.ip;

        location / {
                return 444; # "Connection closed without response"
        }
}

server {
        listen 443;
        server_name replace.me.with.wan.ip;

        ssl_certificate     /etc/ssl/certs/ssl-cert-snakeoil.pem;
        ssl_certificate_key /etc/ssl/private/ssl-cert-snakeoil.key;
        ssl_stapling        off;
        ssl_ciphers         NULL;

        location / {
                return 444; # "Connection closed without response"
        }
}

server {
        listen 80 default_server;
        server_name _;

        location / {
                return 444; # "Connection closed without response"
        }
}

server {
        listen 443 default_server;
        server_name _;

        ssl_certificate     /etc/ssl/certs/ssl-cert-snakeoil.pem;
        ssl_certificate_key /etc/ssl/private/ssl-cert-snakeoil.key;
        ssl_stapling        off;
        ssl_ciphers         NULL;

        location / {
                return 444; # "Connection closed without response"
        }
}
```
```
sudo vi /etc/nginx/sites-available/http_redirect
```
```
server {
        listen 80;
        server_name replace.me.with.subdomain.domain.tld;

        return 301 https://$host$request_uri;
}
```
```
sudo vi /etc/nginx/sites-available/mirror_obscurity-network
```
```
server {
        listen 443 ssl http2;

        server_name replace.me.with.subdomain.domain.tld;

        ssl_certificate /etc/letsencrypt/live/replace.me.with.subdomain.domain.tld/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/replace.me.with.subdomain.domain.tld/privkey.pem;
        ssl_trusted_certificate /etc/letsencrypt/live/replace.me.with.subdomain.domain.tld/chain.pem;

        # Optional
        location /robots.txt {
                add_header Content-Type text/plain;
                return 200 "User-agent: *\nDisallow: /\n";
        }

        location / {
                root /srv/www-root/;
                autoindex on;
                autoindex_exact_size off;
                autoindex_format html;
        }

        location /obscurity {
                alias /srv/mirrors/obscurity.network/obscurity/;
                autoindex on;
                autoindex_exact_size off;
                autoindex_format html;
        }
}
```
```
sudo ln /etc/nginx/sites-available/generics_unavailable /etc/nginx/sites-enabled/generics_unavailable
sudo ln /etc/nginx/sites-available/http_redirect /etc/nginx/sites-enabled/http_redirect
sudo nginx -t
sudo systemctl restart nginx
```

Configure certbot and request certificates (you will have to enter the (sub-)domain):
```
sudo certbot --nginx
```

Enable the mirror nginx configuration:
```
sudo ln sudo vi /etc/nginx/sites-available/mirror_obscurity-network /etc/nginx/sites-enabled/mirror_obscurity-network
sudo nginx -t
sudo systemctl restart nginx
```

Create for rsync password file:
```
echo replace.with.credentials > ~/obscurity_pass
sudo chmod 700 ~/obscurity_pass
```

```
sudo vi ~/rsync_cronjob.sh
```
```bash
#!/bin/bash

/usr/bin/rsync --chown=www-data:www-data --stats -vcaP rsync://user@master:port/obscurity /srv/mirrors/obscurity.network/obscurity/ --password-file /home/username/obscurity_pass;

if [ $? -eq 0 ]
then
        echo "$(date) - obscurity.network SYNC SUCCESS" >> /srv/www-root/mirror_sync_log.txt
else
        echo "$(date) - obscurity.network SYNC ERROR" >> /srv/www-root/mirror_sync_log.txt
fi
```

```
chmod +x ~/rsync_cronjob.sh
```
```
sudo crontab -e
```
```
00 2 * * * /home/username/rsync_cronjob.sh >> /home/username/cron_rsync.log 2>&1
```
