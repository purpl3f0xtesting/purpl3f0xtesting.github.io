---
published: true
title: Bypassing Defender on modern Windows 10 systems
tags:
  - Pentesting
  - Antivirus Evasion
---

-----
# Intro
-----  

PEN-300 taught me a lot about modern antivirus evasion techniques. It was probably one of th emore fun parts of the course, because we did a lot of cool things in C# and learned to bypass modern-day AV. While the information provided was solid, I found that some of the things taught did not bypass Windows Defender. In this write-up, I will show you how I combined several techniques that I learned, along with some of MSFvenom's own features, to finally get a working Meterpreter shell on a Windows 10 VM in my home lab.  

-----
# Kicking the tires
-----  

Just to make sure we have to actually bypass Defender, let's just play around for a bit and see how quickly it will catch us trying to run Meterpreter.  
The first thing I want to do is refresh my memory, and check my Kali VM's IP address:
<center><img src="/assets/images/av/1.png" />  
  
  <i><small>Figure 1 - Attacker IP</small></i></center>  

Next, let's make the default, non-encoded Meterpreter payload. Since we're making a stand-alone executable, we will not have to worry ourselves about "Bad characters" like we do with exploit development:  
<center><img src="/assets/images/av/2.png" />  
  
  <i><small>Figure 2 - Creating the first payload</small></i></center>

Before we do anything, we need to make a change on the victim machine. We don't want Microsoft collecting samples of what we're doing, because it could mean that in the future, our techniques will become null and void after Microsoft updates Defender's detection abilities:  
<center><img src="/assets/images/av/3.png" />  
  
  <i><small>Figure 3 - Configuring the victim</small></i></center>
