---
categories:
  - "administration"
  - "containers"
keywords:
  - "docker"
tags:
  - "docker"
  - "scanning"
  - "lightweight"
  - "golang"
  - "compile"
date: "2017-03-12"
title: "Building Tiny Secure Docker Containers"
thumbnailImagePosition: "left"
thumbnailImage: "/images/thumb/thumb-docker.png"

---

It's very tempting to use the most popular Linux distributions as a base for docker containers. In fact, most of the time, that is actually a good idea. However, when trying to build the most secure container possible, at the lowest possible size, these base images become bloat. Why include libraries and other binaries in your docker container if your application does not need them?
<!--more-->

<!-- toc -->

# Are Size and Security Related?
A security chief once told me something along these lines:
> Security is layered like an onion

Funny enough, this reminds me of the layers of a docker image. Lowering your attack vector is essential to security. Generally speaking, **the more libraries and binaries you have installed on your system, the greater the odds that your system will be susceptible to vulnerabilities discovered tomorrow** (all things being equal). Obviously, there is no **silver bullet** for security, but any hardening guide I ever read always emphasised that:

> Minimization of your installation is the most powerful security tool

This, ofcourse, does not mean that larger docker images are always less secure. However, I would say that it is a decent indicator.

## Image Size compared to CVEs
Let's compare image sizes of the most popular base OS images on [dockerhub](https://hub.docker.com) as of today (2016-03-12):

| Image | Size | C/M CVE |
|-------|------|-----|
| CentOS 7 | 70MB | 21 |
| CentOS 6 | 69MB | 27 |
| Debian 9 | 42MB | 3 |
| Debian 8 | 51MB | 6 |
| Debian 7 | 36MB | 6 |
| Ubuntu 16.10 | 41MB | 6 |
| Ubuntu 16.04 | 50MB | 8 |
| Ubuntu 14.04 | 66MB | 13 |
| OpenSuse 42.2 | 48MB | 20 |
| OpenSuse 42.1 | 38MB | 20 |
| Alpine 3.5 | 2MB | 1 |
| Alpine 3.4 | 2MB | 0 |
| busybox 1.26-musl | 723KB | 0 |

It's no surprise that **most offcial images on the docker hub now include an alpine version!** Not only is the base image anymore between **20 to 35 times smaller** but also contains far less known vulnerabilities of type critical or major.

# How Small can a Container Get?
Using [Alpine Linux](https://hub.docker.com/r/library/alpine/) as your base image can produce pretty small images. However, applications written in languages that can be statically compiled down to a single binary can often make the smallest of container images. This is because **you don't actually need any external libraries for statically compiled applications and can therefore usually skip choosing a base image from above and instead create a docker container from SCRATCH**. Let's take a modern language like [Golang](https://golang.org/) for example (which is getting extremely popular!). Let's create a tiny container for a [test web application](https://github.com/gbolo/go-tinyapi) I wrote in Go to learn the language. **Side note:** If you interested in learning go, I highly recommend [this udemy online course](https://www.udemy.com/learn-how-to-code/) which is taught by gopher [Todd McLeod](https://github.com/GoesToEleven)

## Building a container image from SCRATCH
You might be surprised to know that a docker container can function with just a binary in it. Remember that a docker container is **essentially just a process.** Sometimes, just a set of processes. The kernel is provided by the Host Operating System. It also doesn't need all the stuff your Host OS contains such as `ssh`, `systemd`, `syslog`, exc. Let's demonstrate this:

```bash
## Pull the application from the repo
git pull https://github.com/gbolo/go-tinyapi.git

## Statically compile this go application (requires go, duh!)
cd go-tinyapi; CGO_ENABLED=0 GOOS=linux go build -o tinyapi

## Verify that this binary has no dependancies on external libraries
ldd tinyapi
	not a dynamic executable

## Build the docker Image
docker build -t tinyapi .

## Run the Docker Container:
docker run --rm -p 8080:8080 tinyapi

## (OPTIONAL) -- Too Lazy
## If you don't want to compile this on your own, I have a pre-made image for you:
####  docker run --rm -p 8080:8080 gbolo/tinyapi:v0.1
```

Once the docker container is ready, you can open another console and hit it's various endpoints:

```bash
curl -v http://127.0.0.1:8080/
curl -v http://127.0.0.1:8080/alwaysok
curl -v http://127.0.0.1:8080/healthcheck
curl -v http://127.0.0.1:8080/panic
```

The final endpoint `/panic` will cause the application to exit when hit killing and erasing your container (your welcome!). Going back to the first console, you can view the logs the application generated to `stdout`. As you can see, this is an example of a fully functional (yet useless) **docker container which has absolutely nothing on the filesystem which it does not need**. Pretty neat!

# TL;DR
Docker containers should be treated **more like a process** rather than a Virtual Machine. **Build your container around your application**, instead of just throwing it in at the last step. Include only what is absolutely needed in your container in order for your application to **function properly under all circumstances**. This minimalist approach will almost always **produce a more secure container**. As a bonus, your smaller container will also be more distributable. Now go out there and start making some damned small containers!
