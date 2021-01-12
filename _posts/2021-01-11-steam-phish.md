---
published: false
---
-----
# Intro
-----
More and more often I keep hearing about friends who get random private messages from their friends asking them to click a link and "Vote for my team in this competition" or something along those lines. The messages come from compromised accounts that are being used to take over even more accounts. I am not sure what the goal is, maybe the attacker is getting away with payment info stored in the account.

I decided to poke at the site because I look into phishing sites all the time at work when users report suspicious emails to us, and I find this kind of work enjoyable.

I decided to do a write-up of this because it's actually very well put together attack that looks legitimate on the surface and takes some effort and technical know-how to recognize as malicious.

-----
# Dissecting A Phishing Site
-----
I did grab a screenshot of the main page, but the site presented itself as some big gaming tournament for first-person shooters, and falsely represented itself as being sponsored by Intel. The site asked you to log into Steam if you wanted to participate.

One of the first things I looked at was the certificate:
![]({{site.baseurl}}/assets/images/steam_phish/9.png)

Compare that to Steam's certificate:
![]({{site.baseurl}}/assets/images/steam_phish/8.png)

It doesn't mean too much since the main site wasn't immediately trying to pose as Steam but it was still interesting to check. 

The next thing I did before I really tore into the page's source was to provide fake login details to the "Steam" login and catch the request with Burp Suite:
![]({{site.baseurl}}/assets/images/steam_phish/1.png)

The pop-up window that held the login had the URL of the real steam displayed but as evidenced by this POST request, it wasn't actually sending the details to steam, but rather a PHP script on the scammer's site. The credentials are also passed in the clear.

![]({{site.baseurl}}/assets/images/steam_phish/2.jpg)

This looks real at first glance, but clicking within the address bar does not give you the option of backspacing or typing, which is odd. Also, if you click on the green padlock icon, you won't get any display showing information about the alleged SSL cert.

Trying to get to the bottom of this, I used Inspect Element to look at what was happening. I had a suspicion that this was just an iframe, and my hunch was right:
![]({{site.baseurl}}/assets/images/steam_phish/3.png)

So right away the game is up, but stopping here wouldn't be any fun right~? Scrolling randomly through the code I see a lot of attempted obfuscation which always makes the site look that much more suspicious:
![]({{site.baseurl}}/assets/images/steam_phish/4.png)

While digging in the code for the iframe I found where they put the image for the fake HTTPS logo:
![]({{site.baseurl}}/assets/images/steam_phish/5.png)

Inspecting further, I found how they spoofed the "URL" in the fake address bar:
![]({{site.baseurl}}/assets/images/steam_phish/6.jpg)

Pretty interesting, I haven't seen this kind of thing before. This is the main reason that this phish is so convincing; we're often told to "look at the address" when being asked to log in, since most phishing sites don't spoof the URL and they can be recognized as malicious on sight. But this scammer clearly knows what they're doing, or perhaps they purchased some kind of tool kit that builds this stuff for them.

A funny side note. Phishing sites usually have very poor security, so I can't help myself:
![]({{site.baseurl}}/assets/images/steam_phish/7.jpg)

There wasn't much more to discover. The site would take any non-existant URL typed in and redirect you to the main page, so enumerating pages with something like Gobuster wasn't possible, as literally everything would return a `200 OK` HTTP response.

At some point during my digging someone must have apparently reported the site, because I tried to refresh the page and got presented with this:
![]({{site.baseurl}}/assets/images/steam_phish/10.jpg)

As an addendum, I tried to go back to this site a couple of days later to get more screenshots for this write up, and after bypassing all the phishing warnings, all I saw was this:
![]({{site.baseurl}}/assets/images/steam_phish/11.png)

Looks like the attacker voluntarily wiped the site clean, though for some reason they left the server standing. It's possible they may want to reuse the server for a new scheme, and this could be a placeholder until they get a new domain and rebuild a new fake site.

Kudos to Google Web Security for blacklisting sites so rapidly after they are reported. This scam was so genuine-looking that unfortunately, people were falling for it.

I hope people can learn from this write-up, and become more diligent when clicking on random links that ask for a login. No matter how real it looks, it could just be a well-hidden trap.
