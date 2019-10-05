---
published: true
title: Vulnhub - Misdirection
tags:
  - Pentesting - VulnHub
---
-----
# Intro
-----
Misdirection is a pretty simple OSCP-like machine that was very recently released by [InfoSec Prep's](https://discord.gg/YyWxwE) very own FalconSpy. He built it as some extra practice for people who are gearing up for OSCP and want something outside of the PWK labs. You can find it [here.](https://www.vulnhub.com/entry/misdirection-1,371/)

-----
# Part 1 - Recon
-----
This part was appropriately simple for beginners, not a lot of digging here necessary to get going really. I started as always with Sparta, my recon tool of choice, Sparta kicks off incrementally more intense nmap scans, and as it finds services, it kicks off extra enumeration tools automatically.

![]({{site.baseurl}}/assets/images/misdirection/01.png)

So there's not much going on here, got the usual HTTP server, SSH, and MySQL. The MySQL database was denying logins from anything except `localhost` so there was no point pursuing that.

On port 80 there was a generic looking website that seemed to revolve around e-voting. Initial poking around and source code examination didn't turn anything up, so I took a peek at what was running on port 8080. Just the default Apache2 page.

Time to start digging into the web server more. I ran:
`gobuster -u 10.0.0.141:8080 -w /usr/share/wordlists/dirb/directory-list-2.3-medium.txt`

![]({{site.baseurl}}/assets/images/misdirection/02.png)

I tested these directories one by one, but the only one that really had anything interesting was /debug:

![]({{site.baseurl}}/assets/images/misdirection/03.png)

This shell was running as `www-data` so was limited. It was also a bit slow and not really proper, so I sent myself a better shell with:

![]({{site.baseurl}}/assets/images/misdirection/04.png)

Once I got this shell I "upgraded" it with
```python
python -c 'import pty;pty.spawn("/bin/bash")'
```

-----
# Part 2 - Privilege Escalation 1
-----
Www-data is very limited in what it can do, so the first step is to priv esc up to a proper user. The name of the user on this system was `brexit`. I have to admit, I got stumped here for maybe an hour. There was a text-book priv esc technique that I completely skipped because I assumed this technique wouldn't apply to www-data: Checking sudo permissions:

![]({{site.baseurl}}/assets/images/misdirection/05.png)

On one hand, I thought this was a little too easy, but in a way, a little obscure too. I think it would be easy to assume that www-data would never have sudo permissions. Oh well, I'll chalk this up as "never leave anything to assumptions." With this, I had the first flag:

![]({{site.baseurl}}/assets/images/misdirection/06.png)

-----
# Part 3 - Privilege Escalation 2
-----
Now, I got stuck yet again here, running thru the usual priv esc motions, trying out some SUID binaries, looking for passwords, world-writable files etc. But there was something that evaded my eye for quite a while. When I finally realised what I was looking at, I realised I may have a way in:

![]({{site.baseurl}}/assets/images/misdirection/07.png)

*The passwd file is writable by the brexit group.*

My first attempts at inserting a user failed. I tried making one with no password, one with a plaintext password, but it wouldn't let me `su`, citing `Authentication Failure`.

So I had to look up a way to hash a password to throw into this file, and I found a way [here.](https://serverfault.com/questions/574586/what-is-the-purpose-of-openssl-passwd)

So I generated the password, and added onto the passwd file accordingly:

![]({{site.baseurl}}/assets/images/misdirection/08.png)

Then came the big moment, trying `su` for the fourth time:

![]({{site.baseurl}}/assets/images/misdirection/09.png)

![]({{site.baseurl}}/assets/images/misdirection/10.png)

-----
# Conclusion
-----
All in all I'd say that as long as you're paying attention, and leaving no stone unturned, and not making dumb assumptions like I did, this is a prety easy box. As an OSCP holder this reminded me that I really need to stop assuming something may not be a potential priv esc avenue and leave nothing to chance and always go thru all the steps. For people studying for OSCP, it's probably a good lesson for them to learn as well.

-----
# Links
-----
The InfoSec Prep Discord server - [https://discord.gg/YyWxwE](https://discord.gg/YyWxwE)

Misdirection on VulnHub - [https://www.vulnhub.com/entry/misdirection-1,371/](https://www.vulnhub.com/entry/misdirection-1,371/)

Understanding how passwords are stored in Linux - [https://www.slashroot.in/how-are-passwords-stored-linux-understanding-hashing-shadow-utils](https://www.slashroot.in/how-are-passwords-stored-linux-understanding-hashing-shadow-utils)