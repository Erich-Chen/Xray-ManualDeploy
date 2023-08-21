# Xray Manual Deploy
A Parody of https://github.com/wulabing/Xray_onekey, to note down my manual deployment of Xray VLESS with Nginx Forward.

Only verfified on Debian 12, but should easy to fit on more distros.

## 1. Prepare a VPS

**Prerequisitions**

* A VPS with IPv4 address.

* Debian 12. (Or any distro you like. This is a manually deployment that easily fit for other distro.)

* The A-record (assuming xxx.yyy.zzz) to the IPv4 address of VPS. 

* Firewall is not considered (assumed off).

**Basic Setups**

````bash
# add a sudo user `xray`
adduser xray
usermod -aG sudo xray

# Log out and log in again as new user 'xray'
# Or just switch user by `su -l xray`

sudo apt update && sudo apt upgrade -qy

# Install some might used software, NOT all of the following software are manditory.
sudo apt install vim git curl wget lsof tar cron unzip jq tmux -y
````

## 2. Install Nginx Server

````bash
sudo apt update

sudo apt install nginx -y

# Allow firewall for both http and https traffic
# sudo ufw allow 'Nginx Full'

# Performance improvement
# uncomment the following line in file /etc/nginx/nginx.conf
# server_names_hash_bucket_size 64;
sudo sed -i "/^#\ server_names_hash_bucket_size/server_names_hash_bucket_size" /etc/nginx/nginx.conf
````

**Explanation on some Nginx Direcotries and Files**

* `/var/www/html/` : Actual web content. New web content will be added into new directory like `/var/www/xxx.yyy.zzz/html/`.
* `/etc/nginx/nginx.conf` : the main configuraiton file.
* `/etc/nginx/sites-available/` : Per-site *server blocks*. Only used when one is linked into `sites-enabled`.
* `/etc/nginx/sites-enabled/` : Per-site *site blocks*, typically softlinks from `sites-availabe`.

**A dummy website**

````bash
myurl='xxx.yyy.zzz'
sudo mkdir -p /var/www/$myurl/html
sudo chown -R $USER:$USER /var/www/$myurl/html
sudo chmod -R 755 /var/www/$myurl

# Create a very dummy sample website
cat << EOL | tee /var/www/$myurl/html/index.html
<html>
    <head>
        <title>My Website</title>
    </head>
    <body>
        <h1>Success! Seeing this page means my Nginx server is successfully configured for <em>$myurl</em>. </h1>
        <p>This is a sample page.</p>
    </body>
</html>

EOL

# Create a per-site server block
# Define variable uri into '$uri' to avoid using escape in Line 10.
uri='$uri'
cat << EOL | sudo tee /etc/nginx/sites-available/$myurl
server {
        listen 80;
        listen [::]:80;

        root /var/www/$myurl/html;
        index index.html index.htm index.nginx-debian.html;

        server_name $myurl;

        location / {
                try_files $uri $uri/ =404;
        }
}

EOL

# link to sites-available to make it enabled
sudo ln -s /etc/nginx/sites-available/$myurl /etc/nginx/sites-enabled/

# Syntax check
sudo nginx -t

# Expected outputs:
# nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
# nginx: configuration file /etc/nginx/nginx.conf test is successful

sudo systemctl restart nginx

# Repeat this step to build more websites
````



## 3. Secure Nginx with Let's Encrypt Certificate (certbot)



````bash
sudo apt install certbot python3-certbot-nginx -y
sudo systemctl reload nginx

myurl='xxx.yyy.zzz'
sudo certbot --nginx -d $myurl
````

**Sample Output**

````
Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/xxx.yyy.zzz/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/xxx.yyy.zzz/privkey.pem
This certificate expires on YYYY-MM-DD.
These files will be updated when the certificate renews.
Certbot has set up a scheduled task to automatically renew this certificate in the background.

Deploying certificate
Successfully deployed certificate for xxx.yyy.zzz to /etc/nginx/sites-enabled/xxx.yyy.zzz
Congratulations! You have successfully enabled HTTPS on https://xxx.yyy.zzz.
````

**Check certbot auto renew status**
````bash
# check certbot auto renew status

sudo systemctl status certbot.timer
sudo certbot renew --dry-run
````

## 4. Install Xray-Core

````bash
# Install / Upgrade X-ray core
sudo bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install
````

**Sample Output**

````
installed: /usr/local/bin/xray
installed: /usr/local/share/xray/geoip.dat
installed: /usr/local/share/xray/geosite.dat
installed: /usr/local/etc/xray/config.json
installed: /var/log/xray/
installed: /var/log/xray/access.log
installed: /var/log/xray/error.log
installed: /etc/systemd/system/xray.service
installed: /etc/systemd/system/xray@.service
info: Xray v<x.y.z> is installed.
````

