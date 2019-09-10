---
published: true
title: OSCP Review
tags: Pentesting
---
<p>On April 15th I received the best email I&#39;ve gotten in a long time; a confirmation from Offensive Security that I had passed my PWK exam and obtained my Offensive Security Certified Professional (OSCP) certification! 15 months in the making, it took 2 attempts to get it. A lot of time went into practicing techniques learned throughout the PWK course, taking advantage of not just the PWK labs, but the labs at <a href="https://www.hackthebox.eu">HackTheBox</a> as well, with some additional challenges found at <a href="https://www.vulnhub.com/">VulnHub.</a></p>

<h2>Prerequisites</h2>

<hr />
<p>The required knowledge to jump into the Pentesting With Kali Linux isn&#39;t too expansive, but it does help to have more than the basic requirements. As a minimum, OffSec suggests that you understand:</p>

<ul>
	<li>TCP/IP networking</li>
	<li>Basic Linux know-how</li>
	<li>Basic scripting in Bash and/or Python/Perl</li>
</ul>

<p>Outside of this, I would recommend that you also have a solid understanding and level of comfort working with the Windows command line as well. I don&#39;t really think that a ton of &quot;pre-study&quot; is necessary to get through this course, it is after all, an entry-level pentesting course. If you have any network admin or sysadmin experience, that alone should be enough to get started.</p>

<p>The PWK course will cover all the basics of pentesting, such as information gathering, enumeration, various types of attacks, custom exploit development, post-exploitation recon, and privilege escalation. Not everything covered in the course will necessarily appear in the labs or exam, but don&#39;t skip any sections just because it seems out-of-scope, it&#39;s all good to learn, and if you plan on stepping up to the OSCE later on, some of this will come in handy.</p>

<h2>The Labs</h2>

<hr />
<p>The labs for the PWK course are very good in my opinion. They may be shared labs, but there are so many targets to go after that you shouldn&#39;t have too many instances of running into someone else attacking the same machine. The difficulty ranges from laughably easy, with machines that can be rooted in very few steps, all the way to the insanely hard, with boxes that can take up an entire day just to get low-priv access. Nothing is really given to you that lays out the lab in great detail, so each target you face is essentially a blind challenge; you won&#39;t know how difficult it will be until you get to it. I actually like this approach, because it prevents you from limiting yourself to only the &quot;easy&quot; boxes, and you may find yourself conquering targets that were far more difficult than you thought you were capable of handling.</p>

<h2>My Exam Experience</h2>

<hr />
<p>For this blog post, I&#39;ll only be detailing my 2nd attempt, since my 1st attempt was a complete failure that isn&#39;t worth talking about. Simply put, I was still too under-skilled at research and enumeration, and missed a lot of easy hints that should have pointed me in the right direction.</p>

<p>I took my 2nd exam on April 10th, with a start time of 0930. I woke up at 0800, took a long, calming shower, ate breakfast, drank plenty of water, and then sat down waiting for the email to arrive. I got the exam email with the VPN package and the instructions in it, got everything set up, ran the screen sharing application, got the webcam going, and I was off!</p>

<h3><strong>First Blood</strong></h3>

<p>I decided to start with what would be the &quot;easiest&quot; for me, and went right after the buffer overflow challenge. I have practiced this many times, and even wrote a blog post here detailing one of my practice exercises. On the OSCP exam, you don&#39;t need to fuzz the target for vulnerabilities; they provide a proof-of-concept that already has the vulnerable port number, command, and buffer that will crash the application. You are provided an exploit-development Windows 7 VM with a debugger installed to work on your exploit. After a few tiny mis-steps, I had this box in just over an hour, putting me up to 25/100 points.</p>

<h3><strong>Into The Flames</strong></h3>

<p>Wanting to keep my momentum, I decided that my second target would be the most difficult; the other 25-point box. Getting just low-privilege access on this one was quite the grind. It took just over 2 hours before I had a some-what functional shell. For reasons I never determined, my shells on this target were either so laggy that every command took over a minute to execute (even simple things like cd or pwd), or I would have a shell with no prompt, and just a blank screen, which wasn&#39;t as laggy but made it hard to remember where I was. I took a break here to stretch and breathe, and came back and sunk another 2 hours into attempts at privilege escalation. After having no luck, I remembered some advice that a friend had given me, which was to avoid spending more than 2 hours on any target, and moved on. At this point, the low-priv access had bumped my score up to 35/100, or at least that&#39;s what I believe partial-credit is worth.</p>

