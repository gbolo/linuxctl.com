---
categories:
  - "administration"
  - "security"
keywords:
  - "lets-encrypt"
tags:
  - "tls"
  - "lets-encrypt"
  - "certs"
  - "crypto"
  - "encryption"
  - "automation"
date: "2016-05-28"
title: "Let's Encrypt Muli-Domain Across Unique IPs"
thumbnailImagePosition: "left"
thumbnailImage: "/images/thumb/thumb-lets-encrypt.png"

---

**Let's Encrypt is awesome!** This service allows you to automate the retrieval of as many valid TLS certificates as you wish, as long as you can *"prove"* that you own the domain. One of the first proofs that they offered was the `http-01` challenge. This proof works by essentially sending your domain a random `HTTP GET` request string which your lets-encrypt client must receive and send back.
<!--more-->

## Problems with the `http-01` Challenge
By default, the client will listen on port 80 to receive this request, which may be a problem if your webserver is already listening on port 80 to serve your domain's web traffic. Another potential issue is that you may have subdomains on different hosts and would like to avoid having to install the lets-encrypt agent on each one. Summary of problem:

* port 80 may be in use already by a real webserver
* forcing certbot to listen on port 80 during renewal will cause a small outage
* you need to run certbot on all servers which resolve to the domain list

## The Solution
Fortunately, there are many ways to avoid these issues, my favourite being the use of `docker containers` and `nginx's reverse proxy` capabilities. Let's draw out what a solution would look like (at a high level):

![diagram](/images/posts/letsencrypt-diagram1.png)

The solution involves setting up a central server with a docker container for the lets-encrypt client (certbot) and having nginx reverse-proxy **only HTTP requests specific to these challenges** back to your certbot docker container.

## The Solution Guide
This guide assumes that you already have nginx serving traffic for your domains and that you have some experience with docker.

#### STEP 1
Install docker and create a directory for certbot config and data
```
# Install and start docker (on centos/rhel)
yum install -y docker && systemctl start docker

# Create some directories for certs and configs
mkdir -p /opt/certbot/{config,data}
```

#### STEP 2
Create Config files for your domains
```
cat /opt/certbot/config/linuxctl.com.conf

# Example config file
rsa-key-size = 4096
text = True
non-interactive = True
# If an existing cert covers some subset of the requested names, always expand and replace it with the additional names.
expand = True

email = gbolo@linuxctl.com
domains = linuxctl.com, www.linuxctl.com, blog.linuxctl.com, static.linuxctl.com, repo.linuxctl.com, site1.lab.linuxctl.com, site2.lab.linuxctl.com

# https://certbot.eff.org/docs/using.html#standalone
authenticator = standalone
standalone-supported-challenges = http-01

# tells Certbot to continue with cert generation if only some of the specified domain authorazations can be obtained
allow-subset-of-names = true

```

#### STEP 3
Set up nginx to reverse proxy requests for challenges:
```
# Use this Location section on central server to proxy these requests to local certbot:

  location ~ /.well-known/acme-challenge/ {
      allow all;
      proxy_pass       http://127.0.0.1:8080;
      proxy_set_header Host $host;
      proxy_set_header X-Forwarded-For $remote_addr;
      proxy_set_header X-Forwarded-Proto $scheme;
  }

# Use this Location section on remote servers to proxy requests back to your central nginx server:

  location ~ /.well-known/acme-challenge/ {
      allow all;
      proxy_pass       http://CENTRAL_SERVER;
      proxy_set_header Host $host;
      proxy_set_header X-Forwarded-For $remote_addr;
      proxy_set_header X-Forwarded-Proto $scheme;
  }

# !NOTICE!
# MAKE SURE THAT YOUR CENTRAL NGINX SERVER IS CONFIGURED TO HANDLE REQUESTS FOR THESE REMOTE DOMAINS!
```

#### STEP 4
Launch a docker container for certbot, bound to `127.0.0.1:8080`:

```
 docker run -d --name certbot --restart=always \
   -p 127.0.0.1:8080:80 \
   -v /opt/certbot/config:/config:ro \
   -v /opt/certbot/data:/etc/letsencrypt \
   gbolo/alp-letsencrypt
```

#### STEP 5
Verify that you got your certs

```
# Check container logs:
docker logs certbot

# Check data directory
tree -L 3 /opt/certbot/data
/opt/certbot/data
├── accounts
│   └── acme-v01.api.letsencrypt.org
│       └── directory
├── archive
│   └── linuxctl.com
│       ├── cert1.pem
│       ├── chain1.pem
│       ├── fullchain1.pem
│       └── privkey1.pem
├── csr
│   └── 0000_csr-letsencrypt.pem
├── keys
│   └── 0000_key-letsencrypt.pem
├── live
│   └── smlra.ca
│       ├── cert.pem -> ../../archive/linuxctl.com/cert1.pem
│       ├── chain.pem -> ../../archive/linuxctl.com/chain1.pem
│       ├── fullchain.pem -> ../../archive/linuxctl.com/fullchain1.pem
│       └── privkey.pem -> ../../archive/linuxctl.com/privkey1.pem
└── renewal
    └── linuxctl.com.conf
```

Now that you have your certs in a central location, you can distribute them periodically to any servers that needs them via `scp` or `rsync` through a cronjob.


To be fair, the letsencrypt team has provided additional challenges and methods of receiving certs. The method describe above was what I used since near the beginning of the letsencrypt service going live. Other methods are described here: https://certbot.eff.org/docs/using.html#getting-certificates
