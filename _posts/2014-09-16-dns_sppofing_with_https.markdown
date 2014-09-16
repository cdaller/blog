---
layout: post
title:  "DNS Spoofing with HTTPS and real Verisign Certificate"
date:   2014-09-16 10:16:15
categories: security
tags: [security, java]
image:
  feature: abstract-12.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
comments: true
share: true
---

## Scenario:

(Web)Application uses a webservice of a server that is hosted in the cloud and uses multiple
servers for failover or loadbalancing.

Application is secured by Java Security Manager.

## First problem:

Java uses infinite caching for DNS entries if the security manager is enabled.
So if the webservice's failover is activated, the service cannot be reached anymore, as the
new dns entry to the failover server is never resolved to the new ip adress.

## Solution to first problem:

Disable the infinite DNS caching in the JVM

```
-Dsun.net.inetaddr.ttl=30
```

## Second Problem

Now the dynamic resolving works, but the new ip address is further investigated by the
security manager (java.net.SocketPermission#impliesIgnoreMask):
a reverse lookup is done for the new ip address and the returned hostname
is checked if it is allowed. As the reverse lookuped hostname is a virtual server in the cloud
(e.g. ec2-abc-def-ghi-jkl.eu-west-1.compute.amazonaws.com) the security manager refuses the
connection as there is no explicit rule for this host).

## Solution to second problem:

Add all possible hostnames from the failover system to the security manager's policy file.
The hostname is as relyable as the ip address and might change in near future. So there is only one
solution: add the lines to the security policy file (e.g. catalina.policy)

~~~
permission java.net.SocketPermission "*.eu-west-1.compute.amazonaws.com", "resolve";
permission java.net.SocketPermission "*.eu-west-1.compute.amazonaws.com:443", "connect";
~~~

## Third problem:

the TLS-certificate of the webservice changes quite frequently (like once a year).
Whenever the certificate is not found in the truststore of the Java VM, a connection is refused
by the JVM.

## Solution to third problem:

Adding the root CA to the trusted certificates of the JVM. The root CA is valid for 20 years, so
even long running systems will not have any problems.

## Attacker scenario:

* An attacker uses DNS spoofing to fake the webservice server's hostname.
* The attacker is the owner of an amazon cloud system with a certificate signed by verisign CA.

When the attacker sets the ip address of the webservice to his fake amazon server following happens:

* DNS lookup finds a new IP address
* the new IP address is resolved to the fake amazon cloud system.
* the security manager allows access to the amazon cloud (due to the wildcard socket permission)
* Java VM checks the certificate and finds a valid verisign certificate that is trusted (in the truststore)
* the system talks to the fake system and receives fake data (possibly transmitting username/password as well).

## Conclusion:

Lots of security mechanisms in place (certificate, tls, security manager) but due to real world
problems the complete system is quite easily compromisable!

The real world problems are
* multiple servers as failovers are relocated into the cloud. This leads to
  Anybody is able to rent a system
  in that cloud and is able to buy a "trusted" certificate for it. This would not be possible if the
  failover systems are located inside the webserver's domain (no trusted certificate available for
  an attacker).
* the server's certificate changes too often, so the CA's certificate is used for trust checks.
  But even if the server's certificate is added to the truststore this will not help if
  a certificate of a CA that sells certificates for other systems are also contained in the truststore.
