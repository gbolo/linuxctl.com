---
categories:
  - "administration"
  - "blockchain"
keywords:
  - "blockchain"
tags:
  - "blockchain"
  - "bigchaindb"
date: "2016-03-19"
title: "BigchainDB on CentOS 7"
thumbnailImagePosition: "left"
thumbnailImage: "/images/thumb/thumb-bigchaindb.png"

---

It's been the year of the no-sql distributed databases for me. Technically, it started a few years back when I discovered the ELK stack, but this year I found myself evaluating many more of these distributed databases for an exciting project at my workplace. A few months back I started playing with [Apache Cassandra](http://cassandra.apache.org/) and was really impressed with it's capabilities. However, I since switched my focus to evaluate a new kid on the block (or should I say block chain?) called [bigchaindb](https://www.bigchaindb.com/) which may be more suited to the requirements of our project.
<!--more-->

Since we are mostly a RedHat shop, I decided to try and get this running on the latest CentOS 7. This application has a few requirements:

* RethinkDB
* Python 3.4+
* OpenSSL ==with ECC==

RethinkDB is fairly simple to install and Python 3.4 is available on the epel repo. However, openssl on CentOS 7 only ships with these three ECC curves compiled in:

```
root@c7-dev3 ~ $ openssl ecparam -list_curves
  secp384r1 : NIST/SECG curve over a 384 bit prime field
  secp521r1 : NIST/SECG curve over a 521 bit prime field
  prime256v1: X9.62/SECG curve over a 256 bit prime field
```

In order to get more ECC curves, we are going to have to compile openssl from source. To preserve the current default openssl installation on this OS, I will be compiling it, then placing it to a non-standard location. I consider this a safer approach then just replacing what we already have installed. So let's get started:

```
# Download source and compile
root@c7-dev3 ~ $ yum -y install gcc
root@c7-dev3 ~ $ mkdir -p /opt/openssl-1.0.1s/build; cd /opt/openssl-1.0.1s
root@c7-dev3 /opt/openssl-1.0.1s $ wget https://www.openssl.org/source/openssl-1.0.1s.tar.gz
root@c7-dev3 /opt/openssl-1.0.1s $ tar xzvf openssl-1.0.1s.tar.gz
root@c7-dev3 /opt/openssl-1.0.1s $ mv openssl-1.0.1s src; cd src
root@c7-dev3 /opt/openssl-1.0.1s/src $ export CFLAGS="-fPIC"
root@c7-dev3 /opt/openssl-1.0.1s/src $ ./config --prefix=/opt/openssl-1.0.1s/build shared enable-ec enable-ecdh enable-ecdsa
root@c7-dev3 /opt/openssl-1.0.1s/src $ make all
root@c7-dev3 /opt/openssl-1.0.1s/src $ make install

# Check if we got more curves
root@c7-dev3 /opt/openssl-1.0.1s/src $ openssl ecparam -list_curves | wc -l
3
root@c7-dev3 /opt/openssl-1.0.1s/src $ /opt/openssl-1.0.1s/build/bin/openssl ecparam -list_curves | wc -l
73
```

Alright, now that we got the openssl curves we needed, we can continue installing the rest:

```
# Install RethinkDB
root@c7-dev3 ~ $ wget https://download.rethinkdb.com/centos/6/`uname -m`/rethinkdb.repo -O /etc/yum.repos.d/rethinkdb.repo
root@c7-dev3 ~ $ yum -y install rethinkdb

# Install python 3.4 and some other deps
root@c7-dev3 ~ $ yum -y install epel-release
root@c7-dev3 ~ $ yum -y install libffi-devel gcc-c++ redhat-rpm-config python34-devel

# Install pip
root@c7-dev3 ~ $ curl https://bootstrap.pypa.io/get-pip.py | python3.4

# Install bigchain and specify the openssl headers and libraries
root@c7-dev3 ~ $ CFLAGS='-I /opt/openssl-1.0.1s/src/include' LDFLAGS='-L /opt/openssl-1.0.1s/build/lib -Wl,-rpath=/opt/openssl-1.0.1s/build/lib' pip --no-cache-dir install bigchaindb

# start rethinkdb & bigchaindb
root@c7-dev3 ~ $ rethinkdb --daemon && bigchaindb start
```
