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
Starting out with the fuzzing. I reused my Boofuzz HTTP fuzzer to start testing new parameters. I spent around an hour fuzzing random fields in the header such as HOST, User-Agent, etc. Then it occured to me to fuzz more of the parameters that were sent as data. There are several things being passed, such as "Password", "Redirect", "NoJs", and "LoginButtonName". I started from the bottom and moved up, fuzzing "LoginButtonName" first with no results. Upon fuzzing "NoJs", I got a crash, and saw that EIP was overwritten:

![]({{site.baseurl}}/assets/images/integard_nojs/01.png)

![]({{site.baseurl}}/assets/images/integard_nojs/02.png)

I checked the Boofuzz results in SQLite to see how large the buffer was:

![]({{site.baseurl}}/assets/images/integard_nojs/03.png)

To replicate the crash, I set up the buffer in python:

![]({{site.baseurl}}/assets/images/integard_nojs/04.png)

As usual, the first step was just sending 1500 `A`'s, resulting in a crash that had `EIP` pointing to `41414141`. The next step was to find the offset:

![]({{site.baseurl}}/assets/images/integard_nojs/05.png)

![]({{site.baseurl}}/assets/images/integard_nojs/06.png)

![]({{site.baseurl}}/assets/images/integard_nojs/07.png)

![]({{site.baseurl}}/assets/images/integard_nojs/08.png)

I reused the `JMP ESP` from my last Integard exploit, located at `0xE087557`

![]({{site.baseurl}}/assets/images/integard_nojs/09.png)

I re-ran through the bad character check, and found the following bad characters:

`\x00\x26\x2F\x3D\x3F\x5C`

Due to the nature of Integard acting as a "filter", I've never been able to make reverse or bind shells work, so I set my payload to meterpreter, and got a shell running as `NT_AUTHORITY/SYSTEM`:

![]({{site.baseurl}}/assets/images/integard_nojs/10.png)

-----
# Conclusion
-----
I can't help but feel like this was low-hanging fruit, but it's still exciting to find a vulnerability that hasn't been found before, even if it's for a legacy program that's no longer in use. It was good practice and I learned a little bit about how to go about fuzzing `HTTP POST` requests appropriately.

-----
# Links
-----
[Integard](http://www.tucows.com/preview/519612/Integard-Home)

[Exploit](https://github.com/purpl3-f0x/exploit-dev/blob/master/nojs_integard.py)

CVE:

