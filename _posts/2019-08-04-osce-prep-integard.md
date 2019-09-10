---
published: true
title: OSCE Prep - Integard Exploit
tags:
  - Exploit Dev
  - Security Research
---
<hr />
<hr />
<p>So with all my lab exercises done it&#39;s time to venture outside the course material and do some extra practice. I went to Exploit-DB and looked up some Windows x86 buffer overflow posts that have links to the vulnerable software. The first thing I tried had a complicated setup just to get the application working so I abandoned it and moved onto something called Integard. From what I understand, it looks like it&#39;s an outbound proxy that&#39;s meant to stop devices from visiting undesirable websites. It runs an HTTP server locally and remotely, and has a login page to administrate it. The vulnerability lies there, in the password field, which doesn&#39;t limit how much input can be fed into it, and will crash the server. It&#39;s a pretty straight-forward buffer overflow, don&#39;t have to get too fancy, but then again, all the tricks necessary to topple this server were things I learned and practiced in the OSCE labs, so maybe to me it didn&#39;t feel too complex. I did however, learn not to go in too cocky and be lazy with the early steps. I wasted 2-3 hours on non-functional payloads all because I missed a single bad character. Had it not been for that, I&#39;d have completed this before lunch, but instead, I was working on this until past 6PM. I ended up finding TWO ways to exploit it, which felt pretty good since the PoC on Exploit-DB was an SEH overwrite, and I couldnt&#39; find any POCs for EIP overwrite, which is what I got when fuzzing. So let&#39;s get into the technical details~</p>

<hr />
<hr />
<h1>Part 1 - Fuzzing with Boo-Gen and Boo-Fuzz</h1>

<hr />
<hr />
<p>Since this is an HTTP attack, I started out similarly to how I started on the HP NNM fuzzing from the OSCE labs. I started by capturing an HTTP POST request in Burp Suite and then copying it to a file:</p>

<p><img alt="" src="https://i.imgur.com/3Snvyhr.png" style="height:243px; width:751px" /></p>

<p>Then using Boo-Gen, I used this file to create the fuzzer:</p>

<p><a href="https://i.imgur.com/aTBt66k.png" target="_blank"><img alt="" src="https://i.imgur.com/aTBt66k.png" /></a></p>

<p>As indicated by the arrow, I&#39;m only fuzzing the password field. I fire this up and let it run its course:</p>

<p><a href="https://i.imgur.com/PJ2tyJ7.png" target="_blank"><img alt="" src="https://i.imgur.com/PJ2tyJ7.png" style="height:1045px; width:1916px" /></a></p>

<p>It took about 15 seconds to get a crash that sort of appeared to be an EIP overwrite. I checked the SE Handler window in Immunity and did not have an SEH overwrite, so I ran with this and decided to see if it really was an EIP overwrite. According to Boofuzz, this crash resulted after sending about 2500 bytes:</p>

<p><img alt="" src="https://i.imgur.com/lauhC1T.png" /></p>

<p><img alt="" src="https://i.imgur.com/SQcxR34.png" /></p>

<p>So, the first lesson I learned at this stage; Always adhere to the proper HTTP format. I tried this for half an hour and didn&#39;t get a crash. I looked back at Boofuzz and noticed that &quot;Content-Length&quot; ended with two sets of \r\n, but my fuzzer only had one. After adding that in, I replicated the crash:</p>

<p><img alt="" src="https://i.imgur.com/OzXYVlT.png" /></p>

<p>I neglected to catch it in this screenshot, but EIP was overwritten as &quot;41414141&quot;, indicating that this mass of A&#39;s had overtaken EIP. So I went to the usual next step.</p>

<hr />
<hr />
<h1>Part 2: Taking control of EIP</h1>

<hr />
<hr />
<p><a href="https://i.imgur.com/vObyHpO.png" target="_blank"><img alt="" src="https://i.imgur.com/vObyHpO.png" /></a></p>

<p>Time to look for the offset overwritting EIP with Pattern_Create and Pattern_Offset.</p>

<p><img alt="" src="https://i.imgur.com/Afc9mxg.png" /></p>

<p>EIP was overwritten by this string, which came out to an offset of 832 bytes:</p>

<p><img alt="" src="https://i.imgur.com/Us1vIFY.png" /></p>

<p>I modified the exploit to only send 832 A&#39;s, 4 B&#39;s, and then additional padding with C&#39;s, and ran the exploit again:</p>

<p><img alt="" src="https://i.imgur.com/fx2jmHj.png" /></p>

<p>EIP was accurately overwritten with the four B&#39;s, and ESP is pointing to the buffer of C&#39;s. So far it&#39;s looking like a routine EIP overwrite.<br />
For the JMP ESP I went looking within integard.dll, and found my jump:</p>

<p><img alt="" src="https://i.imgur.com/RUJwzCE.png" /></p>

