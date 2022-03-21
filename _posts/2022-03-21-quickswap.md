---
published: true
title: "Real-world Android Malware Analysis 2: de.app.quickcurrencyswap"
tags:
  - Malware Analysis
  - Reverse Engineering
  - Android
---

---
# Intro
---

Lately there has been an explosion of popularity in crypto currency phishing apps on the Google Play Store, trying to capitalize on the ever increasing craze behind crypto currency and NF/Ts. Some are more obvious than others. Some are more clever than others. The app I'm going to analyze today is a very simple app that makes no attempts at all to hide its malicious behavior from a code or dynamic analysis standpoint. Therefore, this post will be fairly short, but hopefully you'll learn something from it.

---
# Static analysis with Jadx-gui
---

I installed the app on a Pixel 5a that I bought for security research that is not linked to any real accounts or used for anything sensitive like email or banking. In order to get the APK from the device, I ran the command `adb shell pm list packages` to find the app's package name.
<center><img src="/assets/images/quickswap/1.png" /></center>  
<center><i><small>Figure 1 - Find the package name.</small></i></center>  
<br/> 

Next, I needed the app's path:
<center><img src="/assets/images/quickswap/2.png" /></center>  
<center><i><small>Figure 2 - Find the path to the APK.</small></i></center>  
<br/> 

This app is a `split APK`, and is meant to support multiple languages and Android CPU architectures. The one I want is the `base.apk`.

This APK is opened in Jadx-gui, and one of the first things I check as always is the `AndroidManifest.xml`:
<center><img src="/assets/images/quickswap/3.png" /></center>  
<center><i><small>Figure 3 - Examining the manifest.</small></i></center>  
<br/> 

Most of the code in the `MainActivity` class isn't very interesting, but at the very bottom, there is an interesting method that invokes another class:
<center><img src="/assets/images/quickswap/4.png" /></center>  
<center><i><small>Figure 4 - Main activity invokes new code.</small></i></center>  
<br/> 

To follow this code, I right-click the `WebViewActivity` and click "Go To Declaration" to open this new class. There isn't very much code in this class, so finding the interesting bit is very easy:
<center><img src="/assets/images/quickswap/5.png" /></center>  
<center><i><small>Figure 5 - Loading a suspicious web site.</small></i></center>  
<br/> 

Now it's usually at this point where I'll stop static analysis and pivot to OSINT and dynamic analysis in order to determine if the URL is legitimate or not.

---
# OSINT
---

Rather than directly visiting the URL right away, I always submit suspicious links to `urlscan.io` and let this online tool scan the site and show me what it is:
<center><img src="/assets/images/quickswap/6.png" /></center>  
<center><i><small>Figure 6 - Scanning the suspicious URL.</small></i></center>  
<br/> 

So here we can see a pretty shoddy-looking attempt at impersonating QuickSwap, which is a legitimate crypto currency-trading platform. The legitimate site is at `https://quickswap.exchange` and looks like this:
<center><img src="/assets/images/quickswap/7.png" /></center>  
<center><i><small>Figure 7 - The real QuickSwap.</small></i></center>  
<br/>

Now that I'm confident that the app is loading an illegitimate website, it's time for the final nail in the coffin; catching the app in the act of phishing.

---
# Dynamic Analysis with Burp Suite
---

The Pixel 5a is equiped with a Burp Suite certificate allowing me to proxy all traffic from the device through Burp, as long as it is over HTTP/S. I will also being using adb to take screenshots of the phone's screen using `adb shell screencap -p > ss.png`.

The app is launched, and sure enough, Burp captures the request to the C2 server discovered in code earlier, and the app immediately opens the website we saw earlier in `urlscan.io`:
<center><img src="/assets/images/quickswap/8.png" /></center>  
<center><i><small>Figure 8 - Traffic in Burp Suite.</small></i></center>  
<br/>

<center><img src="/assets/images/quickswap/9.png" /></center>  
<center><i><small>Figure 9 - Phishing site loaded on phone.</small></i></center>  
<br/>

Tapping "Continue" presents a box meant to type in the "Recovery Phrase" aka "mnemonic" in order to connect to the wallet. I type in a `very professional` fake mnemonic, knowing full well that the app's dev will see this:
<center><img src="/assets/images/quickswap/10.png" /></center>  
<center><i><small>Figure 10 - Mnemonic theft.</small></i></center>  
<br/>

After tapping "Connect Wallet" the app presents a fake error message in a lousy attempt at avoiding suspicion, but really it just makes the app look even more suspicious.
<center><img src="/assets/images/quickswap/11.png" /></center>  
<center><i><small>Figure 11 - False error message.</small></i></center>  
<br/>

However, Burp tells us all we need to know. We already knew it was coming, but this will serve as the "smoking gun" evidence:
<center><img src="/assets/images/quickswap/12.png" /></center>  
<center><i><small>Figure 12 - Mnemonic uploaded to C2.</small></i></center>  
<br/>

With the combined evidence that was obtained through static analysis, dynamic analysis, and OSINT, I can confidently say that this app is `True Positive` for `phishing`.

## What now?

At the time of writing this, I have already reported the app to Google for takedown from the Play Store, and I have reported the website to its hosting provider for takedown, as well as filled a report with Microsoft and Google Safe Browsing.

If you aren't already aware, there is a phenominal website out there to assist with and streamline the process of reporting phishing sites called `https://phish.report/`. Using this site, you can report malicious URLs directly to their hosting providers, as well as report them to Microsoft and Google, and keep track of the progress in marking the site malicious:
<center><img src="/assets/images/quickswap/13.png" /></center>  
<center><i><small>Figure 13 - Phish.Report</small></i></center>  
<br/>

I've been using them to help report and take down any phishing domains I come across, be it a site loaded by a malicious android app, or Discord Nitro phishing. It's a nice thing to have around if you like fucking over phishers, as I do~

---
# Conclusion
---

This app was very easy to reverse engineer, and to be honest, most crypto wallet phishing apps I see aren't much more complex. Most of them just load a website with webView and it's the site itself that does all the phishing activity. There are lots of different types of phishing apps out there, designed to steal social media accounts, bank accounts, and more, and those apps are usually more difficult to analyze because they utilize different techniques such as cookie stealing, or abusing the Android Accessibility Service and SMS receivers to read keystrokes and retrieve OTP codes from text messages.

I hope that with the information I've shared here, I've helped educate you on how you can detect phishing attempts by suspicious-looking crypto currency wallet apps. Sometimes it's as simple as visiting the legitimate wallet's website to see if they even have an app at all, and if they do, they should have a link to the real thing. ALWAYS be wary of any app that says it needs your mnemonic phrase or recovery key.

Stay safe, keep your coins, and fuck phishers~

---
### References
---
[Android Debug Bridge(adb)](https://developer.android.com/studio/command-line/adb) 
[Jadx Java Decompiler](https://github.com/skylot/jadx)  
[Phish.Report](https://phish.report/)  
[Burp Suite](https://portswigger.net/burp)  
[urlscan.io](https://urlscan.io/)  
