---
published: false
---

---
Title: Vulnserver TRUN - Vanilla EIP overwrite
Published: True
Tags: ["Exploit Dev", "Security Research"]
layout: post
---

<h2>Stack buffer overflow exercise: Vulnserver.exe</h2>

<p>I&#39;ve taken quite a liking to doing basic stack buffer overflow attacks after learning out to do them in the Pentesting With Kali Linux course. I learned so much about assembly, and how to debug and analyze programs and gain a deeper understanding of how the program execution flows. This has led me to wanting to do a write-up of how I go through the process.</p>

<hr />
<p><a href="https://github.com/stephenbradshaw/vulnserver">Vulnserver.exe, found here,</a> is deliberately-vulnerable windows executable that is meant purely for practicing various buffer-overflow attacks. The program has a long list of commands it can execute, and several of them are vulnerable to different types of buffer overflow. After cheating a little bit by looking at some other write-ups, I decided to attack the TRUN command, as it is vulnerable to a very simple, straight-forward stack buffer overflow attack, which is the only type of attack I know how to pull off.</p>

<hr />
<p>Let&#39;s start by checking what the program can do. I have it running on my trusty Windows XP 32-bit VM, which is bridged to my physical network, along with my Kali VM. I start by connecting with netcat:</p>

<p><img alt="" src="https://i.imgur.com/gggnKK9.png" /></p>

<p>The commands seem to be case-sensitive, so you have to use all-caps with them. I haven&#39;t done much fuzzing but from what I can tell, each command expects different input. They don&#39;t appear to do much from what I can tell, but that may not be the case. Anyway, let&#39;s start fuzzing the TRUN command.</p>

<p><img alt="" src="https://i.imgur.com/dA713si.png" /></p>

<p>Pictured above is my python code to do some initial fuzzing. It creates a buffer of 100 A&#39;s and sends it to the server, then as it goes through the &quot;for&quot; loop, it increments by 200, and then sends 300 A&#39;s, and so on. The IP address and port number are hard-coded for ease of use during this demonstration but it is possible to write the code in a way that it accepts command-line arguments.</p>

<p><img alt="" src="https://i.imgur.com/LR3GrVw.png" /></p>

<p>The fuzzer is executed and this is what we see. After sending 2300 bytes, the program stops responding. Now, it&#39;s very likely that the program actually crashed after getting slammed with 2100 bytes, because I neglected to add &quot;time.sleep(1)&quot; to my code, so this loop executes extremely fast.</p>

<p><img alt="" src="https://i.imgur.com/QJWHKF4.png" /></p>

<p>I&#39;m using Immunity Debugger now to analyze the crash. Pictured above is what the debugger displays when it has crashed due to an &quot;access violation&quot;, which is an error that indicates that this program attempted to access a memory location that either does not exist, or is owned by another process. This is because I overwrote the &quot;Instruction Pointer&quot;, which is displayed as &quot;EIP&quot; in the debugger. This is a CPU register that stores a 4-byte value that points to the memory address of the next instruction.</p>

<p><img alt="" src="https://i.imgur.com/GkKLx3d.png" /></p>

<p>Sure enough, EIP reads &quot;41414141&quot;, which is hexidecimal for &quot;AAAA&quot;. So far so good; this is what we want, to overwrite EIP. If you can overwrite EIP, which points to the next instruction, then you can control execution flow. But right now we don&#39;t know the exact number of bytes it takes to reach this point, so we need to figure that out before we move on.</p>

<p>There is a tool already in Kali Linux to help with this. It will generate a unique string that can help us determine how many bytes it took to overwrite EIP. The tool is located at:<br />
/usr/share/metasploit-framework/tools/exploit/pattern_create.rb</p>

<p>We run this Ruby script, with the parameter &quot; -l 2100&quot; which tells it that we want a string that&#39;s 2100 bytes long. I forgot to screencap this so I&#39;ll just show what this looked like after I put it in my fuzzer:</p>

