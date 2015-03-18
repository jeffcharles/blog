---
title: On Apex Domains and Redirects
---
I've recently discovered that hosting content on apex domains is generally a bad idea and it's easy to shoot yourself in the foot attempting to redirect traffic. Apex domains are ones that are immediately below top-level domains (TLDs). For example, `beyondtechnicallycorrect.com` is an apex domain while `www.beyondtechnicallycorrect.com` is a subdomain. The reason it's a bad idea is that apex domains are typically constrained to serving up from a static ip address while subdomains can use canonical names (CNAMES) to refer to content on another domain. Generally, you're forced to use CNAMES to refer to content hosted by GitHub pages, Heroku, AWS Elastic Beanstalk, and a number of other services. Even if you're not using something requiring a CNAME at the present time, using a subdomain makes it much easier to move to a service that requires it.

In my efforts to redirect traffic from my apex domain to this subdomain, at one point I had set up a 301 permanent redirect from the apex domain to the subdomain. Unfortunately, I had misconfigured the redirect such that it worked for the root of my site but redirected traffic from all other pages to the root. I didn't discover this problem until very recently. Compounding this was that I didn't notice that I'd set my canonical urls to the apex domain. This broke a fair number of things that attempted to link to my blog since links to my posts would pick up a canonical link to the apex domain which when visted would redirect the user agent to root of the site.

There were a few lessons I learned from this:

1. Never use apex domains for hosting content
2. Never use 301 permanent redirects without time-limited cache-control headers to redirect traffic
3. Test redirects against non-root pages

My current favoured alternative to 301 redirects is the [CNAME flattening provided by CloudFlare](https://support.cloudflare.com/hc/en-us/articles/200169056-CNAME-Flattening-RFC-compliant-support-for-CNAME-at-the-root).
