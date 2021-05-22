# mtproto-tls-server

These instructions document how to create a Telegram MTProto proxy server with fake TLS name equal to actual hostname. This repository includes a sample web site `html` to announce the proxy details to users.

The mtg version of this tutorial has been updated to version 2.0.

You will need a server and a domain name. Create a DNS record for the hostname pointing to the server IP address.

These instructions are for Debian/Ubuntu logged in as root.

## 1. Install Nginx

Install the Nginx web server from the repositories.

```
apt update && apt upgrade -y
apt install git vim wget -y
apt install nginx
```

Edit `vim /etc/nginx/sites-available/default`.

Insert your actual hostname.

```
        server_name host.example.com;
```

Restart Nginx with your hostname defined in the server configuration.

```
systemctl restart nginx
```

## 2. Add initial web contents

You can adapt the `html` directory from this repository.

## 3. Get certificate from Let's Encrypt

Follow the instructions on https://certbot.eff.org to install a real certificate on your server.

```
apt install snapd
snap install core
snap refresh core
snap install --classic certbot
ln -s /snap/bin/certbot /usr/bin/certbot
certbot --nginx
```

The Let's Encrypt certificate needs to be updated approximately every 90 days. Set things up to check for the necessity of an update.

```
certbot renew --dry-run
```

## 4. Install Go language

See https://golang.org/doc/install for documentation. At the time of writing, the current version of Go language is 1.15.2.

```
wget https://golang.org/dl/go1.16.linux-amd64.tar.gz
tar -C /usr/local -xzf go1.16.linux-amd64.tar.gz
export PATH=$PATH:/usr/local/go/bin
```

## 5. Download 9seconds/mtg

See https://github.com/9seconds/mtg for documentation.

```
git clone https://github.com/9seconds/mtg.git
```

## 6. Build mtg

See https://github.com/9seconds/mtg for documentation.

```
cd mtg
go build
cp mtg /usr/local/bin
```

## 7. Generate secret for mtg

Replace `host.example.com` by your server's actual hostname.

```
mtg generate-secret --hex host.example.com
```

This returns `<SECRET>`.

Generating the secret needs to be repeated every 90 days, when the Let's Encrypt certificate is renewed.

## 8. Reconfigure Nginx

Edit `/etc/nginx/sites-available/default` to make Nginx listen on port 993.

```
        listen [::]:993 ssl ipv6only=on; # managed by Certbot
        listen 993 ssl; # managed by Certbot
```

Restart nginx after configuring mtg.

## 9. Configure mtg

Create `/usr/lib/systemd/system/mtg.service`

