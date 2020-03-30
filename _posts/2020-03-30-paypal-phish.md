---
published: true
title: A Dangerously Convincing Phish
tags:
  - Phishing
---
-----
# Intro
-----
Today I got a phishing email sent to me that, on the surface, was pretty obvious to me that it was fake, but as I dug deeper, it revealed itself to be a terrifyingly convincing phishing scheme (except for one little detail).

-----
# Breakdown
-----
To start, let's show the email:

![]({{site.baseurl}}/assets/images/phish/p1.png)

At first glance, it looks sort of legitimate. Paypal's actual email is "service@paypal.com", and this fake one has the *appearance* of "service@intl.paypal.com", which could be confused as legitimate, until you notice the true domain. 

`customer-ppcuredtlsd.n9eu@lahapaitu.com`

This should be Red Flag 1. The second red flag is the recipient field:

`costumers@scured-paypaldtls.com`

Spelling mistakes, and an attempt to hide the actual recipient emails. Already it's looking bad, but let's get to the meat of the email.

![]({{site.baseurl}}/assets/images/phish/p2.png)

The body of the email looks fairly legitimate. The logo is correct, and so is the grammar and spelling. At first glance, it looks like it's real.

![]({{site.baseurl}}/assets/images/phish/p3.png)

Here is the resulting webpage if you click the link. Again, this looks fairly legitimate. It's a near perfect clone of Paypal's website. But here's where things become a little too obvious again:

![]({{site.baseurl}}/assets/images/phish/p4.png)

All this effort to make a good fake website and the scammer drops the ball on trying to disguise the URL. No effort at all. Still, to the uninformed user who may be panicking over their paypal being locked, this could be overlooked.

I provided some obviously fake credentials to see what would happen:

![]({{site.baseurl}}/assets/images/phish/p5.png)

It accepts my "login" and moves to the next screen, which still looks pretty legitimate so far. Even the font is correct.

![]({{site.baseurl}}/assets/images/phish/p6.png)

I'm presented with another new screen that made my heart drop:
It's not just after your paypal creds, it's asking for your full personal info:

![]({{site.baseurl}}/assets/images/phish/p7-2.png)

Hopefully asking for this much info would scare someone off, but sadly, it probably doesn't. And the fun just continues:

![]({{site.baseurl}}/assets/images/phish/p8.png)

As if stealing your paypal login wasn't enough, now they want your credit card info.

![]({{site.baseurl}}/assets/images/phish/p9.png)

Times two.

I provided fake info for all of these, and was presented with this page:

![]({{site.baseurl}}/assets/images/phish/p10.png)

But that only hung around for about 5 seconds before redirecting to:

![]({{site.baseurl}}/assets/images/phish/p11.png)

***The real paypal page!!***

This is dangerous because it further convinces the victim that all is well, and offers them a chance to log into their "newly restored" account.

-----
# Burner links?
-----
I attempted to revisit the link to get more evidence, but it just redirected me directly to paypal again, without any of the fake prompts.

At first I thought it was a cookie or something, so I cleared cookies and data, and still got sent direct to paypal. Then I tried the link on two completely different browsers, and got the same result.

![]({{site.baseurl}}/assets/images/phish/p12.png)

I fired up Burp Suite and caught the request in Repeater. The first link hits a `301 Moved Permanently` that leads to a different URL.

![]({{site.baseurl}}/assets/images/phish/p13.png)

That URL hits a `302 Found` redirect, that sends you to yet another URL.

![]({{site.baseurl}}/assets/images/phish/p14.png)

...and *THAT* URL also hits a `302 Found` redirect that takes you to paypal.

-----
# Conclusion
-----
Aside from the hideously obvious email addresses and website URLs, this phish is fairly well done. A near perfect clone of Paypal's website aimed at harvesting quite a lot of info. After this post is published, I'll be reporting this to Paypal, Mozila, and Google, in the hopes that Paypal can spread some awareness, and that Firefox and Chrome will blacklist the domain.
