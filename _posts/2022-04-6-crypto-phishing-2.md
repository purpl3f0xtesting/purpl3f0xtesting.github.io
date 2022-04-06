---
published: true
title: "Reversing a simplistic crypto phishing app on the Play Store"
tags:
  - Malware Analysis
  - Reverse Engineering
  - Android
---

---
# Intro
---

Last time we took a look at an Android app that attempted to steal crypto currency. It was fairly easy to analyze, and proving it as malicious didn't take long. Today I want to take at yet another crypto phishing app (they spring up like weeds) that is a little harder to analyze. I learned something cool from it and wanted to share it.

---
## Static Analysis with Jadx-gui
---

Analysis as always starts with static analysis. Looking at the app's entry point, `com.eth.appdroid.SplashActivity`, I can see code that looks like it might be putting a URL together:

<center><img src="/assets/images/eth/1.png" /></center>  
<center><i><small>Figure 1 - Pieces of a URL.</small></i></center>  
<br/> 

I can see that this string is being added to whatever is in the variable `C4556a.f24732b`. I can double-click this in Jadx and get taken to its definition:

<center><img src="/assets/images/eth/2.png" /></center>  
<center><i><small>Figure 2 - The rest of the URL.</small></i></center>  
<br/> 

So this app is doing something we've seen before, it's reaching out to some webpage to retreive its functionality. So let's look at what this website is:

<center><img src="/assets/images/eth/3.png" /></center>  
<center><i><small>Figure 3 - Retreiving a new URL.</small></i></center>  
<br/> 

---
## Static Analysis of a web site with Chromium
---

I'm using a screenshot of Burp here because it's easier to read due to the JSON not word-wrapping in my Chromium.  The app receives a new URL in the JSON body of this response. It's likely that the app will load this site next, so we'll have a look at it, too:

<center><img src="/assets/images/eth/4.png" /></center>  
<center><i><small>Figure 4 - An innocent-looking site.</small></i></center>  
<br/> 

So what we find is just an image and no apparent functionality. Or is there?

<center><img src="/assets/images/eth/5.png" /></center>  
<center><i><small>Figure 5 - Viewing the page source (very illegal if you're a journalist btw).</small></i></center>  
<br/> 

<center><img src="/assets/images/eth/6.png" /></center>  
<center><i><small>Figure 6 - Highly suspicious strings.</small></i></center>  
<br/> 

So it looks like there is a lot more to this site than meets the eye. But why is it like this? Why can't we see it? Well, there is some JavaScript fuckery about:

<center><img src="/assets/images/eth/7.png" /></center>  
<center><i><small>Figure 7 - JavaScript at the very bottom.</small></i></center>  
<br/> 

<center><img src="/assets/images/eth/8.png" /></center>  
<center><i><small>Figure 8 - The script.</small></i></center>  
<br/> 

So basically what's happening is this JavaScript isn't letting us see what's really going on here because of the way the site's backend is configured. It may be that the dev isn't ready to start phishing yet, or may be trying to throw off researchers and analysts.

<center><img src="/assets/images/eth/9.png" /></center>  
<center><i><small>Figure 9 - Firebase returning a "2".</small></i></center>  
<br/> 

So how do we get around this?  

This is what took me a while to figure out. I tried using inspect element to put breakpoints on the JavaScript and try to edit it on the fly, but that wasn't working. Eventually, I decided to just literally do what the JavaScript is doing by running one command in the console:

<center><img src="/assets/images/eth/10.png" /></center>  
<center><i><small>Figure 10 - The real site revealed.</small></i></center>  
<br/> 

<center><img src="/assets/images/eth/11.png" /></center>  
<center><i><small>Figure 11 - After clicking "Get Started".</small></i></center>  
<br/> 

<center><img src="/assets/images/eth/12.png" /></center>  
<center><i><small>Figure 12 - Asking for highly sensitive wallet details.</small></i></center>  
<br/> 

<center><img src="/assets/images/eth/13.png" /></center>  
<center><i><small>Figure 13 - Giving a fake error message after typing in a fake mnemonic.</small></i></center>  
<br/> 

Immediately upon hitting "Continue" after providing a mnemonic, the site makes a request to Firebase again, but the body nor URL appeared to contain the mnemonic. The site may be encrypting the mnemonic before upload, or it may be that since this site doesn't yet apper to be in use, it's simply not uploading to Telegram as expected.

Either way, it's very obvious at this point that the website, and by extension, the app indend to phish victims by pretending to be an Etherium wallet.

---
## Conclusion
---

This phishing site was definitely trickier than the last one we looked at, and it was an interesting lesson in how bad actors might try to hide in plain sight. If you're investigating a suspicious website and it looks blank, or looks like it's just an image, dig deeper, look at any JavaScripts loaded, you never know what you might find.