```
[Unit]
Description=mtg

[Service]
ExecStart=/usr/local/bin/mtg run /etc/mtg.toml
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```
Create `/etc/mtg.toml`
```
# This is an example of the configuration file for mtg. You actually can
# run mtg with it. It starts a proxy on all interfaces with a secret
# ee367a189aee18fa31c190054efd4a8e9573746f726167652e676f6f676c65617069732e636f6d
#
# It has all possible options with default values. So, a real world
# configuration file should contain only those options you are going to
# use. You do not need to enumerate all of them. In other words, each
# option here has a default value. If you comment a key-value pair, it
# should not make any effect.
#
# stats is the only exception.

# Debug starts application in debug mode. It starts to be quite verbose
# in output. Actually, the idea is that you run it in debug mode only if
# you have any issue.
debug = true

# A secret. Please remember that mtg supports only FakeTLS mode, legacy
# simple and secured mode are prohibited. For you it means that secret
# should either be base64-encoded or starts with ee.
secret = "ee367a189aee18fa31c190054efd4a8e9573746f726167652e676f6f676c65617069732e636f6d"

# Host:port pair to run proxy on.
bind-to = "0.0.0.0:443"

# Defines how many concurrent connections are allowed to this proxy.
# All other incoming connections are going to be dropped.
concurrency = 8192

# A size of user-space buffer for TCP to use. Since we do 2 connections,
# then we have tcp-buffer * (4 + 2) per each connection: read/write for
# each connection + 2 copy buffers to pump the data between sockets.
tcp-buffer = "4kb"

# Sometimes you want to enforce mtg to use some types of
# IP connectivity to Telegram. We have 4 modes:
#   - prefer-ipv6:
#     We can use both ipv4 and ipv6 but ipv6 has a preference
#   - prefer-ipv4:
#     We can use both ipv4 and ipv6 but ipv4 has a preference
#   - only-ipv6:
#     Only ipv6 connectivity is used
#   - only-ipv4:
#     Only ipv4 connectivity is used
prefer-ip = "prefer-ipv4"

# FakeTLS uses domain fronting protection. So it needs to know a port to
# access.
domain-fronting-port = 993

# FakeTLS can compare timestamps to prevent probes. Each message has
# encrypted timestamp. So, mtg can compare this timestamp and decide if
# we need to proceed with connection or not.
#
# Sometimes time can be skewed so we accept all messages within a
# time range of this parameter.
tolerate-time-skewness = "5s"

# network defines different network-related settings
[network]
# please be aware that mtg needs to do some external requests. For
# example, if you do not pass public ips, it will request your public ip
# address from some external service.
#
# As for 2.0, if you set a public-ip on your own, mtg won't issue any
# network requests except of those required for Telegram.
#
# so, in order of doing them, it needs to do DNS lookup. mtg ignores DNS
# resolver of the operating system and uses DOH instead. This is a host
# it has to access.
#
# By default we use Quad9.
doh-ip = "9.9.9.9"

# mtg can work via proxies (for now, we support only socks5). Proxy
# configuration is done via list. So, you can specify many proxies
# there.
#
# Actually, if you supply an empty list, then no proxies are going to be
# used. If you supply a single proxy, then mtg will use it exclusively.
# If you supply >= 2, then mtg will load balance between them.
#
# If you add an empty string here, this is an equivalent of 'plain network',
# with no proxy usage.
#
# Proxy configuration is done via ordinary URI schema:
#
#     socks5://user:password@host:port?open_threshold=5&half_open_timeout=1m&reset_failures_timeout=10s
#
# Only socks5 proxy is used. user/password is optional. As you can
# see, you can specify some parameters in GET query. These parameters
# configure circuit breaker.
#
# open_threshold means a number of errors which should happen so we stop
# use a proxy.
#
# half_open_timeout means a time period (in Golang duration notation)
# after which we can retry with this proxy
#
# reset_failures_timeout means a time period when we flush out errors
# when circuit breaker in closed state.
#
# Please see https://docs.microsoft.com/en-us/azure/architecture/patterns/circuit-breaker
# on details about circuit breakers.
proxies = [
    # "socks5://user:password@host:port?open_threshold=5&half_open_timeout=1m&reset_failures_timeout=10s"
]

# network timeouts define different settings for timeouts. tcp timeout
# define a global timeout on establishing of network connections. idle
# means a timeout on pumping data between sockset when nothing is
# happening.
#
# please be noticed that handshakes have no timeouts intentionally. You can
# find a reasoning here:
# https://www.ndss-symposium.org/wp-content/uploads/2020/02/23087-paper.pdf
[network.timeout]
tcp = "5s"
http = "10s"
idle = "1m"

# Some countries do active probing on Telegram connections. This technique
# allows to protect from such effort.
#
# mtg has a cache of some connection fingerprints. Actually, first bytes
# of each connection. So, it stores them in some in-memory LRU+TTL cache.
# You can configure this cache here.
[defense.anti-replay]
# You can enable/disable this feature.
enabled = true
# max size of such a cache. Please be aware that this number is
# approximate we try hard to store data quite dense but it is possible
# that we can go over this limit for 10-20% under some conditions and
# architectures.
max-size = "1mib"
# we use stable bloom filters for anti-replay cache. This helps
# to maintain a desired error ratio.
error-rate = 0.001

# You can protect proxies by using different blocklists. If client has
# ip from the given range, we do not try to do a proper handshake. We
# actually route it to fronting domain. So, this client will never ever
# have a chance to use mtg to access Telegram.
#
# Please remember that blocklists are initialized in async way. So,
# when you start a proxy, blocklists are empty, they are populated and
# processed in backgrounds. An error in any URL is ignored.
[defense.blocklist]
# You can enable/disable this feature.
enabled = true
# This is a limiter for concurrency. In order to protect website
# from overloading, we download files in this number of threads.
download-concurrency = 2
# A list of URLs in FireHOL format (https://iplists.firehol.org/)
# You can provider links here (starts with https:// or http://) or
# path to a local file, but in this case it should be absolute.
urls = [
    # "https://iplists.firehol.org/files/firehol_level1.netset",
    # "/local.file"
]
# How often do we need to update a blocklist set.
update-each = "24h"

# statsd statistics integration.
[stats.statsd]
# enabled/disabled
enabled = false
# host:port for UDP endpoint of statsd
address = "127.0.0.1:8888"
# prefix of metric for statsd
metric-prefix = "mtg"
# tag format to use
# supported values are 'datadog', 'influxdb' and 'graphite'
# default format is graphite.
tag-format = "datadog"

# prometheus metrics integration.
[stats.prometheus]
# enabled/disabled
enabled = true
# host:port where to start http server for endpoint
bind-to = "127.0.0.1:3129"
# prefix of http path
http-path = "/"
# prefix for metrics for prometheus
metric-prefix = "mtg"
```
## 10. Run mtg

Run the mtg service.

```
systemctl enable mtg
systemctl start mtg
```
Now you can generate some useful links:
```
mtg access /etc/mtg.toml
```
Restart Nginx
```
systemctl restart nginx
```
## 11. Check mtg

Check that mtg is active and running, and review messages.

```
systemctl status mtg
journalctl -u mtg
```

## 12. Update web content

Now that everything is running, update the hostname, port, secret, start date and expiry date in your web site's contents.

You will need to update your web site every 90 days, when the Let's Encrypt certificate and the secret are renewed.
