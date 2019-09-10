---
published: true
title: VulnHub - Kioptrix 4
tags:
  - Pentesting
  - VulnHub
---
<p><img alt="" src="https://i.imgur.com/modEYtM.png" /></p>

<hr />
<p>The final box in the Kioptrix series is here! This one was the hardest by far, and every bit of advancement came only after a fair deal of research, head scratching, and frustration. Getting the initial foothold took many steps, some of which I&#39;ve never done before, but getting root was extremely easy by comparison.</p>

<hr />
<h2>Scanning and enumeration:</h2>

<p>As always we start with an nmap scan to find open ports and their versions so we can start poking around.</p>

<p><img alt="" src="https://i.imgur.com/yrIDLCn.png" /></p>

<p>The only ports open are ports 80 and 8080, with the second port just being an alternative web page on the same server. The web page on port 80 wasn&#39;t anything exciting, it was just a blank page that said &quot;It works!&quot; When trying port 8080 I get a &quot;403 FORBIDDEN&quot; error. I move on and try using gobuster to find any hidden directories, but that didn&#39;t turn up anything either.</p>

<p><img alt="" src="https://i.imgur.com/tVv0ECD.png" /></p>

<p>Moving on to the Niko results, it appears that this server may or may not be vulnerable to a mod_ssl exploit. I tried two differen&#39;t exploits for this version of mod_ssl but neither of them worked. I figured that since these weren&#39;t working, this probably wasn&#39;t the way in.</p>

<p>I&#39;ll admit that at this point I got pretty stuck. I had no idea where to go. I had already looked at the source code for the port 80 page but nothing had stuck out. My fault for not actually reading every line, because the answer was right there.</p>

<p><img alt="" src="https://i.imgur.com/RiKC1Y1.png" /></p>

<p>That comment, that stands out from everything else in bright green, has a url in it. I feel kinda stupid for not seeing that sooner and wasting upwards of half an hour using gobuster and dirbuster looking for other pages.</p>

<hr />
<h2>Initial foothold:</h2>

<p><img alt="" src="https://i.imgur.com/acAzONb.png" /></p>

<p>Looks like it&#39;s some sort of chart drawing program. I looked at a few of the tabs and didn&#39;t see anything of interest, so I looked up &quot;pChart 2.1.3&quot; and found that it has a directory traversal vulnerability that allows the viewing of system files.</p>

<p><a href="https://www.exploit-db.com/exploits/31173">https://www.exploit-db.com/exploits/31173</a></p>

<p><img alt="" src="https://i.imgur.com/1SnVyrS.png" /></p>

<p>Following what I found, I decided to use the example given to look at the passwd file.</p>

<p><img alt="" src="https://i.imgur.com/3AFqkzS.png" /></p>

<p>Looks like the vulnerability is present on this installation. I don&#39;t see anything particularly interesting here, but it&#39;s usually a good idea to look at the passwd file on Linux targets.</p>

<p>Here is where I got stuck again. I wasn&#39;t entirely sure what I should go looking for on the system. For lack of a better idea I wanted to go looking at the web server files, there&#39;s usually something hidden there. There&#39;s a problem though. This isn&#39;t Ubuntu or Red Hat, it&#39;s FreeBSD, and the files for apache aren&#39;t in their usual place, so I have to go looking for some hints on where those files are.</p>

<p><a href="https://www.freebsd.org/doc/handbook/network-apache.html">https://www.freebsd.org/doc/handbook/network-apache.html</a></p>

<p><img alt="" src="https://i.imgur.com/xp3vcTK.png" /></p>

<p>This is a good place to go look first, as I had remembered that there was an unexplored page on the server, one that was blocking me from accessing it. The httpd.conf file would give some clues as to why.</p>

<p><img alt="" src="https://i.imgur.com/GX3iJwp.png" /></p>

<p>As I was scrolling down I found this little bit of potentially useful information, so I make note of it and keep scrolling.</p>

<p><img alt="" src="https://i.imgur.com/dl4EUxB.png" /></p>

<p>Then I find exactly what I was looking for. The page hosted on 8080 is blocking me because my &quot;user-agent&quot; isn&#39;t set to &quot;Mozilla4_browser&quot;. To test this out, I turn on my FoxyProxy Firefox add-on, and use burp to intercept and modify my http GET request like so:</p>

<p><img alt="" src="https://i.imgur.com/FQZTcNg.png" /></p>

<p>I replaced everything in the User-Agent field with &quot;Mozilla/4.0 Mozilla4_browser&quot;, then clicked &quot;Foward&quot;, and switched back to my browser.</p>

<p><img alt="" src="https://i.imgur.com/DE1UhGv.png" /></p>

