---
published: true
title: VulnHub - Kioptrix 3
tags:
  - Pentesting
  - VulnHub
---
p><img alt="" src="https://i.imgur.com/mvaODmx.png" style="height:400px; width:400px" /></p>

<hr />
<p>Here we are with Kioptrix level 3! This one was significantly more challenging than the last two if you exploited it manually, but there were some ways to automate the process to get the initial foothold to make things easier.</p>

<hr />
<h2>Recon and initial enumeration:</h2>

<p><img alt="" src="https://i.imgur.com/F0Hkk6P.png" /></p>

<p>This one is going to be pretty straight-forward, clearly we&#39;re meant to exploit whatever is running on this web server. Per the suggestion made by the box maker, I updated my attack machine&#39;s hosts file to add an entry for &quot;kioptrix3.com&quot; and pointed it to the IP address that it got via DHCP.</p>

<p><img alt="" src="https://i.imgur.com/lMSiswJ.png" /></p>

<p>I figured combing over Nikto would be a good idea since this is going to be purely web application hacking, but the only thing I really see here that caught my attention was &quot;phpmyadmin&quot;. It&#39;s an application that streamlines managing instances of MySQL on the server, and is usually a vulnerable attack vector.</p>

<p>I spent maybe an hour or so trying to exploit it, but the exploits I used weren&#39;t working. I tried manual exploitation with a bash script, a Perl script, and Metasploit, but none of them seemed to work. The version number should have been vulnerable, but none of them really did anything, so it was time to poke around the server.</p>

<p>The main page was an uninteresting blog that was advertising a gallery but first I visited the tab that said &quot;Login&quot;.</p>

<p><img alt="" src="https://i.imgur.com/NBUcOAg.png" /></p>

<p>I found this login and tried the usual SQL injection attacks, but nothing worked, so I gave up on that after about 5 minutes to avoid &quot;going down the rabbit hole&quot;.</p>

<p>I decided to check out the gallery, and for the longest time, I didn&#39;t see anything interesting. Finally, I found that if you played around with the &quot;Sorting Options&quot; menu, the URL would change to something that looked like it was SQL injectable:</p>

<p><img alt="" src="https://i.imgur.com/16mUhDa.png" /></p>

<p>Whenver I see a ? followed by some parameter and a number, that always raises an eyebrow. So I deliberately broke the URL to see what would happen:</p>

<p><img alt="" src="https://i.imgur.com/ZQKIDe1.png" /></p>

