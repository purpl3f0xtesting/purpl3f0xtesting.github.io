---
published: false
---


-----
# Intro
-----
This was probably one of the more complex Vulnserver exploits that I made, requiring lots of jumping around, stack adjustments, and the infamous alphanumeric character restrictions. I took this as an extra opportunity to practice some manual encoding but also sped things up with [this nice script here](https:github.com/ihack4falafel/Slink/blob/master/Slink.py)

While on one hand I found this more complex than the HP NNM exercise in the CTP course, I simultaniously found it easier, probably just because I was familiar with the concepts at this point, and also knew how to fix a mis-aligned stack (not to mention how to spot it).

By the way, for anyone who may be confused about the new look in the screenshots, I started using [x64dgb](https://x64dbg.com/) because of some of the nicer features here and there.

-----
# Part 1 - Starting with the SEH overwrite
-----

Long-time readers of my original blog should be familiar with the basics of an SEH overwrite, so I'll skip over the finer details and just get right to the important bits.

SEH overwrite occurs when sending 4000 "A"s, and the offsets for SEH and nSEH are around 3500:

![]({{site.baseurl}}/assets/images/lter/01.png)

![]({{site.baseurl}}/assets/images/lter/02.png)

