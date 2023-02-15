---
layout: post
title: "Using Lets Encrypt with EZproxy"
---
# Lets Encyrpt with EZproxy

## Why?
I've found there's a general lack of documentation about how to use Lets Encrypt with EZproxy, and importantly how to set it up to use wildcard certificates. We will build on the concepts covered
by my previous article <a href="/2021/06/01/letsencrypt.html">Lets Encrypt for internal hostnames</a>, by using DNS rfc2136 ACME challenges. If you haven't read the previous article, please do so now.

## Getting a wildcard certificate
Perhaps you didn't know, but you can have Lets Encrypt issue wildcard certificates. What you need is to have a DNS record like this:

```
proxy		IN	A	192.168.1.10
*.proxy		IN	A	192.168.1.10
```

This record will cause any lookups to to that subdomain to resolve to 192.168.1.10, such as testsite.proxy.yourdomain.com. Once you have done this you can run the following command from the host you
intend to run the EZproxy daemon from:

```
certbot certonly --dns-rfc2136 --dns-rfc2136-credentials /etc/letsencrypt/dns-creds.ini -d "*.proxy.yourdomain.com" -d "proxy.yourdomain.com"
```

A possible hangup; make sure you have an update-policy allowing _acme-challenge.proxy.yourdomain.com to be updated:

```
update-policy {
	grant certbot name _acme-challenge.proxy.yourdomain.com. txt;
};
```

## Configuring EZproxy
EZproxy can be configured to take advantage of this, rather than using an individual port per database resource. There are several advantages of doing this, but a major one is simplifying firewall configurations
and not having to worry about potential network egress filtering remote clients may or may not have. A snipet from what you EZproxy config.txt might look like here:

```
Name proxy.yourdomain.com
Option ProxyByHostname
```

Additional documentation about this can be found <a href=https://help.oclc.org/Library_Management/EZproxy/EZproxy_configuration/Migrate_to_Proxy_by_Hostname>here</a>

EZproxy itself doesn't have a way to specify the location of a certificate from it's config.txt. Instead this is handled through the web administrative page. We however want to automate
this process. The locations of the certificates created by certbot are here:

```
/etc/letsencrypt/live/proxy.yourdomain.com/privkey.pem
/etc/letsencrypt/live/proxy.yourdomain.com/cert.pem
/etc/letsencrypt/live/proxy.yourdomain.com/chain.pem
```

We'll need to do is create symlinks to these files from within the EZproxy install directory. Within the EZproxy install directory there should be a "ssl" sub directory. Within that
subdirectory you'll find a file named "active", which has the current number of the certificate in use  The certificate (and key) are stored using these filenames:

```
00000008.cnf
00000008.key
00000008.err
00000008.csr
00000008.crt
00000008.ca
```

For example, if we want to link our new certificate as "9", we'd create the following symlinks:

```
ln -s /etc/letsencrypt/live/proxy.yourdomain.com/privkey.pem 00000009.key
ln -s /etc/letsencrypt/live/proxy.yourdomain.com/cert.pem 00000009.crt
ln -s /etc/letsencrypt/live/proxy.yourdomain.com/chain.pem 00000009.ca
```

This means we'll be missing the .cnf, .err, and .csr files. You can populate these if you like, but they are not necesary. We can activate our new certificate by issuing the following commands:

```
echo "9" > active
systemctl restart ezproxy
```

## Restarting EZproxy on certificate renewal
EZproxy will not restart on it's own once a certificate is renewed. Instead we need to create a hook to tell EZproxy to restart and reload it's certificates. Create a file /etc/letsencrypt/renewal-hooks/post/ezproxy
and add these lines. Make sure this file is executable (chmod +x).

```
#!/bin/sh
systemctl restart ezproxy
```

## Testing
I would suggest using <a href="https://www.ssllabs.com/ssltest/index.html">ssllabs.com</a> to test your certificate and to identify any other potential problems you might have with your configuration.