<p>I tested this, and reliably landed within the buffer of C&#39;s, so I moved to the final step (or so I thought), and added a reverse shell payload. At this point, the main buffer looked like this:</p>

<p><img alt="" src="https://i.imgur.com/mirOOzy.png" /></p>

<p>It&#39;s at this point that I felt the pain of not looking for bad characters. Everything I tried failed. I wasted hours trying to figure it out. Even after finding MOST of the bad characters, I found that reverse and bind shells didn&#39;t work. Both would succeed initially; reverse shells connected back to my Kali VM, and bind shells would open and listen on ports. But the moment that either shell tried to spawn the cmd.exe process, it would die. So then I tried Meterpreter, but that kept crashing the server in strange ways.</p>

<p>I ran a &quot;sanity check&quot; by loading up the Metasploit module for Integard and tried it out with reverse and bind shells and got the same problem, but, Meterpreter would work. At this point I was thinking that for whatever reason, EIP overwrite was simply unexploitable. It was then that I abandoned this and started over with an SEH overwrite. It was during the redo that I found a bad character I had initially missed, and with that knowledge, I circled back to the EIP exploit and got a working Meterpreter shell:</p>

<p><img alt="" src="https://i.imgur.com/uk11FUh.png" /></p>

<p>Here is a link to the <a href="https://github.com/purpl3-f0x/OSCE-prep/blob/master/eip_integard.py" target="_blank">finalized exploit.</a></p>

<hr />
<hr />
<h1>Part 3: SEH overwrite</h1>

<hr />
<hr />
<p>Looking at the original exploit for MSF, I saw that it was an SEH overwrite. I figured that the reason I got an EIP overwrite is because the fuzzer was starting out with smaller payloads and getting the EIP crash first. Taking a blind stab at it, I doubled my 2500 A buffer to 5000 and tried it again:</p>

<p><img alt="" src="https://i.imgur.com/G0awNXW.png" /></p>

<p>This time I got the SEH overwrite. It&#39;s a little misleading, because EIP is still being overwritten as well, so I had to just ignore that and check the SE Handler in Immunity.</p>

<p><img alt="" src="https://i.imgur.com/kSxpdqO.png" /></p>

<p>Quickly moving along, I used the same Pattern_Create tool to find where the offsets for SEH and nSEH are at:</p>

<p><img alt="" src="https://i.imgur.com/HUgpuoH.png" /></p>

<p>Looks good. SEH and nSEH should be 4 bytes apart. As usual I tested for accuracy...</p>

<p><img alt="" src="https://i.imgur.com/RaCJvFT.png" /></p>

<p><img alt="" src="https://i.imgur.com/GzjCqH6.png" /></p>

<p>...and we have complete accuracy.</p>

<p>The next part is where things would normally get pretty complex, but I dealt with this exact situation in the OSCE labs.</p>

<p>I started out trying a POP POP RET from ntdll, at this address:</p>

<p><img alt="" src="https://i.imgur.com/nMl3CW7.png" /></p>

<p>But this didn&#39;t work. I kept getting &quot;Debugged program was unable to pass the exception&quot;. For a while I was getting very confused, and wasted another half hour here running in circles. It was then that I remembered to run the command &quot;!mona modules&quot;, and found that ntdll has SafeSEH enabled. I haven&#39;t had to deal with SafeSEH yet, but now I understand exactly what it&#39;s meant to do. Fortunately, integard.exe and integard.dll do NOT have SafeSEH enabled, so I can use those. There are just two problems, one I can deal with thanks to the OSCE labs, one I cannot.</p>

<p>All of the address spaces in integard.dll start with &quot;1000&quot;. There&#39;s a null byte right in the middle of the address. No-go.</p>

<p>All of the address spaces in integard.exe start with a null byte. That I can deal with. The trick is to drop the trailing padding of C&#39;s and just overwrite the first 3 bytes of SEH and let the program fill in the last 00 itself. As an experiment, I tried actually putting in the full address anyway, since there was nothing to truncate anymore, and it actually worked out.</p>

<p>Now I know I need to make some backwards jumps to leverage the huge buffer of A&#39;s to hit some shellcode. Luckily, I was able to simply recycle some code from the OSCE lab exercises:</p>

<p><img alt="" src="https://i.imgur.com/ysrwEd2.png" /></p>

<p>This looks a bit jumbled up, so I&#39;ll explain how it works in a moment. For now, I just run this and make sure I reliably hit the &quot;bigBackJump&quot; and that it takes me back enough to hit a payload. Since it works as expected, I add in the meterpreter payload and some NOPs, so my backJump doesn&#39;t land in shellcode:</p>

<p><img alt="" src="https://i.imgur.com/8QfaUI2.png" /></p>

<p>I run this and:</p>

