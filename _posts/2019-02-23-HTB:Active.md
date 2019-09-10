---
published: true
layout: post
title: HTB - Active
tags:
  - Pentesting
  - HackTheBox
---
<p><img alt="" src="https://i.imgur.com/hw8YWUq.png" style="height:89px; width:306px" /></p>

<p><img alt="" src="https://i.imgur.com/BWB8Qkl.png" style="height:284px; width:351px" /></p>

<p>Created by <a href="https://www.hackthebox.eu/home/users/profile/302">eks</a> and <a href="https://www.hackthebox.eu/home/users/profile/2984">mrb3n</a></p>

<hr />
<p>Let me preface this by saying that this was my favorite box on HackTheBox because it was one of the most real-world-like box that I&#39;ve encountered so far. The vulnerabilities exploited here can be exploited in the real world and lead to the compromise of domain servers, possibly even the domain controller, which would of course lead to an entire domain compromise. Hacking this box helped me understand the need for running services under special-made service accounts with very strong passwords and limited permissions, as well as shining a light on the weaknesses of legacy Windows domain controllers.</p>

<hr />
<p>As per usual, I started out with my initial scanning and enumeration. I&#39;ve taken a liking to a very powerful tool called Sparta that takes a target or targets and runs nmap against them. As it discovers open services, it will automatically enumerate those services for you, such as running smbenum against SMB or Samba servers, or running Nikto against web servers.</p>

<p><img alt="" src="https://i.imgur.com/8ocYCYQ.png" style="height:413px; width:1025px" /></p>

<p>Right away I was able to tell that this was going to be a domain controller because of ports 53(DNS), and 3268(LDAP). Ports 135, 139, and 445 didn&#39;t explicitly indicate a domain controller to me because any Windows box can have these services running. After researching some unfamiliar ports, my hunch was confirmed. Port 464, which nmap lists as "kpasswd5", is a protocol used by Kerberos for changing or setting passwords. Kerberos is an authentication protocol used by Windows Active Directory.</p>

<p>Whenever I see SMB on a server I always like to poke at that first, because it can sometimes yeild some juicy information or even some limited file access to the server. To enumerate the shares manually I used</p>

<p>smbclient -L //10.10.10.100</p>

<p><img alt="" src="https://i.imgur.com/JBtA2QI.png" style="height:312px; width:687px" /></p>

<p>Here we see the shares available on the server. For the password I just left it blank and pressed enter, and this is what I got. I usually don&#39;t bother with ADMIN$, C$, or IPC$, as these are default shares that usually don&#39;t allow anonymous login. The "Users" share did not allow anonymous login either. However, the replication share seemed to allow anonymous access:</p>

<p><img alt="" src="https://i.imgur.com/XbSsgdA.png" style="height:111px; width:451px" /></p>

<p>Very good. Initial digging around indicated that this share contained exactly what it said it did; a replication of the domain settings. This is something worth investigating, because legacy Active Directory domain controllers will sometimes store domain passwords in these configurations. Lo and behold, that is exactly what I stumbled on:</p>

<p><img alt="" src="https://i.imgur.com/g4aUMer.png" style="height:79px; width:877px" /></p>

<p>This "Groups.xml" file doesn&#39;t look interesting at first, but taking a look at it is definitely worth it. I downloaded the file to my attacking machine using "get groups.xml" in the smbclient session, which saved it to the current directory that my terminal was in. I opened the XML file in Kali&#39;s text editor Gedit since it has syntax highlighting, making it easier to see what we want.</p>

<p><img alt="" src="https://i.imgur.com/0040aIS.png" /></p>

<p>What we&#39;re interested in is the field "cpassword". This is an encrypted domain password that was stored by group policy, and it seems to belong to a domain user called "SVC_TGS" in the "active.htb" domain. Usually anything encrypted is beyond our reach, but legacy domain controllers stored these in a highly insecure fashion, and the private key is publicly available to decrypt this password.</p>

<p><a href="https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-gppref/2c15cbf0-f086-4c74-8b70-1f2fa45dd4be">https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-gppref/2c15cbf0-f086-4c74-8b70-1f2fa45dd4be</a></p>

<p><img alt="" src="https://i.imgur.com/36kc5RC.png" /></p>

<p>There are several tools and scripts out there to decrypt cpasswords, but Kali has a nice built-in tool to do it for you with no hassle:</p>

<p><img alt="" src="https://i.imgur.com/hNUq643.png" style="height:55px; width:1006px" /></p>

<p>As shown above, the cpassword was effortlessly derypted to "GPPstillStandingStrong2k18". With this password, I logged into the SMB shares again, only this time accessing the "Users" share, where I was able to get the first flag of the box.</p>

<p><img alt="" src="https://i.imgur.com/z5sVveJ.png" style="height:310px; width:679px" /></p>

<p><img alt="" src="https://i.imgur.com/aaoGhPV.png" /></p>

<p><img alt="" src="https://i.imgur.com/JVjZLQf.png" /></p>

<p>Half-way done! Now it&#39;s time for priviledge escalation. Right away we already have a hint. The username of the domain user we got is "SVC_TGS". I remember from my classes about Windows Server, as well as my Security+ study, that Kerberos had something to do with "ticket granting". So I did some research to see if there was any relation between TGS and Active Directory. What I found is that TGS stands for "Ticket Granting Server", which refers to a component in Kerberos authentication.</p>

