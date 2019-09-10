---
published: false
---
<p>This one is going to be fairly short and sweet. It was a pretty simple box found over at vulnhub.</p>

<p><a href="https://www.vulnhub.com/entry/kioptrix-level-1-1,22/">https://www.vulnhub.com/entry/kioptrix-level-1-1,22/</a></p>

<p>Vulnhub is a site that hosts downloadable VMs that are CTF-style challenges. You&#39;ll need VMware to host them, and the drive space, but the upshot is that you don&#39;t have to worry about network latency or VPNs.</p>

<p><img alt="" src="https://i.imgur.com/paG9ZqV.png" style="height:213px; width:912px" /></p>

<p>Initial scans with SPARTA don&#39;t show too much, but there are some pretty common attack vectors here. Samba is usually pretty ripe for the picking, and having a look around websites can reveal some vulnerabilities.</p>

<p>Unfortunately, there isn&#39;t a whole lot running on the web server side of things. I did find some directories with &quot;gobuster&quot; but in the end they were just rabbit holes, or in other words, dead ends that were meant to waste time and/or frustrate the attacker.</p>

<p>So let&#39;s go back to Samba. My version of Kali doesn&#39;t properly grab the banner for some reason, so I have to use a metasploit module to get the version:</p>

<p>auxiliary/scanner/smb/smb_version</p>

<p><img alt="" src="https://i.imgur.com/cRgTZ2e.png" /></p>

<p>This stood out to me like a sore thumb. I&#39;ve seen this version of Samba in my PWK labs, and I know for a fact it&#39;s vulnerable. A quick search using &quot;searchsploit&quot; in Kali yielded many results, but the only result that worked for me was <a href="https://www.exploit-db.com/exploits/10">this exploit here.</a></p>

<p>Compiling it was very straight forward, I used the command &quot;gcc 10.c -o Sambaexploit&quot; and got a fully compiled binary.</p>

<p>Execution consists of providing the target (Choices of Linux, MacOSx, SunOS, and &quot;other&quot;), and the IP address of the target.</p>

<p><img alt="" src="https://i.imgur.com/JBbVkm4.png" /></p>

<p>It was just that simple to get root. A more robust shell was spawned by typing &quot;/bin/bash -i&quot;</p>

<p><img alt="" src="https://i.imgur.com/s0vuB0D.png" /></p>

<p>Looks a bit better now.</p>

<p>There is a &quot;flag&quot; on the box, or rather, just a congradulatory message that can only be read by the root account.</p>

<p><img alt="" src="https://i.imgur.com/v5ePega.png" /></p>

<p>The same vulnerability is also exploitable with metasploit, using the module &quot;linux/samba/trans2open&quot;. The meterpreter payload didn&#39;t work but the &quot;linux/x86/shell/reverse_tcp&quot; did. It resulted in the same outcome, instant-root.</p>

<p>My enumeration turned up a second vulnerability in the version of &quot;mod_ssl&quot; that the server was running. It was running version 2.8.4, which is vulnerable to a buffer overflow attack. There is an exploit out there eloquently named &quot;OpenFuck&quot;, but for the life of me I could NOT get it to work. What I was able to determine is that this is just a very old exploit written in obsolete C code, because every attempt to compile it failed. To make sure I wasn&#39;t going crazy, I &quot;cheated&quot; and looked at other people&#39;s write-ups of this box. In their write-ups, they can compile this exploit with no issues and also get instant-root using it, but these write-ups are from several years ago.</p>

<p>So that&#39;s it, short and sweet like I said. Kioptrix1 is meant to be fairly simple and straight-forward, an entry-level box, so I didnt expect more. There are higher-level versions that are more difficult however, and I&#39;ll be tackling those soon.</p>