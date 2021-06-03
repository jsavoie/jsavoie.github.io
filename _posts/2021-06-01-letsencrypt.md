---
layout: post
title: "Lets Encrypt for internal hostnames"
---
# Lets Encrypt for internal hostnames

## Why and How?
One of the obvious issues with [lets encrypt](https://letsencrypt.org/) is how do we use it to create certificates for hostnames that don't exist on the internet?  Let me describe a scenario; a company has both an internal and external view of their
domain. The external hosts can easily be setup with lets encrypt, but the internal hosts do not appear in the external view and can't be validated. How do we make this work?  

## What you have now
Let's start with a few basics. I'm going to assume you're using bind, but any DNS server that supports different views/ddns/tsig will work. You probably have something like this:

```
view "internal" {
        match-clients { 
                10.10.0.0/8;
                192.168.0.0/16;
        };

        zone "exampledomain.com" IN {
                type master;
        };
};

view "external" {
        match-clients {
                0.0.0.0/0;
        };

        zone "exampledomain.com" IN {
                type master;
        };
};
```

In the internal view you might have a hostname like "infranet.exampledomain.com", which does not exist externally. How do we get Lets Encrypt to assign us a certificate for this hostname?

## RFC 2136
[RFC 2136](https://en.wikipedia.org/wiki/Dynamic_DNS) allows us to perform dynamic DNS updates. We will be using this in combination with certbot to update our DNS servers external zone with an ACME text record temporarily. 
This record will look like this:

```
_acme-challenge.infranet.example.com. 300 IN TXT "gfj9Xq...Rg85nM"
```

## What you need to install
First, we need to install certbot (and the rfc2136 plugin), under Debian you can accomplish this with the following:

```
# apt-get install certbot python3-certbot-dns-rfc2136
```

Next, we need to create a credentials file to use. This file will store your key and tell it which server to update. I would also recommend restricting who can access this file. Before we create this file we need
to generate a TSIG key. The easiest way to do this is to run "tsig-keygen certbot" from the server you have bind installed on. That will create this output.

```
key "certbot" {
        algorithm hmac-sha256;
        secret "9LwsqWeFOOXi3t1410VkeFLFV0l9YM9miFPZd4hNJCM=";
};
```

Save this file under /etc/letsencrypt/dns-creds.ini and chmod 600 it.

```
# Your authorative server
dns_rfc2136_server = 192.168.1.1
# TSIG key
dns_rfc2136_name = certbot
# TSIG secret
dns_rfc2136_secret = 9LwsqWeFOOXi3t1410VkeFLFV0l9YM9miFPZd4hNJCM=
# TSIG algorithm
dns_rfc2136_algorithm = HMAC-SHA512
```

## Changes you need to make
Now you will need to add support for this TSIG key to BIND. Edit your bind configuration file. Under Debian this will be /etc/bind/named.conf.local you will need to add the following:

```
key "certbot" {
        algorithm hmac-sha256;
        secret "9LwsqWeFOOXi3t1410VkeFLFV0l9YM9miFPZd4hNJCM=";
};
```

Next you will need to edit your views such that clients using the certbot TSIG appear in the external view.

```
view "internal" {
        match-clients {
                !key certbot;
                10.10.0.0/8;
                192.168.0.0/16;
        };
};

view "external" {
        match-clients {
                key certbot;
                0.0.0.0/0;
        };
};
```

After this you will need to allow the TSIG key permission to update the external zone. 

```
zone "exampledomain.com" IN {
        type master;
        update-policy {
                grant certbot name _acme-challenge.infranet.exampledomain.com. txt;
        };
};
```


At this point you can finish editing bind and can call "service bind9 reload" to reload the configuration.

## Running certbot

```
# certbot certonly --dns-rfc2136 --dns-rfc2136-credentials /etc/letsencrypt/dns-creds.ini \ 
        -d infranet.exampledomain.com
```

And if all has gone well, you should end up with a /etc/letsencrypt/renewal/infranet.exampledomain.com.conf like this

```
# renew_before_expiry = 30 days
archive_dir = /etc/letsencrypt/archive/infranet.exampledomain.com
cert = /etc/letsencrypt/live/infranet.exampledomain.com/cert.pem
privkey = /etc/letsencrypt/live/infranet.exampledomain.com/privkey.pem
chain = /etc/letsencrypt/live/infranet.exampledomain.com/chain.pem
fullchain = /etc/letsencrypt/live/infranet.exampledomain.com/fullchain.pem

# Options used in the renewal process
[renewalparams]
authenticator = dns-rfc2136
dns_rfc2136_credentials = /etc/letsencrypt/dns-creds.ini
```

## Your keys are here
From here you can load into whatever daemon configuration you need.

```
/etc/letsencrypt/live/infranet.exampledomain.com/privkey.pem
/etc/letsencrypt/live/infranet.exampledomain.com/fullchain.pem
```

## Dealing with renewals
If you're using apache or nginx, you may want to consider adding -i apache (or nginx) to the certbot command. Or you can edit the /etc/letsencrypt/renewal/infranet.exampledomain.com.conf after the fact and add the following under the [renewalparams]

```
installer = apache
```
or
```
installer = nginx
```

Debian by default will call reload on apache to rotate the logs once a week, so this may not be strictly necessary. Your mileage may vary, and you may want to setup something in /etc/letsencrypt/renewal-hooks/post if
you're using a daemon that doesn't regularly reload it's certificates.
