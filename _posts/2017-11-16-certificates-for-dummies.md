---
layout: post
title:  "HTTPS, certificates and CAs for dummies"
categories: security theory
---

# HTTPS, certificates and CAs for dummies

The topic of HTTPS and certificates seems intimidating at first glance, and some vendors' marketing copy definitely don't help, but it all boils down to a few simple concepts.

## Public Key Cryptography / Asymetric Cryptography

The first concept to grasp is public key cryptography. The math involved is quite complex, but to use it, you only need to know this: In public key crypto, you have two keys. What you encrypt with one can only be decrypted by the other. Knowing this, you keep one private (and I mean really private), and you send the other to whoever wants to communicate with you. That way, they can send you secret messages that only you can decrypt.

You can also take a message you want to send (encrypted or not) and encrypt it with your private key. This is called a _signature_, and can only be decrypted with your public key. By adding it to messages you send, recipients can be sure that you did author the message.

## Certificates

Knowing this, you can understand that a certificate is really just a fancy wrapper around a public key, some data about the owner of the key, and a signature. The most important part of this data is the "Common Name" and "Alternate Names", which usually contain the domain(s) for which the certificate is to be used. Knowing this, every term around certificates suddenly become clear:

- A _Self-signed certificate_ is a certificate signed with its own key
- A _CSR_ or _Certificate Signing Request_ is a certificate without a signature
- A _Certificate chain_ is just a series of certificates, with a cert's signature having been done by the parent cert's key. (You can validate all this just by looking at the chain, no network needed)
- A _Wildcard certificate_ is a certificate that contains a wildcard in the domain name (like *.google.com), so it can be used for multiple subdomains.
- An _Extended Validation (EV) certificate_ is a certificate that also contains a company name, triggering that fancy display in your browser address bar.

## Certificate Authority

A Certificate Authority (CA for short) is just a fancy name for someone who signs certificates. They'll do some validations before, though, which is why we trust them. Of course, they have certificates themselves, which can be self-signed (for a "root" CA), or signed by another CA (for an "intermediate" CA).

## Trust

You can _trust_ a CA by _installing_ (basically putting it somewhere in your computer) its certificate. People don't usually do this themselves; most operating systems come with a few of them pre-installed, and some enterprises install theirs on computers they manage.

By trusting a CA, you trust their word on every certificate they sign. For example, when I visit https://www.google.com, I'm presented with a certificate chain. The first item in the chain is  a certificate for *.google.com, which is signed by someone named "Google Internet Authority G2". The second item in the chain is a certificate for "Google Internet Authority G2", which is signed by "GeoTrust Global CA". I trust "GeoTrust Global CA" (I have their certificate in that special place on my computer), so I deem the certificate for google.com valid, so I get that green padlock in my address bar.

## Why is it needed?

Everyone getting started with certificates gets annoyed at some point, and asks "Why do I need to do this? I just want to encrypt my traffic, I don't need to prove who I am!". Unfortunately, you can't have one without the other.

Imagine that Alice wants to send a secret message to Bob, and Eve wants to read it. She asks him to send her his public key, but Eve intercepts his message, and switches Bob's key for hers. Alice then sends her secret message, encrypted with what she thinks is Bob's key, but in reality it's Eve's. Eve then intercepts the secret message, reads it easily and re-encrypts it with Bob's real key before sending it to Bob. Alice and Bob never knew, but Eve read their message.

This is basically what can happen if we don't validate the keys we use, hence the need for CAs.

## SSL / TLS

_Secure Sockets Layer_ (SSL) is the old name for a technology now known as _Transport Layer Security_ (TLS). It's a protocol to assert the identity of client & server (altough the client part is optional) and establish a key to use for the rest of the session (Public key crypto is pretty expensive computationnaly, so we switch to symmetric crypto which is much faster).

## HTTPS

HTTPS is just plain old HTTP done over a TLS connection. There are a few tricks you need to be aware of when you're developing an HTTPS-enabled website (like insecure content), but it's pretty straightforward otherwise.

## Conclusion

So that's it! If you still have questions, just leave a comment and I'll try to answer it the best I can.