<p><img alt="" src="https://i.imgur.com/dFhJySe.png" /></p>

<p>Another functional Meterpreter shell! Integard has been thoroughly busted~</p>

<hr />
<hr />
<h1>Part 4: Explaining how I do backwards jumps</h1>

<hr />
<hr />
<p>This is a technique I learned doing the GMON command overflow in Vulnserver, and I&#39;ve reused it a couple of times and it works out pretty well. I&#39;ll break down how it all works here:</p>

<p><img alt="" src="https://i.imgur.com/iINSVIa.png" /></p>

<p>Firstly, as expected, I overwrote SEH with a POP POP RET to get the program to jump back 4 bytes and hit my nSEH overwrite, &quot;\xEB\xD0\x90\x90&quot;, which in assembly is JMP SHORT -48 bytes, and two NOPs for padding.</p>

<p><img alt="" src="https://i.imgur.com/qwCAEzJ.png" /></p>

<p>When this jumps back, I hit that 50-byte NOP sled that I placed just under the shellcode, and slide on down to the variable called &quot;jumpCall&quot;, which is just a forward jump using &quot;\xEB\x09&quot;. This is so that I will jump to the CALL instruction inside of &quot;bigBackJump&quot;. Here is where things get interesting.</p>

<p><img alt="" src="https://i.imgur.com/5XIBu8Z.png" /></p>

<p>As you can see here, the hex that makes up &quot;bigBackJump&quot; translates to POP ECX, DEC CH, DEC CH, DEC CH, JMP ECX, CALL -0D. There is a reason I&#39;m jumping over everything and going right into the CALL.</p>

<p><a href="https://i.imgur.com/3sULHPL.png" target="_blank"><img alt="" src="https://i.imgur.com/3sULHPL.png" /></a></p>

<p>As pictured above, when I execute the CALL, execution flow redirects to the POP ECX. At the same time, the CALL placed a return address onto the stack.</p>

<p><a href="https://i.imgur.com/8fcxmX1.png" target="_blank"><img alt="" src="https://i.imgur.com/8fcxmX1.png" /></a></p>

<p>That return address is then POP&#39;d int ECX.</p>

<p><a href="https://i.imgur.com/cpxaXnD.png" target="_blank"><img alt="" src="https://i.imgur.com/cpxaXnD.png" style="height:285px; width:1244px" /></a></p>

<p>The higher-end of the CX register, CH, is decremented 3 times. Each decrement of CH lowers the overall value of ECX by 256, totalling up to 768. This is explained in more depth in my <a href="https://purpl3f0xsec.tech/blog/single_post.php?post-slug=osce-prep---seh-bof-w-o-egghunter" target="_blank">GMON SEH post.</a></p>

<p><img alt="" src="https://i.imgur.com/BvNYoPt.png" /></p>

<p>JMP&#39;ing to the value now in ECX lands me in a huge NOP sled. I should probably tweak the function to jump only 512 bytes next time..</p>

<p><img alt="" src="https://i.imgur.com/u44xrX8.png" /></p>

<p>Eventually the NOP sled ends up hitting the start of the Meterpreter decoder, and from there of course, we get a shell.<br />
The finalized exploit is <a href="https://github.com/purpl3-f0x/OSCE-prep/blob/master/integard.py" target="_blank">here.</a></p>

<hr />
<hr />
<h1>Conclusion</h1>

<hr />
<hr />
<p>I spent my entire Saturday on this, all because I fell into the rabbit hole of once again over-thinking when things don&#39;t work right. I should have been more diligent in my bad character checks, and paid more attention to which modules have SafeSEH enabled. Other than that, this exercise was a pretty simple one. The EIP overwrite is pretty vanilla and straight forward, and the SEH isn&#39;t too bad if you can work with backwards jumps. It was defintely interesting to work with a target that was vulnerable to both types of attacks.</p>

<hr />
<hr />
<h1>Links</h1>

<hr />
<hr />
<p>Boo-Gen HTTP fuzzer generator - <a href="https://github.com/h0mbre/CTP/tree/master/Boo-Gen" target="_blank">https://github.com/h0mbre/CTP/tree/master/Boo-Gen</a></p>

<p>Boofuzz Python fuzzer - <a href="https://boofuzz.readthedocs.io/en/latest/" target="_blank">https://boofuzz.readthedocs.io/en/latest/</a></p>

<p>Integard vulnerability disclosure - <a href="https://www.coresecurity.com/content/integard-home-and-pro-remote-buffer-overflow-exploit" target="_blank">https://www.coresecurity.com/content/integard-home-and-pro-remote-buffer-overflow-exploit</a></p>

<p>Exploit-DB Metasploit Module - <a href="https://www.exploit-db.com/exploits/14941" target="_blank">https://www.exploit-db.com/exploits/14941</a></p>

<hr />
<hr />
<p>&nbsp;</p>