<p><img alt="" src="https://i.imgur.com/QoTOYfc.png" style="height:289px; width:1211px" /></p>

<p>Don&#39;t forget to modify the connection part of your code; you don&#39;t want this looping because it only needs to run once.</p>

<p><img alt="" src="https://i.imgur.com/rXBHh9u.png" /></p>

<p><img alt="" src="https://i.imgur.com/QDQVqCD.png" /></p>

<p>EIP was over-writen again, but this time by &quot;386F4337&quot;.</p>

<p>Now you must be asking yourself, &quot;How the hell am I supposed to find this in that string? Do I count the bytes??&quot; Luckily no, Kali rescues us again:</p>

<p><img alt="" src="https://i.imgur.com/T1s0Fgw.png" /></p>

<p>Just run the companion Ruby script to &quot;pattern_create&quot;, and pass it the parameter &quot;-q 386F4337&quot; and it tells you exactly how many bytes it took to reach this point. For us, it was 2003 bytes. But I want to test this to be absolutely sure, so I&#39;m going to change the fuzzer again:</p>

<p><img alt="" src="https://i.imgur.com/YKCh6vN.png" /></p>

<p>Here I am sending 2003 &quot;A&#39;s&quot;, 4 &quot;B&#39;s&quot;, and just to &quot;pad&quot; it, 500 &quot;C&#39;s&quot;. The theory is that, if it takes exactly 2003 bytes to overwrite EIP, then I should be able to precisely overwrite EIP with the 4 B&#39;s here.</p>

<p><img alt="" src="https://i.imgur.com/o00pd7f.png" /></p>

<p>I run the fuzzer again after restarting the program in the debugger (Ctrl+F2 in Immunity Debugger), and EIP is now &quot;42424242&quot;, which is hexidecimal for &quot;BBBB&quot;. Perfect. We know we can accurately overwrite EIP now, which means we can now control execution flow with the right code. So....what do we put in EIP exactly? Well we need it to run our shellcode. Well, where is that?</p>

<p><img alt="" src="https://i.imgur.com/BKd2uDb.png" /></p>

<p>Say hello to the ESP register, aka, the &quot;Stack Pointer.&quot; This CPU register points to the current location in the stack that is being executed. Remember that 500 C&#39;s we sent? See how ESP seems to be pointing to them?</p>

<p><img alt="" src="https://i.imgur.com/ECrzFfZ.png" /></p>

<p>If we right-click the ESP and click &quot;Follow in Dump&quot;, it will show us the memory dump of the system&#39;s RAM. I didn&#39;t screencap it, but trust me when I say that it&#39;s just showing a whoooole lot of C&#39;s. That means that whatever we put in place of the C&#39;s will get executed, <em>IF </em>we can make the program execute what&#39;s at the ESP. But we don&#39;t always know what address that is; it changes everytime the program is run. But fear not, there is a way to accurately point the EIP in a way that will execute whatever is at ESP, no matter what the address is!</p>

<p><img alt="" src="https://i.chzbgr.com/full/8335254016/h5B348BA5/" /></p>

<p>Actually we&#39;re not taking a selfie, but there <em>IS </em>something we need to do first before we worry about EIP, because this next step is crucial to perform first. Failure to do so could cause several future steps to fail.</p>

<p>We need to identify the &quot;bad characters&quot;.</p>

<p><img alt="" src="https://litreactor.com/sites/default/files/imagecache/header/images/column/headers/breaking-bad-dark-side.jpg" /></p>

<p>Yea th- wait no! Not <em>those </em>bad characters.</p>

<p>I&#39;m talking about hexidecimal values that the program could potentially interpret as instructions, which would <em>truncate, </em>or in other words, prematurely terminate our input. This could cause problems for us when deploying the shellcode, or trying to input a certain hex value to overwrite EIP with. So we need to send EVERY possible hex combination from x00 to xFF to the server to see what happens.</p>

