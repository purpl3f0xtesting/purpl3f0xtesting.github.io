---
published: true
title: Exploit Dev - New Integard 0-day - CVE-2019-16702
tags:
  - Exploit Dev
  - Security Research
  - CVE-2019-16702
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
# Update - 12/4
-----
When submitting the exploit to Exploit-DB, they asked me to test this attack against Windows 7 and 10, as they apparently do not accept Windows XP-only exploits anymore.

I had to face the challenge of fighting with ASLR on both 7 and 10. I started by trying to find some JMP ESPs or CALL ESPs in non-ASLR modules, since `integard.exe` and `integard.dll` did not have ASLR enabled.

Sadly there were none.

I thought about doing a partial EIP overwrite to bypass ASLR, but none of the registers were pointing to my payload:

![](https://cdn.discordapp.com/attachments/648919825640325121/651867804324200488/unknown.png)

At this point I was thinking of throwing in the towel, because turning ASLR off would be meaningless, since ASLR is on by default on Windows 7 and 10, making the exploit useless on anything but my debugging VMs.

Then I remembered that I know of an SEH overwrite for this software! But the offset has probably changed, since this is not only a different vulnerability, but also different OS'es.

After finding the SEH offset of 2776 bytes, I was able to reliably overwrite SEH:

![](https://cdn.discordapp.com/attachments/648919825640325121/651879229734256770/unknown.png)

The POP POP RET being used is part of `integard.exe` and doesn't use ASLR, but it does have a null byte in it, so I have to bury my shellcode somewhere in the original crash buffer and do some back jumping:

![](https://i.imgur.com/5w01Oe9.png)

The nSEH jumps backwards by 48 bytes, hitting a short NOP sled. That slides down to the `\xEB\x09` which makes a short jump forward. Why? Well let's analyze the "bigBackJump" code

```nasm
59              POP ECX
FE CD           DEC CH
FE CD           DEC CH
FE CD           DEC CH
FF E1           JMP ECX
E8 F2 FF FF FF  CALL (Relative -10)
```

By hitting the jump forward, execution will jump to the `Call (Relative -10)`. This `CALL` will direct execution to the `POP ECX`.

The reason for this technique is that making a `CALL` will `PUSH` a return address onto the stack, which is then `POP`'d into ECX. ECX is then decremented by 768 bytes, which is then jumped to. This will make sure it's jumping 768 bytes backwards from the `CALL`.

As shown two screenshots above, the meterpreter payload is buried in the original crash buffer, which is now just a giant NOP sled. This will lead execution all the way to the meterpreter payload.

![](https://cdn.discordapp.com/attachments/648919825640325121/651879774108516362/unknown.png)

To my happy surprise, this same payload works on Windows 7 as well:

![](https://cdn.discordapp.com/attachments/648919825640325121/651889071681044511/unknown.png)

The exploit was re-written to function as a multi-option exploit so it can change the payload based on user input:

![](https://i.imgur.com/eYosc1I.png)

![](https://i.imgur.com/MDQdaLH.png)

-----
# Disclosure
-----
When I tried to seek out Race River to inform them of this vulnerability I found that their websites are all 404'd. It appears as if this vendor may have gone out of business.

-----
# Conclusion
-----
I can't help but feel like this was low-hanging fruit, but it's still exciting to find a vulnerability that hasn't been found before, even if it's for a legacy program that's no longer in use. It was good practice and I learned a little bit about how to go about fuzzing `HTTP POST` requests appropriately.

-----
# Links
-----
[Integard](http://www.tucows.com/preview/519612/Integard-Home)

[Exploit](https://github.com/purpl3-f0x/exploit-dev/blob/master/nojs_integard.py)

[CVE](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2019-16702)
