---
published: false
---
<p><img alt="" src="https://i.imgur.com/Oa0ciBM.png" /></p>

<hr />
<p>Now it&#39;s time for the next pentest challenge in this series, Kioptrix 4!</p>

<hr />
<h2>Recon and enumeration:</h2>

<p><img alt="" src="https://i.imgur.com/zYMELc0.png" /></p>

<p>As always we start with an nmap scan, courtesy of my favorite enum tool Sparta, and can see some pretty common ports open, SSH, web, and SMB. I always like to check out SMB when I see it, because shares with poorly configured permissions are often something that can either expose sensitive information or could otherwise be leveraged for dropping files on the target.</p>

<p><img alt="" src="https://i.imgur.com/PO6rMaB.png" /></p>

<p>Doesn&#39;t look like there&#39;s much here to really go after. The shares here are default shares that usually can&#39;t be read and don&#39;t have anything in them. The nmap scan from before didn&#39;t really turn up the exact version, so I decided to grab the full version with metasploit, since my version of smbclient doesn&#39;t seem to enumerate versions anymore.</p>

<p><img alt="" src="https://i.imgur.com/b4gpyog.png" /></p>

<p>With the version enumerated now, I of course look to see if there are any exploits for this version of Samba:</p>

<p><img alt="" src="https://i.imgur.com/E9d5fEW.png" /></p>

<p>Doesn&#39;t look like there are any exploits for this exact version of Samba, so I&#39;m not going to pursue this. Next step is to check out the web page.</p>

<p><img alt="" src="https://i.imgur.com/Ny1CppG.png" /></p>

<p>A login page. I want to try a SQLi login bypass here but while I play with that, I get a gobuster scan going to see if there are any directories hidden.</p>

<p><img alt="" src="https://i.imgur.com/W857HW4.png" /></p>

<p>Only a few results. There&#39;s nothing of interest in /images, /index is just the login page, /logout is self-explanitory. So I check out john and robert. They are directories that contain php scripts that just take me back to the login page if I click on them, and the source code as visible in the browser is just the login page code. So I go back to trying SQLi.</p>

<p><img alt="" src="https://i.imgur.com/m6MCCo5.png" /></p>

<p>So trying the SQLi in both the username and password fields reveals that the user field has a form for santization by escaping single quotes. Leaving this syntax in the password field, I try a few more usernames:</p>

<p><img alt="" src="https://i.imgur.com/1PqsuZK.png" /></p>

<p>Logging in as John:</p>

<p><img alt="" src="https://i.imgur.com/x9bNvGt.png" /></p>

<p><img alt="" src="https://i.imgur.com/c5K6RmY.png" /></p>

<p>There we go, we have a password for robert. Not sure why loggin in as John didn&#39;t yield anything. So at first glance, this password looks like it is Base64 encoded because of the two &#39;=&#39; on the end, which is common for Base64-encoded strings.</p>

<p><img alt="" src="https://i.imgur.com/IDGmwDp.png" /></p>

<p>Or........maybe not. To rule out improper padding, I tried decoding it again with no =, one =, and three =, but nothing worked. So, taking a wild guess, I use this password to SSH into the box.</p>

<p><img alt="" src="https://i.imgur.com/IwZfYsM.png" /></p>

<p>And it works without any form of decoding. I guess the two = were put on the end to fool us, maybe.</p>

<p><img alt="" src="https://i.imgflip.com/1uca54.jpg" /></p>

<hr />
<h2>Privilege Escalation:</h2>

<p>So, looking closer at the login prompt that we landed in, it&#39;s obvious that we&#39;re stuck in a limited shell. I neglected to take a screenshot, but the only commands permitted were cd, echo, ll, lpath, ls, and help. Trying to chain commands with &amp;&amp;, ||, or ; resulted in &quot;forbidden syntax&quot; warnings, stating that 2 more violations would result in being kicked. Exceeding the warning limit results in the SSH session being terminated. It took me a while to figure out exactly how to escape this shell, but ultimately it was pretty simple.</p>

<p><img alt="" src="https://i.imgur.com/fvv4bpa.png" /></p>

<p>By typing &quot;help help&quot; it finally told me what type of shell it was. With this, I was able to research ways to escape &quot;lshell&quot; and quickly found my way out:</p>

<p><a href="https://www.aldeid.com/wiki/Lshell">https://www.aldeid.com/wiki/Lshell</a></p>

<p><img alt="" src="https://i.imgur.com/c32hfQu.png" /></p>

<p>Free of the limited shell, it&#39;s time to start doing the post-exploitation recon to find a way to get to root. First I checked the processes running as root:</p>

<p><img alt="" src="https://i.imgur.com/yVcm0ZH.png" /></p>

<p>Right away I see a very tempting attack vector; MySQL is running as root. But I don&#39;t have any way to login right now.</p>

<p>Thinking back to what I found on the web page, I decide to poke around in the www directory. Usually if a web application needs to connect to MySQL, there will be some PHP code that will login, and the username and password will be in that PHP file. We can&#39;t see this on the webpage because the server is executing the PHP code and won&#39;t show it in the browser when using &quot;inspect element&quot; or &quot;show page source&quot;.</p>

<p><img alt="" src="https://i.imgur.com/bEReKTQ.png" /></p>

<p>It didn&#39;t take long to find the file, and it was the &quot;robert.php&quot; file. The MySQL server root account doesn&#39;t have a password. This is a huge security flaw, because the MySQL server is actually capable of executing system commands, and since the SQL server is running as root, those commands are run as root.</p>

<p>Logging into the server is done from the terminal, with this command:</p>

<p><img alt="" src="https://i.imgur.com/BsHMvaK.png" /></p>

<p>Even though the -p option is being provided, you don&#39;t put the password in on this line, you press enter, at which point the server will ask for a password. I just press enter again, since there is no password, and I&#39;m logged into the database. I run a few commands to check if I an execute commands as root.</p>

<p><img alt="" src="https://i.imgur.com/n1PtSjV.png" /></p>

<p>Odd. The server is running as the root account, but it&#39;s running shell commands as robert. I&#39;m a bit confused, I really don&#39;t know what&#39;s going on. So it&#39;s back to researching. I search using terms along the lines of &quot;exploiting mysql running as root&quot;. I found a suggestion to use a completely different command, &quot;sys_exec()&quot;. I decide to test if this will really execute commands:</p>

<p><img alt="" src="https://i.imgur.com/PNlj3xI.png" /></p>

<p>I decided to use &#39;touch&#39; to see if I could make a file, and it allowed me to do so. What I should have done to check afterwards was &quot;ls -la&quot; to see who owned the file, and confirm that root owned it. Regardless, I moved on:</p>

<p><img alt="" src="https://i.imgur.com/6Bhh6VQ.png" /></p>

<p>The command above is adding robert to the admin group. This doesn&#39;t make robert a &quot;root&quot; user, but it does allow robert to run any command as root with &quot;sudo&quot;.</p>

<p><img alt="" src="https://i.imgur.com/Kr3C9yW.png" /></p>

<p>Just like that, we won, we have root! A quick look into the root directory reveals the &quot;flag&quot;:</p>

<p><img alt="" src="https://i.imgur.com/r6UwhnY.png" /></p>

<hr />
<h2>Summary:</h2>

<p>This one wasn&#39;t too hard, it encouraged some web directory enumeration, and explored how you can use processes running as root to gain administrative privileges. To be honest, I felt that Kioptrix 3 was harder, mostly because I was rusty on how to conduct SQLi, as well as identifying vulnerable urls/pages. There&#39;s only one more left in the Kioptrix series, and I&#39;ll be tackling that one very soon~</p>