<p><img alt="" src="https://i.imgur.com/1W7aDTF.png" style="height:582px; width:1245px" /></p>

<p>I leave off x00 when I do this, because x00 is NULL, which will always be truncated. So I found somewhere to copy-paste this to avoid typing it out by hand, and then sent this buffer to the target.</p>

<p><img alt="" src="https://i.imgur.com/osbaEOK.png" /></p>

<p>So now we have to manually comb through and follow the memory dump to see if any of our characters got left off. You can find this after right-clicking the ESP and clicking &quot;Follow in Dump&quot; as shown above. What&#39;s very nice about this particular vulnerability is that there are no bad characters aside from x00, which is nice. In other instances of buffer overflow practice, I&#39;ve found characters that completely truncated the input, so I had to remove that one hex character from my python fuzzer, and then re-launch the attack to see if anything else causes truncation. You have to repeat this process until nothing else is truncated or dropped.</p>

<p>Now that we have taken care of this, back to figuring out what we want to put in EIP. Recall that we want to get the program to <strong><em>jump </em></strong>to <em><strong>ESP, </strong></em>and there&#39;s a way to do that. In assembly, there is function that will tell the program to jump to the address stored in ESP and resume execution. That function is &quot;JMP ESP&quot;. But we can&#39;t just search for this in the debugger, we won&#39;t find it. We need the hex first.</p>

<p>You could always google it, but once again Kali makes it easy for us:</p>

<p><img alt="" src="https://i.imgur.com/6WBqdTZ.png" /></p>

<p>The nasm_shell.rb script is the one we want. It will take assemlby commands we give it and return the hex value.</p>

<p><img alt="" src="https://i.imgur.com/KV2oTL1.png" /></p>

<p>FFE4 is what we want, with FF corresponding to JMP and E4 corresponding to ESP. Now we can search for it in Immunity Debugger. But first, we need something.</p>

<p><a href="https://github.com/corelan/mona">We need this python utility called Mona.</a> Download this and copy the mona.py file into Immunity Debugger&#39;s &quot;py commands&quot; folder. After that&#39;s done, we can call up some of its features to help us find a JMP ESP we can use.</p>

<p><img alt="" src="https://i.imgur.com/OfePkH8.png" style="height:518px; width:1221px" /></p>

<p>At the bottome of the debugger window, type &quot;!mona modules&quot;. This will query the program for the individual modules that it has loaded up.</p>

<p>Now this next part is very important. Notice I highlighted &quot;ASLR&quot;. This stands for Address Space Layout Randomization. This is a memory protection measure that randomizes where it stores objects in memory every time the application is run. If ASLR is enabled on a module we <strong><em>can&#39;t use it for our JMP ESP.</em></strong> Luckily, nothing here seems to have it.</p>

<p>The other highlighted box is just to show that dll&#39;s are usually a good target to go after. I decided to go after &quot;essfunc.dll&quot; since it is part of vulnserver.exe</p>

<p><img alt="" src="https://i.imgur.com/Ik6FrMk.png" /></p>

<p>I invoke mona again to search for the hexidecimal values for JMP ESP. Mona turned up 9 results, and none of them have ASLR protection enabled. I keep it simple and decide to use the very first result, &quot;625011AF&quot;</p>

<p><img alt="" src="https://i.imgur.com/Je1UQAZ.png" /></p>

<p>I use &quot;Follow in expression&quot; to verify that this address is pointing to a JMP ESP instruction, which it is. While I&#39;m here, I set a &quot;breakpoint&quot; by pressing F2. A &quot;breakpoint&quot; is a point in a debugger which will halt execution if it is hit. I am doing this to verify that I am successfully redirecting execution to this instruction.</p>

<p><img alt="" src="https://i.imgur.com/lEPXBjb.png" /></p>

<p>Now this looks a little wierd now that it&#39;s plugged into the exploit. This is a 32-bit application running on 32-bit Windows. That means that the system is using &quot;Little Endian&quot;, or in other words, &quot;Least significant byte first.&quot; So we have to add the JMP ESP address&#39;s individual bytes in reverse order as shown above.</p>

