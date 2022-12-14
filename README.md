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

.acme.sh/acme.sh --install-cert -d a-???????????? --ecc --fullchain-file  /etc/ssl/private/????????????.crt --key-file  /etc/ssl/private/????????????.key
echo "Xray Certificates Renewed"
       
chmod +r /etc/ssl/private/????????????.key
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
- ??????????????????
```
echo "net.core.default_qdisc=fq" >> /etc/sysctl.conf
echo "net.ipv4.tcp_congestion_control=bbr" >> /etc/sysctl.conf
```
- ????????????
```
sysctl -p
```
- ???????????????????????????BBR
```
sysctl net.ipv4.tcp_available_congestion_control
```
- ???????????????????????????
```
net.ipv4.tcp_available_congestion_control = reno cubic bbr
```
- ??????BBR????????????
```
lsmod | grep bbr
```
- ?????????????????????????????????
```
tcp_bbr 20480 21
# ???????????? tcp_bbr ??????????????? bbr ?????????
```
### WARP
- check netflix
```
#???????????????https://github.com/sjlleo/netflix-verify

#????????????????????????
wget -O nf https://github.com/sjlleo/netflix-verify/releases/download/v3.1.0/nf_linux_amd64 && chmod +x nf

#??????
./nf

#??????????????????
./nf -proxy socks5://127.0.0.1:40000
```
- install warp
```
#???????????????https://pkg.cloudflareclient.com/install

#??????WARP??????GPG ?????????
curl https://pkg.cloudflareclient.com/pubkey.gpg | sudo gpg --yes --dearmor --output /usr/share/keyrings/cloudflare-warp-archive-keyring.gpg

#??????WARP??????
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/cloudflare-warp-archive-keyring.gpg] https://pkg.cloudflareclient.com/ $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/cloudflare-client.list

#??????APT?????????
apt update

#??????WARP???
apt install cloudflare-warp

#??????WARP???
warp-cli register

#????????????????????????????????????????????????
warp-cli set-mode proxy

#??????WARP???
warp-cli connect

#??????????????????IP?????????
curl ifconfig.me --proxy socks5://127.0.0.1:40000
```
### modify xray configuration
- outbounds
```
    {
      "tag": "netflix_proxy",
      "protocol": "socks",
      "settings": {
        "servers": [
          {
            "address": "127.0.0.1",
            "port": 40000
          }
        ]
      }
    }
```
- routing.rules
```
  "routing": {
    "rules": [
      {
        "type": "field",
        "outboundTag": "netflix_proxy",
        "domain": [
          "geosite:netflix",
          "geosite:disney"
        ]
      }
    ]
  }
```