<p>It worked! It let me in. But there&#39;s a problem. Every link I click just takes me right back to the &quot;403 FORBIDDEN&quot; error page, and trying to intercept and modify every packet I send can be annoying. I look up how to modify Firefox to change its user-agent and found a way:</p>

<p><img alt="" src="https://i.imgur.com/I4vFwNE.png" /></p>

<p>Making a simple modification to the &quot;about:config&quot; file allowed me to manipulate how Firefox presented its user-agent information. After I ddi this I was able to browse the forbidden page without using burp anymore.</p>

<p><img alt="" src="https://i.imgur.com/I1peUjD.png" /></p>

<p>Again I find another application I don&#39;t recognize, and I don&#39;t really see anything of interest here that I can abuse, so off to google I go for the 30th time!</p>

<p>Turns out that &quot;phptax&quot; is vulnerable to command injection that could lead to executing system commands on the victim.</p>

<p><a href="https://www.exploit-db.com/exploits/21665">https://www.exploit-db.com/exploits/21665</a></p>

<p>I followed the directions here and didn&#39;t have any luck. I was able to open ports on the target, but my netcat wouldn&#39;t connect. Using nmap, I found a strage phenomenon occuring. The ports was showing up as &quot;filtered&quot; instead of closed. I assumed there must be some sort of local firewall enabled on the target, so I tried a reverse shell, instructing the target to connect back to me on a specific port, but that didn&#39;t work either.</p>

<p>Now, I try to avoid using Metasploit when I can, but in instances like these, it seems that it&#39;s the only way in.</p>

<p>I found a metasploit module named &quot;exploit/multi/http/phptax_exec&quot; that can exploit the vulnerability in the link above, but with better results.</p>

<p>Where things got confusing is when I tried to set the payload to a bind shell or reverse shell. Metasploit would report that the exploit was completed but no shell was caught. This was another point where I got stuck for a while, but then I discovered that if I ran the exploit without specifying a payload, it would automatically chose a payload for me, and this payload worked.</p>

<p><img alt="" src="https://i.imgur.com/hfE2Yig.png" /></p>

<p>At long last, after all this struggle, we have a shell on the server as the user &quot;www&quot;.</p>

<hr />
<h2>Privilege Escalation</h2>

<p>The priv esc on this box was super easy. I found my way in running what is usually the very first thing I run during post-exploitation recon:</p>

<p><img alt="" src="https://i.imgur.com/Z7KyU2r.png" /></p>

<p>It&#39;s always worth it to look up OS versions for vulnerabilities, but personally, I usually save that for last, as I&#39;m not always comfortable running kernel exploits. In the real world, a kernel exploit gone wrong could knock a box offline, and you don&#39;t want to do that to a client. I try to treat these challenges like the real-world if possible, so I ended up spending 45 minutes exploring other priv esc routes before I finally circled back to this.</p>

<p><a href="https://www.exploit-db.com/exploits/28718">https://www.exploit-db.com/exploits/28718</a></p>

<p>The very first google result was this reliable-looking exploit. I tried to compile it on my attacking maching, but it was a no-go, it was giving lots of errors. Given that FreeBSD is old, I&#39;m guessing the code for the exploit is old. I checked to see if the victim had &quot;gcc&quot; installed, which is a utility for compiling C code, and it did. So I copied the source code over to the victim:</p>

<p><img alt="" src="https://i.imgur.com/VFv4f9V.png" /></p>

<p>On the left is my attacking machine of course, and on the right is the target. On my box, I set up netcat to <strong><em>listen </em></strong>and send the file to anything that connects to it, and used netcat on the target to <strong><em>connect to </em></strong>my box, and pipe whatever it receives into a file named 28718.c (it doesn&#39;t have to be the same filename but keeping it consistent avoids confusion.</p>

<p>Now that it&#39;s on the victim I compile it with a very simple command, &quot;gcc 28718.c -o pwn&quot;, which compiled it into an executable. Running the binary resulted in a full compromise:</p>

<p><img alt="" src="https://i.imgur.com/84BsU5y.png" /></p>

<p><img alt="" src="https://i.imgur.com/PUf0NEP.png" /></p>

<hr />
<h2>Summary:</h2>

<p>This one was a lot more complicated than the rest, and most of the work went into just getting that initial shell. As a mercy, root was so easy that it would have been effortless, had I not wasted time looking for alternative methods. All in all, I liked it still, and how it taught me how to bypass restrictions on web pages to get to previously inaccessable directories.</p>

<p>There are tons of challenges on Vulnhub, so I&#39;ll be taking on something more challenging over the weekend! At the time of writing this I have 6 days to go until I take the OSCP exam, so I need to crank up the difficulty to get myself in the right mindset for the exam. Thanks for viewing!</p>