<p>I restart the program and send the new payload:</p>

<p><img alt="" src="https://i.imgur.com/SnBRx8Y.png" /></p>

<p>This is displayed at the very bottom of the debugger window. It&#39;s a message to let me know that execution hit my breakpoint and paused.</p>

<p><img alt="" src="https://i.imgur.com/AsaAk10.png" /></p>

<p>EIP has been accurately overwritten with exactly what I wanted.</p>

<p><img alt="" src="https://i.imgur.com/jTPa0b2.png" /></p>

<p>If I press F7, it will &quot;step&quot; into the next instruction. That means the debugger advances by a single instruction and pauses again. We see here that execution immediately jumped to the C&#39;s, as indicated by hex 43. Now we know for sure that we can get our payload to execute if we put it after EIP in our exploit.</p>

<p>Now it&#39;s time for the fun part, the shellcode. This is going to be our actual payload, the malicious code that we want to execute. Writing shellcode from scratch is incredibly difficult, and not something that most people are expected to do, so we&#39;ll just keep using Kali&#39;s built-in tools to do it for us. My payload of choice is usually a <em><strong>reverse shell that connects over 443.</strong></em> A reverse shell means that my code will instruct the victim to make a connection back to me. I do this for two reaons:</p>

<p>1. If the victim has a firewall turned on, we cannot reliably use a payload that simply listens on a port for us to connect to.<br />
2. I use port 443 in the event that the victim is protected by an egress firewall, a firewall that filters out-bound traffic. 443 is HTTPS, web traffic, so as long as the theoretical egress firewall isn&#39;t doing deep-packet inspection, this should work.</p>

<p><img alt="" src="https://i.imgur.com/MDyz0xw.png" /></p>

<p>The payload is in hex and is already formatted in a way that we can copy-paste it right into our exploit. I&#39;ll break downt he command below:</p>

<p>MSFvenom is a utility that is part of the Metasploit-Framework that generates stand-alone payloads that can be deployed independantly of MSFconsole.</p>

<p>The -p flag is our payload.</p>

<p>The -f flag is the format, and I&#39;m formatting it as c code. You can specify python if you want, but c works in python scripts.</p>

<p>The -a flag is for architecture. x86 will format the payload as 32-bit.</p>

<p>The -e flag is for encoding. If our payload cannot have certain &quot;bad characters&quot;, the encoder will format the payload in a way that it will not need to use those bad characters.</p>

<p><img alt="" src="https://i.imgur.com/xRNS4bI.png" /></p>

<p>Another important note; notice the &#39;\x90\ * 20 I put in after the EIP address and before the shellcode. In assembly, x90 is NOP, or NO OPERATION. It does exactly what it sounds like, nothing. It&#39;s padding. Our encoded payload will self-decode once it executes, and we need some padding as the payload unpacks itself. If you don&#39;t add a few NOPs (often called the &quot;NOP sled, since the program &quot;slides&quot; down the NOPS and hits the payload), the exploit could fail.</p>

<p>Do or die time. The exploit is complete. If we did this right, we will get a shell from the victim after executing this.</p>

<p><img alt="" src="https://i.imgur.com/aOjU9Q6.png" /></p>

<p>My breakpoint at the JMP ESP is still set, and I see that after &quot;stepping&quot; with F7 again, I hit my NOP sled.</p>

<p>I let normal execution continue by hitting F9 aaaaaand....!</p>

<p>Nothing. The program crashes and I get no shell. What happened? Weeeelll....</p>

<p><img alt="" src="https://i.imgur.com/R3dfJjv.png" /></p>

<p>You see that there? My payload has a NULL character in it. I got too relaxed when thinking &quot;There are no bad characters&quot; and forgot that x00 is ALWAYS a bad character. Time to re-generate my shellcode...</p>