<p>By adding a single quot ( &#39; ) after the 1, I broke the SQL statement, which resulted in a verbose error message that should never appear. Blind SQL injection is one thing, but &quot;error-based enumeration&quot; is much easier if the server is misconfigured to spit out error messages like this.</p>

<hr />
<h2>Manual SQL Injection:</h2>

<p>So now we need to enumerate the database, but first, we need to see if there are fields on this form that will print out info:</p>

<p><img alt="" src="https://i.imgur.com/e6YScPC.png" /></p>

<p>This is sort of a blind attack, because you have to keep adding numbers to the end of this SQL statement until it starts producing results. This web app wouldn&#39;t display anything if I used less than 6 fields.</p>

<p>Now that I know I can print things to fields 2 and 3, it&#39;s time to see how much information we can get.</p>

<p><img alt="" src="https://i.imgur.com/ZqvEmg4.png" /></p>

<p>By removing the &quot;2&quot; and replacing it with &quot;@@version&quot;, we got the MySQL server to tell us which version it is. This is less about looking for an exploit, and more about just making sure I&#39;m using the correct syntax. This technique also works if you put in &quot;version()&quot;.</p>

<p>I was curious about which user the MySQL server was running as, so I replaced &quot;@@version&quot; with &quot;user()&quot;</p>

<p><img alt="" src="https://i.imgur.com/ogj1oxs.png" /></p>

<p>Again, a fatal misconfiguration. Never ever ever ever run your SQL server as root!</p>

<p><img alt="" src="https://memegenerator.net/img/instances/61409048/never-ever-ever-ever-ever-ever-ever.jpg" /></p>

<p>At this point I had run out of ideas for what I could inject into the server, so some quick googling was in order, and I turned up a very good cheat-sheet for MySQL:</p>

<p><a href="https://www.perspectiverisk.com/mysql-sql-injection-practical-cheat-sheet/">https://www.perspectiverisk.com/mysql-sql-injection-practical-cheat-sheet/</a></p>

<p>I figured the best place to start was enumerating the tables in the database that was in use, but first I had to get the database name:</p>

<p><img alt="" src="https://i.imgur.com/j2AIicy.png" /></p>

<p>By using &quot;database()&quot;, I instructed the server to tell me which database was currently in use by this web app. Armed with this information, I can proceed to getting the tables:</p>

<p><img alt="" src="https://i.imgur.com/GdyaFQu.png" /></p>

<p>There were many tables of course, but the first table on the list was the most interesting. The next move was to enumerate this table to see if it really did have what I wanted, in the form of enumerating the &quot;columns&quot;:</p>

<p><img alt="" src="https://i.imgur.com/9BiSfP9.png" /></p>

<p>This looks very promising. So now, naturally, I want to dump what&#39;s in these columns.</p>

<p><img alt="" src="https://i.imgur.com/FvnfIjc.png" /></p>

<p>Perfect, we have what looks like hashed passwords, and some usernames. At first glance, I assumed they were MD5 sums, but Kali has a tool to identify hashes.</p>

<p><img alt="" src="https://i.imgur.com/Pb2zrTS.png" /></p>

<p>So, as I thought, these are most likely MD5 hashes, which are pretty easily cracked.</p>

<p>I could always fire up hashcat, but since hash-cracking on a VM can be slow, and I didn&#39;t want to copy the hashes to a physical machine with a GPU, I looked up a simple online MD5 cracker. It&#39;s worth trying if you don&#39;t want to sit around waiting for simple hashes to be cracked.</p>

<p><img alt="" src="https://i.imgur.com/npr1hu6.png" /></p>

<p>So the cleartext password for user &quot;loneferret&quot; is &quot;starwars&quot;. Something that probably could have been guessed.</p>

<p>I tried this password on the &quot;login&quot; page on the main site, as well as phpMyadmin, but they didn&#39;t work. I was confused for a moment but then remembered that SSH was open. Sure enough, this password logged me into the server as &quot;loneferret&quot;.</p>

<hr />
<h2>Privilege Escalation:</h2>

<p>I landed in loneferret&#39;s home directory, and did an immediate &quot;ls&quot; to see what we had to work with:</p>

<p><img alt="" src="https://i.imgur.com/NP92GXA.png" /></p>

<p>I looked at the checksec.sh, but it was very long and complex. Didn&#39;t look like something that the maker of the box wrote, it was something aquired online. I didn&#39;t really know what to do with it, so I read the README file:</p>

<p><img alt="" src="https://i.imgur.com/wIfLWzb.png" /></p>

<p>So we&#39;re basically being told exactly how we&#39;re going to get root here. This user has the ability to use &quot;ht&quot; as root, whatever that is. The program that opened up when using this command was very unfamiliar to me, so I had to look it up a bit. Ht is just a file/text editor that has a slight &quot;gui&quot; feel to it, despite being a terminal application. That being the case, I knew that we could edit any file owned by root. So I decided to go the route of editing the &quot;sudoers&quot; file:</p>

<p><img alt="" src="https://i.imgur.com/jORPoWY.png" /></p>

<p>It took a while to figure out but in order to invoke the &quot;File&quot; option, I had to hold Alt and press F.</p>

<p><img alt="" src="https://i.imgur.com/eQq2ShY.png" /></p>

<p>I modified the sudoers file to give loneferret the ability to run /bin/sh as root. I hit Alt+F again, clicked &quot;Save&quot;, closed out ht, and proceeded to run &quot;sudo /bin/sh -i&quot;</p>

<p><img alt="" src="https://i.imgur.com/dxhwQCl.png" /></p>

<p>Easy root. A brief search of root&#39;s directory revealed the &quot;flag&quot;:</p>

<p><img alt="" src="https://i.imgur.com/5qORufF.png" /></p>

<hr />
<h2>Bonus: Automated SQL Injection</h2>

<p>All of the work we did compromising those passwords can be done all with a single tool, &quot;SQLMap&quot;. SQLMap is a tool that can automate enumerating, and then exploiting, web apps vulnerable to SQLi.</p>

<p><img alt="" src="https://i.imgur.com/6cGLLnX.png" /></p>

<p>To get started, you invoke it by name, and provide the URL with the &quot;-u&quot; option. To specify which syntax to use, we provide the &quot;--dbms=mysql&quot; parameter. With the &quot;--dump&quot; option, we tell it to start dumping EVERYTHING it finds, which can be very very verbose. The &quot;--threads&quot; option tells it how many threads to open up to the web app. The more threads, the faster we get results, but having too many threads could overload the target.</p>

<p><img alt="" src="https://i.imgur.com/FeBEntd.png" /></p>

<p>After a while of churning through I saw the above. Looks familiar right? Well looks like SQLMap will even crack the hashes for us if we want it to. So I say &quot;Y&quot;</p>

<p><img alt="" src="https://i.imgur.com/pQDMtGr.png" /></p>

<p>And all that work we did manually was completed in about 60 seconds.</p>

<hr />
<h2>Summary:</h2>

<p>Kioptrix 3 was challenging enough to give me a headache and make me pace around several times throughout the night, but it wasn&#39;t so hard that it was discouraging. It was a nice chance to practice SQL injection, which I prefer to do anyway. SQLMap makes everything quick and easy, but I&#39;m currently studying for the OSCP exam, and SQLMap is forbidden on the exam.</p>

<p>The priv esc was very obvious and straight-forward, and taught the lesson that you have to be very careful about what programs you allow a user to run as root with sudo.</p>

<p>Kioptrix 4 soon!</p>