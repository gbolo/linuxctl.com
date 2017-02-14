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
date: "2016-12-05"
title: "OpenSSL - Reference"
thumbnailImagePosition: "left"
thumbnailImage: "/images/thumb/thumb-openssl.png"

aliases:
  - "/openssl"
---

[OpenSSL](https://www.openssl.org/) is quite and extensive project. The docs for the `cli` ([openssl commands](https://www.openssl.org/docs/man1.1.0/apps/)) gives you an overview on just how many things you can do with openssl. Knowing openssl is ***essential*** in the security field. I will use this post as a reference for frequent things I do with openssl and update it when needed.
<!--more-->

<!-- toc -->

{{< alert warning >}}
All `openssl` commands listed below were taken from manual of version [**1.1.0**](https://www.openssl.org/docs/man1.1.0/) and tested with the same version.
{{< /alert >}}

# Generating Key Material
Commands related to the generation of private key material for asymmetric encryption. Remember to always protect your private keys!

## Elliptic Curve (EC)
The only Elliptic Curve algorithms that [OpenSSL currently supports](https://wiki.openssl.org/index.php/Command_Line_Elliptic_Curve_Operations) are Elliptic Curve Diffie Hellman (ECDH) for key agreement and Elliptic Curve Digital Signature Algorithm (ECDSA) for signing/verifying. Reasons to use EC over RSA include:

> * **Shorter keys** needed for equivalent protection compared to RSA.
> * **Lower CPU/Memory requirements**.
> * **Better Performance**.

References: [ecparam](https://www.openssl.org/docs/man1.1.0/apps/ecparam.html) | [ec](https://www.openssl.org/docs/man1.1.0/apps/ec.html)
```bash
# list available EC curves
openssl ecparam -list_curves

# create EC parameters (using a curve from above list) and a private key but discard parameters
openssl ecparam -name secp384r1 -genkey -noout -out /tmp/ec-secp384r1-key.pem

# create EC parameters (using a curve from above list) and a private key
openssl ecparam -name secp384r1 -genkey -out /tmp/ec-secp384r1-key.pem

# view EC parameters
openssl ecparam -in /tmp/secp384r1-key.pem -noout -text

# encrypt the private key with a symmetric key algorithm
# options are: -aes128|-aes192|-aes256|-camellia128|-camellia192|-camellia256|-des|-des3|-idea
openssl ec -in /tmp/ec-secp384r1-key.pem -aes256 -out /tmp/ec-secp384r1-aes256-key.pem

# remove encryption of a private key
openssl ec -in /tmp/ec-secp384r1-aes256-key.pem -out /tmp/ec-secp384r1-key.pem

# extract public key from private key
openssl ec -in /tmp/ec-secp384r1-key.pem -pubout -out /tmp/ec-secp384r1-pub.pem
```

## Rivest-Shamir-Adleman (RSA)
RSA is the most common type of Public/Private Key. For that reason, it is the most compatible.

References: [genrsa](https://www.openssl.org/docs/man1.1.0/apps/genrsa.html) | [rsa](https://www.openssl.org/docs/man1.1.0/apps/rsa.html)
```bash
# generate rsa private key of length 4096 bits
openssl genrsa -out /tmp/rsa-4096-key.pem 4096

# encrypt the private key with a symmetric key algorithm
# options are: -aes128|-aes192|-aes256|-camellia128|-camellia192|-camellia256|-des|-des3|-idea
openssl rsa -in /tmp/rsa-4096-key.pem -aes256 -out /tmp/rsa-4096-aes256-key.pem 4096

# remove encryption of a private key
openssl rsa -in /tmp/rsa-4096-aes256-key.pem -out /tmp/rsa-4096-key.pem

# extract public key from private key
openssl rsa -in /tmp/rsa-4096-key.pem -pubout -out /tmp/rsa-4096-pub.pem
```

# x509 Certificates
An X.509 certificate is a digital certificate used in PKI for authentication. x.509 is the format (standard) used for TLS. The certificate includes the public key along with additional data to help provide authentication.

## Creating and Viewing x509 Certificates
Commands related to the generation and viewing of x509 certificates.

References: [x509](https://www.openssl.org/docs/man1.1.0/apps/x509.html)
```bash
# create a x509 certificate signed by a private key generated above
openssl req -new -x509 -key /tmp/ec-secp384r1-key.pem -out /tmp/ec-secp384r1-x509.pem -days 365

# view this certificate in human readable format
openssl x509 -in /tmp/ec-secp384r1-x509.pem -text -noout

# view the public key of a x509 certificate in pem format
openssl x509 -in /tmp/ec-secp384r1-x509.pem -noout -pubkey

# convert the encoding of a certificate (text/bin)
# the default is pem, so we need to specify in/outform to der
openssl x509 -in /tmp/ec-secp384r1-x509.pem -outform der -out /tmp/ec-secp384r1-x509.der
openssl x509 -in /tmp/ec-secp384r1-x509.der -inform der -out /tmp/ec-secp384r1-x509.pem
```

## Certificate Signing Request (CSR)
Commands related to the generation, viewing, and verification of Certificate Signing Requests.

References: [req](https://www.openssl.org/docs/man1.1.0/apps/req.html) | [verify](https://www.openssl.org/docs/man1.1.0/apps/verify.html)
```bash
# create a CSR from a private key with sha384 digest
openssl req -new -sha384 -key /tmp/ec-secp384r1-key.pem -nodes -out /tmp/ec-secp384r1-csr.pem

# view a CSR in human readable format
openssl req -in /tmp/ec-secp384r1-csr.pem -noout -text

# sign the CSR with a CA certificate
openssl x509 -req -sha384 -days 360 -in /tmp/ec-secp384r1-csr.pem -CA /tmp/ca-rsa-4096-x509.pem -CAkey /tmp/ca-rsa-4096-key.pem -CAcreateserial -out /tmp/ec-secp384r1-x509-signed.pem
```

## Verification of x509 Certificates
There are two things I usually want to verify about a signed x.509 Certificate:
* Did this private key create this certificate?
* Did this other x509 certificate sign this certificate?

We can verify that an x509 certificate was generated by a specific private key by comparing the public keys. We can do this for both EC and RSA keys.

References: [dgst](https://www.openssl.org/docs/man1.1.0/apps/dgst.html)
```bash
# compare the hashes of the public keys
openssl x509 -in /tmp/ec-secp384r1-x509.pem -noout -pubkey | openssl dgst -sha1
openssl ec -in /tmp/ec-secp384r1-key.pem -pubout | openssl dgst -sha1
```

For signature verification, we also need the certificate of the issuer.

References: [verify](https://www.openssl.org/docs/man1.1.0/apps/verify.html)
```bash
# verify the signature
openssl verify -verbose -CAfile /tmp/ca-rsa-4096-x509.pem /tmp/ec-secp384r1-x509-signed.pem
```

**UPDATE** - I have written a blog post that includes a script for verifying signatures. Can be found here: [x.509 Certificate Signature Verification](/x509verify)

# Symmetric Encryption
Openssl can be used to encrypt/decrypt data using either keys derived from passphrase or pre-made keys.

References: [enc](https://www.openssl.org/docs/man1.1.0/apps/enc.html)
```bash
# list available symmetric ciphers
openssl enc -ciphers

# encrypt a file with aes-256-cbc
openssl enc -aes-256-cbc -salt -in /tmp/rsa-4096-key.pem -out /tmp/rsa-4096-key.pem.aes

# decrypt the file created above
openssl enc -d -aes-256-cbc -salt -in /tmp/rsa-4096-key.pem.aes -out /tmp/rsa-4096-key.pem
```

# Performance Testing
OpenSSL is also a very convenient way to test the encryption capabilities of your server. This is really helpful when deciding what ciphers to use, and the performance impact of them.

References: [speed](https://www.openssl.org/docs/man1.1.0/apps/speed.html)
```bash
# basic aes tests software only (no hardware cpu instruction set aes-in)
openssl speed aes-128-cbc aes-192-cbc aes-256-cbc

# enable hardware cpu instruction aes-in
openssl speed -evp aes-128-cbc aes-192-cbc aes-256-cbc

# increase threads
openssl speed -elapsed -evp aes-128-cbc -multi 4

# list available suites (-help triggers error and displays suites)
openssl speed -help
```
