# SQUID Proxy Server with SSL Bump

## Installation

Using squid version 5

```bash
apt-get update
apt-get install build-essential openssl libssl-dev pkg-config
wget http://www.squid-cache.org/... .tar.gz
tar -zxvf squid-*
cd squid-*
./configure --prefix=/usr/local/squid --enable-icap-client --enable-ssl --with-openssl --enable-ssl-crtd --with-default-user=squid
make
make install

mkdir /usr/local/squid/etc/ssl_cert -p
chown squid:squid /usr/local/squid/etc/ssl_cert -R
chmod 700 /usr/local/squid/etc/ssl_cert -R
cd /usr/local/squid/etc/ssl_cert
openssl req -new -newkey rsa:2048 -sha256 -days 365 -nodes -x509 -extensions v3_ca -keyout myCA.pem -out myCA.pem
/usr/local/squid/libexec/security_file_certgen -c -s /usr/local/squid/var/logs/ssl_db -M 4MB
chown squid:squid /usr/local/squid/var/logs/ssl_db -R


chown -R proxy:proxy /usr/local/squid -R
/usr/local/squid/sbin/squid -z
```

## squid.conf
```conf
#
# Recommended minimum configuration:
#

# Example rule allowing access from your local networks.
# Adapt to list your (internal) IP networks from where browsing
# should be allowed
acl localnet src 192.168.0.0/16 # RFC1918 possible internal network
acl step1 at_step SslBump1
acl step2 at_step SslBump2
acl step3 at_step SslBump3
acl NoSSLIntercept ssl::server_name_regex "/usr/local/squid/etc/url.nobump"
acl nocache dst 192.168.0.0/16 #Bypass local networks

acl SSL_ports port 443
acl Safe_ports port 80		# http
acl Safe_ports port 21		# ftp
acl Safe_ports port 443		# https
acl Safe_ports port 70		# gopher
acl Safe_ports port 210		# wais
acl Safe_ports port 1025-65535	# unregistered ports
acl Safe_ports port 280		# http-mgmt
acl Safe_ports port 488		# gss-http
acl Safe_ports port 591		# filemaker
acl Safe_ports port 777		# multiling http

acl CONNECT method CONNECT
#http_access allow !Safe_ports
#http_access allow CONNECT !SSL_ports
http_access allow localhost manager
http_access deny manager
http_access allow localnet
http_access allow localhost
http_access allow all
#always_direct allow all
cache deny nocache

# Squid normally listens to port 3128
http_port 192.168.3.4:3127
http_port 192.168.3.4:3128 ssl-bump generate-host-certificates=on dynamic_cert_mem_cache_size=4MB cert=/usr/local/squid/etc/ssl_cert/myCA.pem

sslproxy_cert_error allow all
sslproxy_flags DONT_VERIFY_PEER
sslcrtd_program /usr/local/squid/libexec/security_file_certgen -s /usr/local/squid/var/logs/ssl_db -M 4MB
sslcrtd_children 3

ssl_bump splice NoSSLIntercept
ssl_bump peek step1
ssl_bump bump all

# Uncomment and adjust the following to add a disk cache directory.
cache_dir ufs /var/www/local2/.squid/ 10000 16 256

# Leave coredumps in the first cache dir
coredump_dir /var/www/local2/.squid/coredump

# Add any of your own refresh_pattern entries above these.
refresh_pattern ^ftp:           1440    20%     10080
refresh_pattern ^gopher:        1440    0%      1440
refresh_pattern -i (/cgi-bin/|\?) 0     0%      0

## Aggressively cache everything
refresh_pattern . 10080 9999% 43200 override-expire ignore-reload ignore-no-cache ignore-no-store ignore-must-revalidate ignore-private override-lastmod reload-into-ims store-stale

maximum_object_size 100 MB
minimum_object_size 0 KB
maximum_object_size_in_memory 50 MB
offline_mode off

dns_nameservers 192.168.3.4

error_directory /usr/local/squid/share/errors/en/

```


## Whitelist Broken Sites
/usr/local/squid/etc/url.nobump
```bash
#Whatsapp
(w[0-9]+|[a-z]+)\.web\.whatsapp\.com
 Whatsapp CDN issue
.whatsapp\.net
.whatsapp.net
.whatsapp.com

#Discord
.discord\.com
.discord\.gg
.discord\.media

#Honeygain
.honeygain\.com
(.*\.)?honeygain\.com.*


(.*\.)?ooklaserver\.net.*
(.*\.)?krunker\.io.*
(.*\.)?myki\.io.*

#Instagram
(.*\.)?instagram\.(.*\.)net.*
i.instagram\.com
graph.instagram\.com

#Tiktok
(.*\.)?tiktokv\.com.*
(.*\.)?tiktokcdn\.com.*
(.*\.)?byteoversea\.com.*
(.*\.)?ibytedtos\.com.*

(.*\.)?tokopedia\.net.*
(.*\.)?tokopedia\.com.*
(.*\.)?tokopedia\.co.id.*
(.*\.)?shopee\.com.*
(.*\.)?shopee\.co.id.*
(.*\.)?shopee\.co.*
(.*\.)?lazada\.co.id.*
(.*\.)?lazada\.com.*

(.*\.)?netflix\.com.*
(.*\.)?emb-api\.com.*
(.*\.)?nflxso\.net.*
(.*\.)?nflxvideo\.net.*

.*\.)?speedtest\.net.*
.*\.)?speedtest\.com.*
.*\.)?stvidtest\.net.*

(.*\.)?line-scdn\.net.*
(.*\.)?naver\.jp.*

#Google
(.*\.)?i.ytimg\.com.*
(.*\.)?googleapis\.com.*
(.*\.)?googleusercontent\.com.*
(.*\.)?googleapis\.com.*
(.*\.)?googlevideo\.com.*
(.*\.)?s.youtube\.com.*
(.*\.)?ggpht\.com.*
accounts.google.com
.googleapis.com
drive.google.com
dl.google.com
one.google.com
play.google.com
mail.google.com

(.*\.)?spotify\.com.*
(.*\.)?wzrkt\.com.*
(.*\.)?i.scdn\.co.*

(.*\.)?viuing\.io.*
(.*\.)?viu\.com.*
(.*\.)?viu\.net.*
(.*\.)?akamaized\.net.*

(.*\.)?pinterest\.com.*
(.*\.)?pinimg\.com.*

.apple.com
.cdn-apple.com
.icloud.com
.icloud-content.com
.itunes.com
.mzstatic.com

.slack.com
.slack-msgs.com
```

## Run as System.d Service
```service
[Unit]
Description=Squid Web Proxy Server
Documentation=man:squid(8)
After=network.target network-online.target nss-lookup.target

[Service]
Type=forking
PIDFile=/usr/local/squid/var/run/squid.pid
ExecStartPre=/usr/local/squid/sbin/squid --foreground -z
ExecStart=/usr/local/squid/sbin/squid -sYC
ExecReload=/bin/kill -HUP $MAINPID
KillMode=mixed
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target

```
