---
categories:
  - "administration"
  - "blockchain"
keywords:
  - "blockchain"
  - "docker"
  - "hyperledger"
  - "fabric"
tags:
  - "blockchain"
  - "docker"
  - "hyperledger"
  - "fabric"
  - "automation"
date: "2017-08-09"
title: "Bootstrapping Hyperledger Fabric 1.0"
thumbnailImagePosition: "left"
thumbnailImage: "/images/thumb/thumb-hyperledger.png"

---

I am proud to say that I have been working with the [Hyperledger Fabric](https://www.hyperledger.org/projects/fabric) project nearly since it's inception almost two years ago. While frustrating at times in the beginning (as is the case with most open-source, popular, and pre-alpha software), I have grown to enjoy working with the Fabric. It has improved my skills in a variety of areas such as golang, docker, encryption, pkcs11, continuous integration, and many more. Since the Fabric recently went 1.0, this blog post will focus on how to bootstrap the fabric without the aid of `cryptogen` tool.
<!--more-->

<!-- toc -->

# Introduction
The Hyperledger Fabric is an opensource project which provides you a platform/framework to implement/build a blockchain (distributed ledger) application. It is modular in design, and is capable of running chaincode (smart contracts) which essentially is your customized logic on how to write/read from the ledger. Ofcourse this project is tough to explain in just a sentence, so you can read more about it [here](http://hyperledger-fabric.readthedocs.io/en/latest/blockchain.html).

# Fabric Components
In order for this blog post to make any sense, you must first understand how the Hyperledger Fabric works. I highly recommend reading the [official documentation](http://hyperledger-fabric.readthedocs.io/en/latest) to accomplish this. Here are some links to components/concepts of interest:

 - [Fabric-CA Server](http://hyperledger-fabric-ca.readthedocs.io/en/latest/users-guide.html#overview)
 - [Fabric Peer](http://hyperledger-fabric.readthedocs.io/en/latest/arch-deep-dive.html)
 - [Fabric Orderer](http://hyperledger-fabric.readthedocs.io/en/latest/arch-deep-dive.html)
 - [Membership Service Providers (MSP)](http://hyperledger-fabric.readthedocs.io/en/latest/msp.html)

# Why not use pre-generated crypto material?
The Fabric repo provides a tool which you can compile to produce crypto material (called `cryptogen`) required for MSP. While this tool is great for development and testing purposes, I feel that is unacceptable to use for a production grade deployment. In my opinion, **private key material should be completely private and secured in pkcs11 based HSMs**. Also, the Fabric CA server should be aware of any certificate that it signs. In the current design, the Fabric Peer and Orderer cannot create it's own MSP directory. It is assumed that the administrator would pre-create this and provide it to them. In older versions of the Fabric (when the orderer didn't even exist), the Peer was able to enroll itself against the Fabric CA (called `membersrvc` back then) provided that you supplied the correct secret in it's config file. In order to accomplish something similar today, we will need to include the `fabric-ca-client` binary on our fabric peer host.

# Bootstrapping the Peer
As mentioned previously, we need the `fabric-ca-client` binary to enroll with the Fabric CA server in order for us to begin the construction of our MSP directory. A successful enrollment with this binary will produce the following:
 - The ECDSA private key stored in the `keystore` directory (if SW based BCSSP is used)
 - The Enrollment x509 Certificate signed by the CA will be stored in the `signcerts` directory
 - The Fabric CA self-signed x509 root Certificate will be stored in the `cacerts` directory

While this is a great start, we still need two more directories to complete our [MSP](http://hyperledger-fabric.readthedocs.io/en/latest/msp.html):
 - `tlscacerts` directory is needed to set up our chain of trust for TLS connections
 - `admincerts` directory must contain x509 certificates corresponding to an administrator

# Obtaining the Administrator Certificate
Before we Enroll the peer we will need to enroll an Administrator user, so that we can obtain it's x509 Certificate to pass along to the peer. When doing this for development/testing purposes, on our local laptop, we can use local docker volumes to share data between our containers. In this case, we would spin up a container that completes an enrollment for the administrator user and stores it on a volume. Then when the peer starts up, we can have some custom entrypoint logic to copy this certificate to its own local `MSP` directory after it enrolls itself.

In a more secure production grade deployment, we would enroll the administrative user on a seperate machine, attached to an HSM to secure it's private key. We can then store it's x509 enrollment certificate in a git repository and deploy it with a configuration management host such as `ansible`.

# Custom Entrypoint Logic
Since we wish to have the peer enroll it's self we need to create a custom entrypoint script. This entrypoint script should take in additional values from `environment variables` for enrollment purposes. This will include things such as the url to the Fabric-CA server and valid credentials.

**It is important to note that, for a production grade deployment, these credentials should be registered by the administrator with a `max_enrollments` set to 1. This will ensure that once these credentials are used once, they can never be re-used again.**

In my [public test repository](https://github.com/gbolo/dockerfiles/tree/master/hyperledger-fabric/softhsm/compose), I use the following scripts to accomplish this:
 - [entrypoint.sh](https://github.com/gbolo/dockerfiles/blob/master/hyperledger-fabric/softhsm/dockerfiles/install/entrypoint.sh)
 - [entrypoint-functions.sh](https://github.com/gbolo/dockerfiles/blob/master/hyperledger-fabric/softhsm/dockerfiles/install/entrypoint-functions.sh)

# Testing out the Bootstrapping
The bootstrapping i covered above is actually more involved then I explained. For example, I also bootstrap the orderer and use `provisional` genesis block creation. However, you are free to see and test the full setup in my [public git repository](https://github.com/gbolo/dockerfiles/tree/master/hyperledger-fabric/softhsm/compose). **Also note that the images used in this repository are not the official images and are instead created and maintained by myself and are available here:**

 - [Fabric-CA Server](https://hub.docker.com/r/gbolo/fabric-ca/)
 - [Fabric Peer](https://hub.docker.com/r/gbolo/fabric-peer/)
 - [Fabric Orderer](https://hub.docker.com/r/gbolo/fabric-orderer/)
 - [Fabric Tools](https://hub.docker.com/r/gbolo/fabric-tools/)

You will need `git`, `docker`, and `docker-compose` installed in order to try it out for yourself. Here we go:

```bash
# Clone the repository
$ git clone https://github.com/gbolo/dockerfiles.git

# Change directory to where the compose file is
$ cd dockerfiles/hyperledger-fabric/softhsm/compose/

# Run the wrapper script which will fetch the required images and compose the environment
$ ./wrapper-test1.sh
```

# Details left out
There is a lot going on with the demonstration provided above that I did not touch on. If I touched on everything, then this blog post would be many times longer. If there's some questions about fabric or what's going on in this demo output please feel free to leave a comment below, or find me (gbolo) on the official Hyperledger [rocket chat](https://chat.hyperledger.org) (you will need [Linux Foundation](https://www.linuxfoundation.org/) credentials to join this chat). For those of you who would like to suggest or make improvments to this boot-strapping demo, please feel free to submit a Pull Request to [gbolo/dockerfiles](https://github.com/gbolo/dockerfiles)