## 5. Configure VLESS with Nginx Forward

**Performance Improvement**

````bash
# performance improvement
sed -i '/^\*\ *soft\ *nofile\ *[[:digit:]]*/d' /etc/security/limits.conf
sed -i '/^\*\ *hard\ *nofile\ *[[:digit:]]*/d' /etc/security/limits.conf
echo '* soft nofile 65536' >>/etc/security/limits.conf
echo '* hard nofile 65536' >>/etc/security/limits.conf

````

**Create `config.json` file for Xray**

````bash
# Create config.json file for Xray
# The following is a copy from https://raw.githubusercontent.com/wulabing/Xray_onekey/nginx_forward/config/xray_tls_ws.json
# Must define the values in the following lines
port='10086'                                           # L19
uuid='3f3effce-2640-4f29-b95b-a2106df6d96d'            # L26, uuid can be generated by command `xray uuid`
path='/e01ec5ea/'                                      #L35
#
#
#
cat << EOL | sudo tee /usr/local/etc/xray/config.json
{
  "log": {
    "access": "/var/log/xray/access.log",
    "error": "/var/log/xray/error.log",
    "loglevel": "warning"
  },
  "inbounds": [
    {
      "port": $port,
      "listen": "127.0.0.1",
      "tag": "VLESS-in",
      "protocol": "VLESS",
      "settings": {
        "clients": [
          {
            "id": "$uuid",
            "alterId": 0
          }
        ],
        "decryption": "none"
      },
      "streamSettings": {
        "network": "ws",
        "wsSettings": {
          "path": "$path"
        }
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "freedom",
      "settings": {},
      "tag": "direct"
    },
    {
      "protocol": "blackhole",
      "settings": {},
      "tag": "blocked"
    }
  ],
  "dns": {
    "servers": [
      "https+local://1.1.1.1/dns-query",
      "1.1.1.1",
      "1.0.0.1",
      "8.8.8.8",
      "8.8.4.4",
      "localhost"
    ]
  },
  "routing": {
    "domainStrategy": "AsIs",
    "rules": [
      {
        "type": "field",
        "inboundTag": [
          "VLESS-in"
        ],
        "outboundTag": "direct"
      }
    ]
  }
}

EOL
````



**Configure nginx forward for Xray**

````bash
# configure nginx forward for Xray
# The following is a combination of standard nginx server block and wulabing's
# https://raw.githubusercontent.com/wulabing/Xray_onekey/nginx_forward/config/web.conf
# Must define variables for the values in the following lines:
myurl='xxx.yyy.zzz'             # multiple places, L20, L26, L27, L33, L35, L66, L68
path='/e01ec5ea/'               # L49
port='10086'                    # L51
#
# Variables to escape the $ symbol in tee output
uri='$uri'
remote_addr='$remote_addr'
proxy_add_x_forwarded_for='$proxy_add_x_forwarded_for'
http_upgrade='$http_upgrade'
http_host='$http_host'
ssl_early_data='$ssl_early_data'
host='$host'
request_uri='$request_uri'
#
sudo cp /etc/nginx/sites-available/$myurl{,.bak}
cat << EOL | sudo tee /etc/nginx/sites-available/$myurl
server {

    listen 443 ssl http2;   # modified for xray
    listen [::]:443 ssl http2 ipv6only=on; # managed by Certbot, modified for xray

    ssl_certificate        /etc/letsencrypt/live/$myurl/fullchain.pem; # managed by Certbot
    ssl_certificate_key    /etc/letsencrypt/live/$myurl/privkey.pem; # managed by Certbot
    include                /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam            /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
    # ssl_protocols          TLSv1.3; # Added for xray but already included in /etc/letsencrypt/options-ssl-nginx.conf
    ssl_ecdh_curve         X25519:P-256:P-384:P-521; # added for xray

    server_name $myurl;

    root /var/www/$myurl/html;
    index index.html index.htm index.nginx-debian.html;

    # Config for 0-RTT in TLSv1.3, added for xray
    ssl_early_data on;
    ssl_stapling on;
    ssl_stapling_verify on;
    add_header Strict-Transport-Security "max-age=63072000" always;

	location / {
        try_files $uri $uri/ =404;
    }

	# Added for xray
	location $path {
        proxy_redirect off;
        proxy_pass http://127.0.0.1:$port;
        proxy_http_version 1.1;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $http_host;
        # Config for 0-RTT in TLSv1.3
        proxy_set_header Early-Data $ssl_early_data;
        }
}

server {
    listen 80;
    listen [::]:80;
    server_name $myurl;

	if (\$host = $myurl) {
        return 301 https://$host$request_uri;
    } # managed by Certbot
    return 404; # managed by Certbot
}

EOL
````

````bash
# Restart to take effect
sudo nginx -t
sudo systemctl restart nginx

sudo systemctl restart xray
````