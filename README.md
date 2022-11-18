# step
### resources
###### vultr vps
- os (debian 10)
###### namesilo dns
### install necessary software
```
apt-get install -y openssl cron socat curl unzip vim
```
### apply for a certificat
###### using acme.sh
- download acme.sh
```
curl https://get.acme.sh | sh -s email=my@example.com
source ~/.bashrc
```
- verify
```
acme.sh --issue -d domainname --standalone -k ec-256
# first try, timeout, so I change the default ca org
acme.sh --set-default-ca --server letsencrypt
# second try, Timeout during connect (likely firewall problem), so I disable the firewall, and try again, then success.
ufw disable
```
- install
```
acme.sh --installcert -d domainname --fullchainpath /etc/ssl/private/domainname.crt --keypath /etc/ssl/private/domainname.key --ecc
chmod 755 /etc/ssl/private/*
```
- auto upgrade for acme.sh
```
acme.sh --upgrade --auto-upgrade
```
### set up fake website
###### using nginx and static template website
- install nginx
```
apt update && apt install nginx -y
```
- create a dir for website
```
mkdir -p /var/www/website/html
```
- download a template (https://templated.co/)
```
wget -O web.zip --no-check-certificate url
```
- unzip to dir setted before
```
unzip -o -d /var/www/website/html web.zip
```
- modify nginx.conf, be careful about some include in default config
```
vim /etc/nginx/nginx.conf
server {
  listen 80;
  server_name domain;
  root /var/www/website/html;
  return 301 https://$http_host$request_uri;
}
server {
  listen 127.0.0.1:8080;
  root /var/www/website/html;
  index index.html;
  add_header Strict-Transport-Security "max-age=63072000" always;
}
```
### install xray
- download and install
```
bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install
```
- get an uuid
```
cat /proc/sys/kernel/random/uuid
```
- modify config.json
```
vim /usr/local/etc/xray/config.json
{
    "log": {
        "loglevel": "debug",
        "access": "/var/log/xray/access.log",
        "error": "/var/log/xray/error.log"
    },

    "inbounds": [
        {
            "port": 443,
            "protocol": "vless",
            "settings": {
                "clients": [
                    {
                        "id": "UUID", 
                        "flow": "xtls-rprx-direct",
                        "level": 0,
                        "email": "333@ffff.com"
                    }
                ],
                "decryption": "none",
                "fallbacks": [
                    {
                        "dest": 8080
                    }
                ]
            },
            "streamSettings": {
                "network": "tcp",
                "security": "xtls",
                "xtlsSettings": {
                    "alpn": [
                        "http/1.1"
                    ],
                    "certificates": [
                        {
                            "certificateFile": "/etc/ssl/private/domain.crt",
                            "keyFile": "/etc/ssl/private/domain.key"
                        }
                    ]
                }
            }
        }
    ],
       
  "outbounds": [
        {
            "protocol": "freedom"
        }
    ]
}
```
### start and start automatically when power on
```
systemctl start xray
systemctl start nginx
systemctl enable xray
systemctl enable nginx
```
### apply certification automatically
```
vi /etc/ssl/private/xray-cert-renew.sh
```
```
#!/bin/bash

.acme.sh/acme.sh --install-cert -d a-你的域名 --ecc --fullchain-file  /etc/ssl/private/你的域名.crt --key-file  /etc/ssl/private/你的域名.key
echo "Xray Certificates Renewed"
       
chmod +r /etc/ssl/private/你的域名.key
echo "Read Permission Granted for Private Key"

sudo systemctl restart xray
echo "Xray Restarted"
```
```
chmod +x /etc/ssl/private/xray-cert-renew.sh
crontab -e
0 1 1 * *   bash /etc/ssl/private/xray-cert-renew.sh
```
### bbr
- 修改系统变量
```
echo "net.core.default_qdisc=fq" >> /etc/sysctl.conf
echo "net.ipv4.tcp_congestion_control=bbr" >> /etc/sysctl.conf
```
- 保存生效
```
sysctl -p
```
- 查看内核是否已开启BBR
```
sysctl net.ipv4.tcp_available_congestion_control
```
- 显示以下即已开启：
```
net.ipv4.tcp_available_congestion_control = reno cubic bbr
```
- 查看BBR是否启动
```
lsmod | grep bbr
```
- 显示返回值即启动成功：
```
tcp_bbr 20480 21
# 返回值有 tcp_bbr 模块即说明 bbr 启动。
```
