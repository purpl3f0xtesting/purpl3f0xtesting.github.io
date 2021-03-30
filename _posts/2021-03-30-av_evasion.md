---
published: false
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
<center><img src="/assets/images/av/1.png" /></center>  
<center><i><small>Figure 1 - Attacker IP</small></i></center>  

Next, let's make the default, non-encoded Meterpreter payload. Since we're making a stand-alone executable, we will not have to worry ourselves about "Bad characters" like we do with exploit development. While we're at it, let's put it in the default apache web server directory:  
<center><img src="/assets/images/av/2.png" /></center>  
<center><i><small>Figure 2 - Creating the first payload</small></i></center>

Before we do anything, we need to make a change on the victim machine. We don't want Microsoft collecting samples of what we're doing, because it could mean that in the future, our techniques will become null and void after Microsoft updates Defender's detection abilities:  
<center><img src="/assets/images/av/3.png" /></center>  
<center><i><small>Figure 3 - Configuring the victim</small></i></center>

With that set up, there are two ways we can get the binary on the victim. If we assume that there is RDP access, we can of course just browse to it:  
<center><img src="/assets/images/av/4.png" /></center>  
<center><i><small>Figure 4 - Using web browser to get payload</small></i></center>

This isn't ideal, because Edge is using Windows Defender to scan things as it downloads them, and it gets caught immediately:  
<center><img src="/assets/images/av/5.png" /></center>  
<center><i><small>Figure 5 - Edge detecting malware</small></i></center>

However, we can click the ellipsis and chose to keep this download anyway:  
<center><img src="/assets/images/av/6.png" /></center>  
<center><i><small>Figure 6 - Keeping the binary</small></i></center>  

<center><img src="/assets/images/av/7.png" /></center>  
<center><i><small>Figure 7 - Keeping the binary pt.2</small></i></center>

After this, we'll go ahead and configure the MSFconsole listener to catch anything that may come through:  
<center><img src="/assets/images/av/8.png" /></center>  
<center><i><small>Figure 8 - Preparing to catch Meterpreter</small></i></center>

Predictably, as soon as we double-click the executable, Windows flags and deletes it:  
<center><img src="/assets/images/av/9.png" /></center>  
<center><i><small>Figure 9 - Meterpreter caught by Defender</small></i></center>

<center><img src="/assets/images/av/10.png" /></center>  
<center><i><small>Figure 10 - The alert inside of Security Center</small></i></center>

<center><img src="/assets/images/av/11.png" /></center>  
<center><i><small>Figure 11 - Empty downloads folder after Defender cleans infection</small></i></center>

