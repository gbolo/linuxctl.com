---
categories:
  - "security"
  - "hardening"
keywords:
  - "hardening"
  - "nginx"
tags:
  - "hardening"
  - "nginx"
  - "crypto"
  - "tls"
  - "reference"
date: "2016-07-17"
title: "Hardening Nginx"
thumbnailImagePosition: "left"
thumbnailImage: "/images/thumb/thumb-nginx.jpg"

---

[Nginx](https://www.nginx.com/resources/wiki/) is a great web server which offers very high performance with little resource consumption. This makes it ideal for docker containers, small embedded devices, or even just dealing with a ton of connections. I also often use Nginx's powerful proxy capabilities. Nginx is one of those applications I use quite often, pretty much for anything related to `http(s)`. Having said that, it becomes very important for me to be able to deploy this in a secure manner.
<!--more-->

<!--toc-->

# Default Configuration
I like to keep the default configuration file very sparse and just do includes. I always start by removing the default `server block` in `/etc/nginx/nginx.conf`. I then follow that up by removing all files from `/etc/nginx/conf.d`

# Disable Server Token
Really simple, just add this to the `server block`. All it does is refrain nginx from displaying it's version number. Throw this file in `/etc/nginx/conf.d/00_disable-tokens.conf`
```bash
# disable information leak
Server_tokens        off;
```

# Disable Unknown Virtual Hosts
I sometimes create a wildcard entry in DNS for certain domains. Doing this may invite `http` requests for sub-domains which do not exist on your nginx server. Also, attackers sometimes scan the web, and do not even provide a `host header` when hitting your nginx server. The point here is, we need to create a default server which will catch all these bad `host headers` and return 444 (nothing)! Throw this file in `/etc/nginx/conf.d/01_catch-bad-vhost.conf`
```bash
# Catch vhosts which we do not host!
server {

  listen        80  default_server;
  server_name  	_; # some invalid name that won't match anything

  # return nothing!
  return        444;
}
```

# TLS Hardening
This next section will deal with TLS hardening

## Disable non-SNI requests
Similar to the above section where we disabled unknown virtual hosts, we want to do the same for [SNI](https://en.wikipedia.org/wiki/Server_Name_Indication). You see, we can't see the user's `host header` yet because the tls connection handshake hasn't happened yet. Since we may have many different tls certificates to offer depending on the domain you would like to visit, they invented **SNI** to have the client tell the web server which domain they would like to visit as part of the TLS `Client Hello` message. This would allow the web server to look up the appropriate certificate to respond with. While not all clients can support this (like Internet Explorer on Windows XP), I choose to enforce it anyways. So when a request comes in on my **https** port that fails to have an SNI or provides an SNI that I do not know about, lets give back a weak self signed certificate and immediately hang up (return a 444!). Throw this file in `/etc/nginx/conf.d/01_catch-bad-sni.conf`
```bash
# Catch SNIs which we do not host!
server {

  listen        443 ssl;
  server_name  	_; # some invalid name that won't match anything

  ssl_certificate     /etc/nginx/ssl/weak/weak.crt;
  ssl_certificate_key /etc/nginx/ssl/weak/weak.key;

  # return nothing!
  return        444;
}
```
{{< alert warning >}}
The above configuration needs weak TLS Certificates to be provided. You can create and sign your own by reading this: [openssl reference](https://linuxctl.com/2016/12/openssl---reference/#rivest-shamir-adleman-rsa)
{{< /alert >}}


## TLS Allowed Protocols and Cipher Suites
Choosing cipher suites will hopefully be a thing from the past with the up-comming TLS 1.3 spec. Until then I would highly recommend [mozilla's cipher suite selections](https://wiki.mozilla.org/Security/Server_Side_TLS#Modern_compatibility). They break it down into compatibility categories like modern, intermediate, old. Ofcourse I would highly recommend choosing from the modern section (this is a hardening post after all). Also I would only enable TLS 1.1 and 1.2. Throw this file in `/etc/nginx/conf.d/00_tls-cipher-suites.conf`
```bash
# allowed protocols and ciphers
ssl_prefer_server_ciphers on;
ssl_protocols TLSv1.1 TLSv1.2;
ssl_ciphers 'EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH:!aNULL:!MD5';
```

## Diffie-Hellman Parameter for EDH Cipher Suites
You will notice that I have chosen some cipher suites that contain `EDH`. The E stands for **Ephemeral**. This basically means that these suites provide [forward secrecy](https://en.wikipedia.org/wiki/Transport_Layer_Security#Forward_secrecy) that can prevent your past sessions (if they were recorded by man-in-middle) to be decrypted should the server private key get compromised. Creating Diffie-Hellman Parameters help better maintain integrity of this key exchange
```bash
# generate parameters - may take a while
openssl dhparam -out dhparam4096.pem 4096
```

Once those parameters have been created, Throw this file in `/etc/nginx/00_tls-dhparams.conf`
```
# Diffie-Hellman parameter for DHE ciphersuites
ssl_dhparam /etc/nginx/tls/dhparam4096.pem;
```

## Enable HTTP Strict Transport Security (HSTS)
If everything on your domain is encrpyted, you should probably set this `http header`. It basically tells the browser that it should expect everything in this domain be accessed via an encrypted channel (https). This would help in situations where an attacker would like to perform `ssl stripping` on your connections. While this is not a silver bullet by any means, it just adds another layer (a small layer) of additional security. Throw this file in `/etc/nginx/00_tls-hsts.conf` or individually to certain domains.
```bash
# HSTS (HTTP Strict Transport Security) avoids ssl stripping
add_header Strict-Transport-Security max-age=15768000; # six months
# use this only if all subdomains support HTTPS!
# add_header Strict-Transport-Security "max-age=15768000; includeSubDomains";
```
