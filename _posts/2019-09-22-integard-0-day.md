---
published: false
title: Exploit Dev - New Integard 0-day
tags:
  - Exploit Dev
  - Security Research
---

-----
# Intro
-----
After getting some tips from a friend about a way of finding 0-days, I decided to return to Integard Pro v2.2.0.9026 and fuzz some different parameters in the HTTP POST header. After about an hour, I found a new buffer overflow that allowed me to overwrite EIP. There are no exploits or documentation that cover this specific parameter, so in my mind, this counts as a 0-day, even if it's still for a legacy program that is probably never used anymore. There's not a ton of new information here so this should be a pretty short post.

-----
# Vanilla EIP overwrite in the "NoJs" parameter
-----