<p><img alt="" src="https://i.imgur.com/cip5klq.png" style="height:525px; width:1236px" /></p>

<p>Notice the new flag on the end of the command, the -b. This is where you tell MSFvenom what the bad characters are and it will use the selected encoder to make sure the shellcode does NOT have the bad characters.</p>

<p><img alt="" src="https://i.imgur.com/QZbLnmZ.png" /></p>

<p>The revised exploit with the new shellcode. Let&#39;s try again.</p>

<p><img alt="" src="https://i.imgur.com/COIyEIl.png" /></p>

<p>SUCCESS!! THE VICTIM HAS BEEN COMPROMISED!!</p>

<p>I caught the incoming reverse shell with &quot;nc -nvlp 443&quot;, and we have a cmd.exe prompt that is sending commands to the victim. The permissions you get from this depend on which user was running the program. This was running as administratior sooooo...</p>

<p><img alt="" src="https://i.imgur.com/TbUYFfP.png" /></p>

<p>We are the Administrator. The victim has been completely compromised.</p>

<p><img alt="" src="https://i.imgur.com/oQb1nlq.png" /></p>

<p>Just to amuse myself I put a flag on there just so I have something to say &quot;hey look I captured the flag!&quot;</p>

<p>BUT-!</p>

<p>We&#39;re still not done. There&#39;s a problem. If we exit our reverse shell, the victim application crashes. Bad news! We&#39;ve not only tipped off the victim, we&#39;ve lost our foothold! But we can fix that.</p>

<p><img alt="" src="https://i.imgur.com/3YDjq3S.png" /></p>

<p>By adding yet another parameter to the end of the MSFvenom command, EXITFUNC, we can control what the shellcode does when it finishes executing. By specifying &quot;EXITFUNC=thread&quot;, we&#39;re telling the victim program to close a single thread, instead of just crashing and burning. I update my exploit, run it again, and then close my reverse shell:</p>

<p><img alt="" src="https://i.imgur.com/kyYTODX.png" /></p>

<p>Immunity Debugger confirms that the program merely closed a thread, and is still running.</p>

<p><img alt="" src="https://i.imgur.com/80fLSug.png" /></p>

<p>As shown here, I can run my exploit several times in a row without having to restart vulnserver.exe because my shellcode isn&#39;t terminating the application anymore.</p>

<p>Just for shits &#39;n giggles, I decided to add in a little bonus to show how you can use any payload you want.</p>

<p>I made new shellcode and set the payload as &quot;windows/meterpreter/reverse_tcp&quot;, and kept all other options the same. I fired up MSFconsole and prepared to receive the connection:</p>

<p><img alt="" src="https://i.imgur.com/XRIoRIp.png" /></p>

<p>I run the exploit again:</p>

<p><img alt="" src="https://i.imgur.com/vibCjPy.png" style="height:172px; width:1185px" /></p>

<p>Now we have a nice shiney meterpreter session instead of a standard shell. I won&#39;t go into a deep lesson on meterpreter but here are some cool things you can do with it:</p>

<p><img alt="" src="https://i.imgur.com/zPYNVKe.png" /></p>

<p><img alt="" src="https://i.imgur.com/cAvMVtw.png" /></p>

<p><img alt="" src="https://i.imgur.com/GcjUg8O.png" /></p>

<p><img alt="" src="https://i.imgur.com/lJgCzBc.png" /></p>

<hr />
<p>And that, boisengurls, is how you do a standard stack buffer overflow exploit, provided you don&#39;t have to worry about memory protections or more complicated techniques.</p>

<p>The final exploit is here: https://github.com/purpl3-f0x/vulnserver-trun/blob/master/vuln_server_bof.py</p>

<p>If you want to try this yourself, stand up a Windows VM, it really doesn&#39;t matter which, but XP or 7 are the best choices, and try to get 32-bit if you can. <a href="https://www.immunityinc.com/products/debugger/">Immunity debugger</a> is free, and fairly easy to use once you get the hang of it.</p>