<p><img alt="" src="https://www.roguelynn.com/words/explain-like-im-5-kerberos/images/Kerb.001.jpg" style="height:424px; width:566px" /></p>

<p>Doing some more research into this led me to something called SPNs, or "Service Principal Name". Services that support Kerberos authentication require a Service Principal Name associated with them to point users to the appropriate resource for connection. Kerberose uses SPNs to associate a service witha service login account, allow a client application to request that the service authenticate an account even if the client does not have the account name. In other words, it maps a service running on a server to an account that it is running as, so that it can do or accept kerberos authentication.</p>

<p>What does that mean to us? Well, any valid domain user can request a kerberos ticket for any domain service. Kerberos tickets are signed to verify authenticity, and those tickets are signed with the hashed password of the service account. Interesting yes? Now we see why service accounts are important. If anything is running as the built-in admin, then the hashed password of the admin is being tossed around the network for anyone to see. So how do we get it?</p>

<p><a href="https://github.com/SecureAuthCorp/impacket">Enter SecureAuthCorp&#39;s Impacket suite of python tools.</a></p>

<p>These python scripts are tools used for interacting with many network protocols, but we&#39;re of course most interested in how it can interact with Kerberos.</p>

<p>Perhaps the most obvious of these tools is "GetUserSPNs.py", since SPNs link services to service acccounts. The syntax I used to invoke the tool was:</p>

<p>./GetUserSPNs.py active.htb/svc_tgs -dc-ip 10.10.10.100 -request</p>

<p><img alt="" src="https://i.imgur.com/LWMUrxc.png" style="height:350px; width:1130px" /></p>

<p>As pictured above, we see that CIFS(SMB) is running as Administrator. This is extremely bad news for the target, but extremely good news for us. That is because of one simple fact: The built-in administrator account for domain controllers is also the domain admin. In other words, getting the password for this account grants us the keys to the entire kingdom.</p>

<p>The string that starts with $krb5tgs$ is our Kerberos TGS hashed password. This is a special hash that is different from LM/NTLM hashes that Kerberos uses for signing tickets.</p>

<p>Time to get crackin&#39;. My prefered tool is hashcat, because I have a fairly high-powered gfx card in my host computer that can run through the "rockyou.txt" wordlist in short order. But for us to use hashcat, we have to tell the tool what format hash this is. Luckily, there is an online guide for this:</p>

<p><a href="https://hashcat.net/wiki/doku.php?id=example_hashes">https://hashcat.net/wiki/doku.php?id=example_hashes</a></p>

<p>I hit CTRL+f in my browser and searched for TGS, and got a familiar result:</p>

<p><img alt="" src="https://i.imgur.com/yI0r4gh.png" style="height:48px; width:453px" /></p>

<p>The exampl shows that it starts with $krb5tgs$ just like our captured hash. The number "13100" is the code we will need to provide hashcat with. I copied the hash into a file on my host and named it "hash.txt" just to keep it from cluttering my terminal prompt, and then fired up my Windows version of hashcat so I can leverage my GPU.</p>

<p><img alt="" src="https://i.imgur.com/vnVrc9n.png" style="height:45px; width:807px" /></p>

<p>Hashcats resulting output is a little messy and sometimes it&#39;s easy to miss the result, but all you have to do is look at the end of the provided hash and you&#39;ll see the clear-text password:</p>

<p><img alt="" src="https://i.imgur.com/zOrPxxC.png?" /></p>

<p>It&#39;s Game Over for our target. We have the domain admin password. In the real world, this would be catastrophic if this was exploited by the wrong person. But since we&#39;re just doing an HTB box, let&#39;s finish it off by getting the final flag:</p>

<p><img alt="" src="https://i.imgur.com/8Z69KWn.png" style="height:422px; width:681px" /></p>

<p><img alt="" src="https://i.imgur.com/OfQmCay.png" /></p>

<hr />
<h1>Final Thoughts</h1>

<p>This box was problably the most fun just because it was so realistic and didn&#39;t have too much of a gimmicky, CTF feel to it. It defintely expanded my knowledge of how Active Directory works, at least on older domains, and the dangers of not configuring your services properly.</p>

<p>These days, Group Policy no longer stores passwords in the way shown here, so at the very least, that is something that shouldn&#39;t concern sysadmins too much. However, I can&#39;t stress enough that services should be configured to use service accounts with strong passwords to avoid being cracked, and if they are cracked, these accounts should have the least priviledges possible to stop an entire domain compromise.</p>

<p>Requesting SPNs using valid credentials is normal behavior in a network, so an attack such as this is virtually undetectable; it won&#39;t set off IDS/IPS, won&#39;t get blocked by a firewall, and won&#39;t generate any anomalous logs. Worst of all, by default, new domain controllers only have 1 account, the built-in administrator, and by default, SMB is already running, and of course runs as admin, which means that the behavior seen in this box wasn&#39;t something that had to be deliberately set-up, this was default out-of-the-box configuration, which makes such a mistake all the more easy to overlook.</p>

<p>Finally, I want to share this video that helped me tremendously while researching this topic:</p>

<p><a href="https://youtu.be/PUyhlN-E5MU">https://youtu.be/PUyhlN-E5MU</a></p>
