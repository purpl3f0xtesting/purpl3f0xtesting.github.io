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
