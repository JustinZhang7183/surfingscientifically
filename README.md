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
acme.sh --issue -d domainname --standalone
# first try, timeout, so I change the default ca org
acme.sh --set-default-ca --server letsencrypt
# second try, Timeout during connect (likely firewall problem), so I disable the firewall, and try again, then success.
ufw disable
```
- install
```
acme.sh --install-cert -d domainname --fullchainpath /etc/ssl/private/domainname.crt --keypath /etc/ssl/private/domainname.key --ecc
chmod 755 /etc/ssl/private
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
