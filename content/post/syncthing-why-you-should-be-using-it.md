---
categories:
  - "security"
  - "encryption"
keywords:
  - "syncthing"
tags:
  - "syncthing"
  - "tls"
  - "syncing"
  - "storage"
date: "2017-04-23"
title: "Syncthing - Why You Should be Using it"
thumbnailImagePosition: "left"
thumbnailImage: "/images/thumb/thumb-syncthing.png"

---

Choosing a secure file syncing application has never been easier. After evaluating a variety of options such [Dropbox](https://www.dropbox.com), [OwnCloud](https://owncloud.org/), and [Seafile](https://www.seafile.com) for over 5 years, the journey is finally over. I have found the best option - [Syncthing](https://syncthing.net). Let me explain why you should consider it.
<!--more-->

<!-- toc -->

# A Brief Log of the Journey
Before I get into why I feel [Syncthing](https://syncthing.net) is the best option for a self-hosted syncing solution, let me give an account of my history with syncing software.

## Start of the Journey - DropBox
Looking through my email inbox, I found the welcome email I received when I first joined Dropbox. It was delivered in 2011, and I remember how much I loved it back then. Finally, I had a convenient way to sync files across all my computers. Life was easier back then, ignorance was bliss. I was happy for a while, but when I started running my own server at home full time, I began thinking:

> I have my Linux box at home with all this free storage available, why don't I start it for my syncing needs?

This immediately made me thing of the potential benefits of self-hosting options:
- All-you-can-eat storage for free!
- I can put some sensitive/personal files in my shares, and not worry about it being in the cloud.
- Gain experience/knowledge with this type of use case.

Of course, I didn't think much about the issues I may run into.

## My two years with OwnCloud
My first foray into te self-hosted options was OwnCloud. I believe at the time it was pretty much the only option I knew of. Since I did a bunch of web development back then with php, OwnCloud was very attractive to me. Installation couldn't be simpler, just extract the tar into your webroot, set up a mysql database with provided schema and you were ready. Just NAT the port to my OwnCloud VM (which was running `Apache` at the time with `mod_php`) and it was now publicly available. At the time there was no let's encrypt, but browsers also had *less of a problem* with **self-signed** TLS certificates. I remember being very proud of it, and would take any opportunity to talk about how cool my set up was. However, my time with OwnCloud was not without it's issues:

### OwnCloud was slow
I have nothing against PHP, in fact I have written most of my web applications in PHP and I know it very well. But, for what ever reason, I found OwnCloud to be slow at times.

### Sync conflicts
This would happen often enough, that I had to create a script to periodically run across my shares to scan for them.

I will cut OwnCloud some slack, after all they were kind of pioneers in this. It served my needs for two years but it was time to try something else.

## My two years with SeaFile
I divorced OwnCloud and married SeaFile in **just an afternoon.** Although the installation of SeaFile was significantly more involved, I had already become a talented Linux System Administrator by then. Fronted with `nginx`, and secured with legitimately trusted TLS certificates obtained for free thanks to [Let's Encrypt](https://linuxctl.com/2016/05/lets-encrypt-muli-domain-across-unique-ips/), I didn't think it could get much better than this. **Speed was much improved** and the whole experience just felt more polished. Things were great, but as my knowledge in the field grew so did my concerns. Although I had control of the in-transit security with TLS terminated on my nginx VM, **authentication was still handled by username and password.** Also the Android app wasn't the greatest and would occasionally failed to sync all my photos properly. The last straw was when [SeaFile split into two](https://forum.syncwerk.com/t/reason-for-removing-statements-and-forum-thread-regarding-split-up-between-Syncwerk-gmbh-and-Syncwerk-ltd/5637/4) due to some bizarre conflict between two companies.

# Syncthing - The Light at the end of the Tunnel
I like to think of Syncthing as my final destination after years searching for the perfect solution. It has a ton of benefits so lets go over them.

## Protected with TLS Mutual Authentication
No simple **username + password** here! When you initialize Syncthing for the first time, it will create an **Elliptic Curve Key Pair** (although older versions used RSA). It then uses a hash function (sha256) on it's self-signed x509 certificate to determine it's unique Device ID as described [here](https://docs.syncthing.net/dev/device-ids.html). This Device ID must now be shared with other devices you would like to communicate with, which means you are **explicitly telling other devices to only trust the holder of the private key** which created the public key mentioned in the x509 certificate. This is a far more secure authentication method than any of the other products I used in the past. Sure it is a little *less convenient* than the others (although the QR code helps), but I am willing to **sacrifice some convenience for much improved security.** Also, **only perfect forward secrecy ciphers are supported,** which means if your keys get compromised some how, your previous transfers (if recorded) will still be safe. More details on the protocol spec can be found [here](https://docs.syncthing.net/specs/index.html).

## Distributed Peer to Peer Model
Unlike the other products, **there is no central server in Syncthing**, it's completely distributed. Every installation has the same capabilities. Any node in the network can die and it wont disrupt syncing as long as your shares are not shared with only that one dead endpoint. This makes maintenance much easier.

## Written in Golang
The previous two self-hosted products were made with PHP and Python respectively. Usually, I wouldn't even mentioned choice of programming language, but I feel strongly about choice of Golang for this project. Golang is one of the most performant programming languages available today, and also enjoys having one of the best modern `http` and `crypto` libraries. When you think Google, you think **massive http traffic and top of the line encryption.** The quality of those two libraries is **a reflection of Google's strength** and I think is a perfect marriage for a product like Syncthing.

## Fully Open Source
While this is not unique to this product, I still feel that it's an important quality to mention. It's also created and maintained by a really talented and devoted guy, [Jakob Borg](https://github.com/calmh) (calmh), who has done an incredible job with this thing. My hats off to him and everyone else who has contributed to this project.

## Focus Only on Syncing Files
Unlike SeaFile or OwnCloud, **Syncthing's purpose is only to sync files!** There is no code to provide additional features like calendars, document editing, exc. This is great for me, because I never needed any of the other stuff, I was just interested in syncing files! I feel confident in knowing that every commit to this project only has one purpose: to improve the file syncing capabilities of the product.

## Cross Platform
Last but not least, cross platform! While the other products were also cross platform (for server side), it's great to know that **the exact same code base is being used on all platforms** since there is no separate code base for clients and servers. As a result, my Android app for Syncthing works perfectly. I guess Golang has a bit to do with this as well, but it also speaks volumes of the great design of Syncthing.

# TL;DR
Bottom line, if your looking for the most secure self-hosted file synchronization software out there, look no further! [Sycthing](https://syncthing.net) has you covered, go check it out!