<h3><strong>Getting My Second Root</strong></h3>

<p>Moving on to a lower-difficulty box, I was able to get low-priv access in just around an hour and a half, somewhere in that range, prompting another short break to gather my thoughts and regain focus. Privilege escalation on this machine wasn&#39;t quite what I expected to have to do, and I only figured it out after doing some thorough post-exploitation-recon to discover some services that weren&#39;t in my initial enumeration. Some research on the application and a bit of trail-and-error eventually led to root access on this target. Score: 55/100.</p>

<h3><strong>Taking It Easy</strong></h3>

<p>To avoid completely cooking my brain, I decided to give myself a break in the form of setting my sights on the easiest of the lab machines. I actually really enjoyed this one because it contained a vulnerability that I remember reading about several months ago, so I could tell that OffSec is definitely updating their exam machines to be more relevant to today&#39;s IT landscape. I only had a few small roadblocks on this one that were easily overcome, and I had root access in about 45 minutes. Score: 65/100. Almost passing.</p>

<h3><strong>Sealing The Deal</strong></h3>

<p>Time to get my last points needed to pass. There was only one target left, another medium-difficulty box. Getting the inital low-priv foothold on this one was quite frustrating. I found something that was obviously vulnerable, but all of my attempts to manually exploit it were failing. After about an hour of not making progress, I bit the bullet and used my 1-time allowable usage of Metasploit. Lucky for me that I had saved it, because Metasploit just so happened to have a module for the very thing I was attacking, and it was instantly successful in exploiting it, even though my attempts to manually exploit it failed. Privilege escalation wasn&#39;t too hard, I believe I went from low-priv to root in 20 minutes. Score: 85/100, passing.</p>

<h3><strong>Circling Back</strong></h3>

<p>At this point I was only 11 hours into my exam, and it was 2030 at night. I had enough points to pass, my documentation thus far was solid, so I went back to the one box I hadn&#39;t rooted yet and resumed my attempts at privilege escalation. Sadly, even after another 3 hours, I just couldn&#39;t get it. At 2330, I informed the proctor that I was done, ended my exam early, and went to bed.</p>

<h3><strong>Reporting Day</strong></h3>

<p>I had put in a lot of time practicing documentation to make the reporting process go by more smoothly, and went into this part of the exam feeling very relaxed. Over a span of just under 5 hours, I had assembled my 32-page report, proofread it twice, made some tweaks, and sent it off to OffSec as per the reporting instructions.</p>

<h3><strong>Results</strong></h3>

<p>I submitted my report on Thursday, April 11th. OffSec states that it can take up to 5 business days to get your results back. I didn&#39;t have to wait quite that long. On Monday, April 15th, at around 0200 in the morning, I got an email informing me of my success.</p>

<p><img alt="" src="https://i.imgur.com/KR9XfOp.png" /></p>

<p>It didn&#39;t take long for me to claim my Acclaim badge and get it into my email signature at work, as well as share it on Twitter and LinkedIn to prove my new status as an OSCP~<br />
The only thing I&#39;m waiting on now is the physical copy of my certification.</p>

<h2>Looking Forward</h2>

<hr />
<p>I&#39;m eager to start my OSCE journey, but I will work on my skills in web application pentesting and advanced exploit development before-hand. I have taken a few stabs at the OSCE registration challenge but I can&#39;t really make any head-way on it, which tells me that my skill set isn&#39;t sufficient yet. I&#39;m also quite interested in the new AWAE course that is now avaible online, instead of being in-person exclusive, and since that one doesn&#39;t require a registration challenge and focues heavily on web app pentesting, I&#39;m considering that one as well, since web app hacking is something I need to really work on for the real-world. I&#39;ll continue researching both the CTP course for the OSCE, and the AWAE course for the OSWE, and determine which one I want pursue.</p>

<p><a href="https://www.youracclaim.com/badges/7a10d10d-3c04-4a96-98a7-e23c8f668f57/public_url" target="_blank"><img alt="" src="https://i.imgur.com/Hx9DoPN.png" style="height:352px; width:352px" /></a></p>