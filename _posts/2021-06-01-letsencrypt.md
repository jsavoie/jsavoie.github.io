## Using Lets encrypt for internal hostnames

One of the obvious issues with [lets encrypt](https://letsencrypt.org/) is how do we use it to create certificates for hostnames that don't exist on the internet?  Let me describe a scenario; a company has both an internal and external view of their
domain. The external hosts can easily be setup with lets encrypt, but the internal hosts do not appear in the external view and can't be validated.  How do we make this work?  

### Bind

Lets start with a few basics. I'm going to assume you're using bind, but any DNS server that supports different views will work.  You will probably have something like this:

view "internal" {
	match-clients { 
		10.10.0.0/8;
		192.168.0.0/16;
	};

	zone "exampledomain.com" IN {
		type master;
	};

view "external" {
	match-clients {
		0.0.0.0/0;
	};

	zone "exampledomain.com" IN {
		type master;
	};
}

In the internal view you might have a hostname like "infranet.exampledomain.com", which does not exist externally.  How do we get Lets Encrypt to assign us a certificate for this hostname?

### RFC 2136

[RFC 2136](https://en.wikipedia.org/wiki/Dynamic_DNS) allows us to perform dynamic DNS updates.  We will be using this in combination with certbot to update our DNS servers external zone with an ACME text record temporarily.
