---
categories:
  - "security"
  - "encryption"
keywords:
  - "openssl"
tags:
  - "openssl"
  - "encryption"
  - "crypto"
  - "tls"
  - "reference"
  - "script"
date: "2017-02-03"
title: "x509 Certificate Manual Signature Verification"
thumbnailImagePosition: "left"
thumbnailImage: "/images/thumb/thumb-tls.png"

aliases:
  - "/x509verify"
---

While going through the manual of [openssl](https://www.openssl.org/docs/man1.1.0/), I thought it would be a good exercise to understand the signature verification process for educational purposes. As a fruit to my labor, I would also develop a simple script to automate the process.
<!--more-->

<!-- toc -->

# What Does "Signing a Certificate" Mean?
A **Certificate Authority (CA)** utilizes **asymmetric cryptography** to form a key pair. This key pair is usually referred to as the *public* key and the **private** key. Messages encrypted with one key, can only be decrypted with the other key. The public key is advertised and known to all, however the private key is kept secret and should only been known by the CA. When a **Certificate Authority (CA)** signs a certificate, what it actually does is **hash** the certificate then **encrypt that hash** with it's private key. This hex code is then embedded into the certificate along with information on how it was derived called the `Signature Algorithm`. Therefore, in order for one to **verify** that a certificate was signed by a specific CA, we would only need to possess the following:

> * the public key of the CA (issuer)
> * the **Signature** and **Algorithm** used to generate signature

# The Verification Process
Obtaining the two listed items above is not a difficult task. Let's examine how we would do this manually

## Obtaining the Issuer's Public Key
The issuer of a x.509 certificate should have it's own x.509 certificate (that's also signed if it's an Intermediate CA, or slef signed if Root CA) to prove it's authenticity. Once obtaining this certificate, we can extract the public key with the following `openssl` command:
```bash
openssl x509 -in /tmp/rsa-4096-x509.pem -noout -pubkey > /tmp/issuer-pub.pem
```
## Extracting the Signature
Now let's take a look at the signed certificate. The signature (along with algorithm) can be viewed from the **signed certificate** using `openssl`:
```bash
openssl x509 -in /tmp/ec-secp384r1-x509-signed.pem -noout -text
...
Signature Algorithm: sha384WithRSAEncryption
     ad:cd:dd:71:cf:8a:79:2c:2d:86:5b:a2:08:7e:00:e0:28:30:
     17:f3:b0:75:a6:12:46:fc:f8:75:9c:4a:57:2f:89:f9:8a:c6:
     03:e4:58:fa:c4:53:13:a7:67:18:60:17:fc:0d:e4:1a:ba:86:
     fd:14:a7:a5:cb:98:a3:d3:ea:40:a8:71:eb:cd:07:b7:e3:16:
     33:8d:ad:45:9f:ca:c7:ca:77:0a:ec:10:42:2a:85:8d:79:de:
     f0:a8:af:38:bc:70:2d:c1:5c:d7:b7:bb:a3:e8:5c:44:d3:89:
     a6:f0:00:8a:b7:1f:df:fd:5b:2a:9e:d3:9c:5f:ec:ad:7a:b8:
     5b:3b:6c:f5:8a:9c:4f:e5:84:d1:c9:ba:af:d5:9c:75:e8:43:
     17:2f:3c:be:e2:49:ef:6a:e1:28:73:f6:57:aa:4e:ec:a3:f9:
     7d:6f:c5:ce:83:20:fc:51:e0:3b:9f:7b:51:06:d3:29:28:08:
     19:99:2b:ca:fb:9e:7c:e9:5e:38:89:ec:71:f8:a4:aa:be:e0:
     1f:65:ed:24:d7:c0:13:aa:0c:f4:b6:f1:24:62:85:38:ac:05:
     f0:d0:de:69:91:dd:98:9d:9a:46:10:30:ac:6a:6a:20:d5:3c:
     c6:10:b4:2b:aa:d9:38:58:07:f2:41:12:23:45:1c:a8:ca:41:
     11:f2:fd:dd:6f:1a:83:ba:4e:ff:ef:b2:0a:7c:32:9e:0d:6b:
     01:17:76:69:59:9c:25:90:54:3d:08:f3:e9:3d:48:14:11:1d:
     09:59:6e:a9:22:1b:f5:c2:80:ee:4d:e7:f5:df:0a:b2:27:f3:
     05:68:3e:23:99:64:bb:d9:dc:25:95:1a:5f:b5:be:7d:0e:b9:
     47:e4:c6:34:0c:22:0a:ad:d0:c5:f9:be:cb:e3:c9:ca:71:d3:
     ea:f6:62:33:2f:08:c4:1e:d6:1e:aa:4b:5c:dd:bc:66:43:18:
     d9:56:68:93:cd:3a:ef:51:b3:67:8a:e9:bd:07:04:95:23:10:
     6c:42:2e:0d:28:a8:b4:c1:81:ba:1b:7e:9d:5c:48:16:6c:04:
     de:53:e6:7d:aa:cb:52:dd:8f:55:8d:09:1f:cf:32:70:ec:81:
     ea:00:07:52:8d:95:2b:b8:b6:ac:b6:36:0a:fd:02:f3:0e:42:
     20:1f:ce:1b:ba:c7:f4:aa:01:53:fb:af:9c:b5:e2:d5:61:7e:
     3c:2b:16:de:22:2b:43:13:f8:5a:6e:ab:c2:0a:85:96:6f:7b:
     b3:ef:49:0f:5f:bf:01:e9:52:f8:de:44:12:23:f5:e6:78:de:
     f6:e0:40:8e:97:89:45:92:f7:73:ee:41:71:7a:7c:de:79:10:
     f2:db:c3:86:85:3c:ac:fe
```
In the above example, we can tell by the algorithm name `sha384WithRSAEncryption` that [SHA-384](https://en.wikipedia.org/wiki/SHA-2) is the cryptographic hash function used and that it was encrypted via [RSA](https://en.wikipedia.org/wiki/RSA_(cryptosystem)).

We can take this hex and dump it to disk as a binary like this:
```bash
# extract hex of signature
SIGNATURE_HEX=$(openssl x509 -in /tmp/ec-secp384r1-x509-signed.pem -text -noout -certopt ca_default -certopt no_validity -certopt no_serial -certopt no_subject -certopt no_extensions -certopt no_signame | grep -v 'Signature Algorithm' | tr -d '[:space:]:')

# create signature dump
echo ${SIGNATURE_HEX} | xxd -r -p > /tmp/cert-sig.bin
```

## Decrypting the Signature (RSA)
Now that we have both the encrypted dump of the signature as well as the public key of the issuer. We can decrypt the signature like so:
```bash
openssl rsautl -verify -inkey /tmp/issuer-pub.pem -in /tmp/cert-sig.bin -pubin > /tmp/cert-sig-decrypted.bin
```

We can now finally view the hash with `openssl`
```bash
openssl asn1parse -inform der -in /tmp/cert-sig-decrypted.bin
0:d=0  hl=2 l=  65 cons: SEQUENCE          
2:d=1  hl=2 l=  13 cons: SEQUENCE          
4:d=2  hl=2 l=   9 prim: OBJECT            :sha384
15:d=2  hl=2 l=   0 prim: NULL              
17:d=1  hl=2 l=  48 prim: OCTET STRING      [HEX DUMP]:0C8EDBABEA1EAF2DB57A2A7CC29E2AAA9103167659572DFE10C0D7FF62BA74EC5481D6F049B956775E2BA3E11C5A046A
```
The digest is in the last line: `0C8EDBABEA1EAF2DB57A2A7CC29E2AAA9103167659572DFE10C0D7FF62BA74EC5481D6F049B956775E2BA3E11C5A046A`

## Hashing the Original Signature
Now that we got a hash of the orginal certificate, we need to verify if we can obtain the same hash by using the same hashing function (in this case SHA-384). In order to do that, we need to extract just the body of the signed certificate. Which, in our case, is everything but the signature. The start of the body is always the first digit of the second line of the following command:
```bash
openssl asn1parse -i -in /tmp/ec-secp384r1-x509-signed.pem
    0:d=0  hl=4 l= 856 cons: SEQUENCE          
    4:d=1  hl=4 l= 320 cons:  SEQUENCE     
...     
```

We can extract this data and store it to disk like so:
```bash
openssl asn1parse -in /tmp/ec-secp384r1-x509-signed.pem -strparse 4 -out /tmp/cert-body.bin -noout
```

Finally, we can run this through the same hashing function to determine the digest
```bash
openssl dgst -sha384 /tmp/cert-body.bin
SHA384(/tmp/cert-body.bin)= 0c8edbabea1eaf2db57a2a7cc29e2aaa9103167659572dfe10c0d7ff62ba74ec5481d6f049b956775e2ba3e11c5a046a
```

As you can see, **both hashes match**, so we can now confirm that:

> /tmp/rsa-4096-x509.pem **did sign** /tmp/ec-secp384r1-x509-signed.pem

# The Finished Script
Now that we went through that manual process, I have put together a script which undergoes a similar process to determine the valididty of a signature. Now in the real world, your browser will be tasked with validating a chain of certificates not just the certificate that signed your domain's cert. This script only checks if CERT A signed CERT B.
{{< alert warning >}}
This script should not be relied upon in any shape, way or form. Simply educational.
{{< /alert >}}

Here is an example of how to use this script
```bash
# supply the signed cert (-s) and issuer's cert (-i) in pem encoding
sh x509-cert-sig-verify.sh -i /tmp/rsa-4096-x509.pem -s /tmp/ec-secp384r1-x509-signed.pem

Issuer Certificate: /tmp/rsa-4096-x509.pem
Signed Certificate: /tmp/ec-secp384r1-x509-signed.pem
Verified OK

# when no arguments supplied
sh x509-cert-sig-verify.sh
  Missing REQUIRED option -i <ISSUER_CERTIFICATE>

  usage: x509-cert-sig-verify.sh [OPTIONS...]

  -i issuer-cert  required  path or url of issuer x509 cert
  -s signed-cert  required  path or url of signed x509 cert
  -v              optional  verbose logging

  *WARNING*
  This script was made for educational purposes.
  VERIFICATION MAY BE WRONG, USE AT YOUR OWN RISK!
```

If you made it this far down the post, you are awarded the source of the script!

<script src="https://gist.github.com/gbolo/807e1c05335843758abe80935730537a.js"></script>
