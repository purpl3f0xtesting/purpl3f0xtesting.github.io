---
published: false
---
## A New Post

-----
# Intro
-----
After much hemming and hawing about making my own Vulnhub box, I sat down one night, and after a marathon session of evil laughing and chugging Dr. Pepper I created the box now known as FoxHole!

The box is meant to be easy-to-intermediate, with a fairly straight-forward initial foothold, but with a challenging privilege escalation that requires a bit of a niche skill.

Upon release I was happy to see people doing my box. Some got hung up on the privesc but it was fun hanging out in the Vulnhub discord and seeing people submitting my flag.

The box has been out roughly two weeks, and now that things have slowed down, I am releasing my official write up.

-----
# Part 1 - Initial Foothold
## Stepping into the Foxholes
-----
To start off, we just turn on the VM. DHCP is enabled so it should pick up an IP address.
I usually just scan the entire subnet that my VMs live on to find it:

![]({{site.baseurl}}/assets/images/foxhole/1.png)

Based on those ports, we can safely assume that `192.168.183.131` is the target, since it's running SSH and HTTP.

![]({{site.baseurl}}/assets/images/foxhole/2.png)

We look at the web page in Firefox to see what we're working with. Looks like some generic photo-sharing website, a personal portfolio for a photographer maybe. There are two links visible at the top, centered; Gallery, and About.

![]({{site.baseurl}}/assets/images/foxhole/3.png)

Looking at the gallery, we see some nature photography. The pictures appear to be in black-and-white, but when you hover over them, the color version is revealed.

![]({{site.baseurl}}/assets/images/foxhole/4.png)

On the about page, we don't really see a lot of anything going on. Scrolling around doesn't reveal anything.

Before digging too deep, we should circle back and do some basic recon.
Using gobuster, we enumerate some of the pages on the site:

![]({{site.baseurl}}/assets/images/foxhole/6.png)

![]({{site.baseurl}}/assets/images/foxhole/7.png)

That didn't really turn up much...there's almost nothing in these results. Why is that? Let's look closer at the URL of the site:

![]({{site.baseurl}}/assets/images/foxhole/5.png)

The extentions end with `.html`, which is something we can provide to gobuster to help it along a bit:

![]({{site.baseurl}}/assets/images/foxhole/8.png)

![]({{site.baseurl}}/assets/images/foxhole/9.png)

Better! There are a lot more results to work off of now. A few jump out right away:

* `admin.html`

* `secret.html`

* `phpmyadmin`

Let's start checking these out to see if there is anything interesting.

![]({{site.baseurl}}/assets/images/foxhole/10.png)

Looks like a phpMyAdmin login, but unfortunately, it's just an image. There is no login. It was just a rabbit hole...or maybe...it was a foxhole~

Now let's see if `secret.html` actually has anything secret:

![]({{site.baseurl}}/assets/images/foxhole/13.png)

More trolling. Scrolling down the page shows a vague hint that there are more hints to be found if we keep looking. 

What about `admin.html`?

![]({{site.baseurl}}/assets/images/foxhole/11.png)

Oh. Even more trolling. Or....is it? If we scroll down a little bit...:

![]({{site.baseurl}}/assets/images/foxhole/12.png)

So that fox picture on the main page might contain a secret! Let's download it and poke around:

![]({{site.baseurl}}/assets/images/foxhole/14.png)

We save the file to our attacking machine. Those who have done CTFs before might already know where this is headed: `steganography`.

But what is steganography?

```
Steganography is the practice of concealing a file, message, image,
or video within another file, message, image, or video. 
```

In other words, steganography (often referred to as "stego" for short) can be used to hide information inside of other files. The hint we found told us to take a close look at this fox image, hinting that stego is indeed the method to use.

The following tool is *not* installed on Kali by default, so you will need to install it with `sudo apt install steghide`. After it is installed, you can use it to either embed information into another file, or extract information that has been embedded into a file.

We can learn how to use it by invoking the built-in help with `--help`:

![]({{site.baseurl}}/assets/images/foxhole/15.png)

Now that we know what the options are for extracting files, we'll try it out:

![]({{site.baseurl}}/assets/images/foxhole/16.png)

Since we never found any password on the site, we just "wing-it" and hit enter without providing a passphrase, and it works! It dumps what appears to be a `base64`-encoded string. Decoding this is simple enough:

![]({{site.baseurl}}/assets/images/foxhole/17.png)

Easy! Now we have the password and a username. Recall that SSH is open on this box. This will let us not only walk right in the front-door, but we'll have a fully-functioning shell, which even supports auto-complete. Comfy!

![]({{site.baseurl}}/assets/images/foxhole/18.png)

True to the hidden file's message, we are able to log in as user `fox` with the provided password. The first thing I usually do when landing a shell as a user is to see what's in their home directory:

![]({{site.baseurl}}/assets/images/foxhole/19.png)

Right away we can see the intended priv esc; a file named `GiveMeRootPlz` that is highlighted red. What does that mean?

Since we have this nicely-functioning shell, the terminal highlights files in red if they have the `SUID bit` set. 

What does that mean?

```
Usually, Linux commands and programs run with the same set of permissions as the person who launches the program. When root runs the passwd command to change a password, it runs with root’s permissions. That means the passwd command can freely access the stored passwords in the /etc/shadow file.

What would be ideal is a scheme in which anyone on the system could launch the passwd program, but have the passwd program retain root’s elevated privileges. This would empower anyone to change her own password.

The above scenario is precisely what the Set User ID bit (SUID) does. It runs programs and commands with the permissions of the file owner, rather than the permissions of the person who launches the program.
```

Typically in a CTF setting, if we happen upon a custom binary with SUID set, it's most likely the intended method of obtaining root. That file is running with root permissions and we have to discover how to abuse that.

-----
# Part 2 - Getting root
## Time to get covered in bits inside of GDB
-----

The first thing we should do is just simply `run the file` to see what happens.

![]({{site.baseurl}}/assets/images/foxhole/20.png)

So the binary gives us a prompt, lets us type into it, and then mocks us for not convincing it to let us have root permissions. But it also drops a hint. Pay attention to the last line printed.

Before trying to attack this, let's do some very basic `reverse engineering` to see if this binary has anything interesting inside of it that is in plain-text:

![]({{site.baseurl}}/assets/images/foxhole/21.png)

Very interesting. We can see `/bin/bash` is inside this binary. Let's hold this in the back of our mind as we move forward.

Time to get to the attack now. Instead of just keyboard smashing for an hour, we can let python generate some long strings for us:

![]({{site.baseurl}}/assets/images/foxhole/22.png)

We feed this huge string into the binary and it crashes; note the message saying `Segmentation fault (core dumped)`. Many things can cause a binary to throw this error, but since we're slamming it with so much data, we can assume that we have overwritten something that has caused the program to attempt to access an invalid memory address, which would raise this error.

## The fun begins

GDB is usually installed on most Linux distros by default, but the **generous box-maker** has provided us with something extra: PEDA.

PEDA stands for Python Exploit Development Assistance and is a plug-in for GDB that assists in the exploit development process. It adds several new commands to GDB, and also makes things easier to look at by formatting data and even using coloration to make it all more pleasant to comb through.

We run the target in GDB with `gdb ./GiveMeRootPlz`, and then feed it that huge buffer again:

![]({{site.baseurl}}/assets/images/foxhole/24.png)

The application crashes, but GDB "catches" the crash and immediately shows us the important data that we want when analyzing a crash: The registers, the stack, the exception that was raised, `SIGSEGV`, and the faulting instruction. 

In our case, the faulting instruction looks weird. It's at address `0x41414141`. Why does it look like that?

Well, let's look at the ASCII chart:

![](https://i.imgur.com/Mt6OT8d.png)

According to this, hex 0x41 is the letter "A". This means that our huge block of "A"s has overwritten an instruction somehow. The program has crashed because memory address `0x41414141` is an invalid address. Either this address belongs to another process, or there's nothing at this address at all. Whatever the reason, `GiveMeRootPlz` isn't allowed to access this address, so we got a `SIGSEGV`.

So, now what?

We sent 1000 "A"s at this, how do we know at what point we took control of execution flow? PEDA to the rescue!

PEDA implements a new command into GDB, `pattern`. We can generate "cyclic pattern" (a repeating, predictable pattern), and then send it to the binary. This will help us discover where the overwrite happens.

![]({{site.baseurl}}/assets/images/foxhole/25.png)

By using `pattern create 1000` we are telling PEDA to make a cyclic pattern that is 1000 bytes long. By putting `pattern` again at the end, I am simply telling the command to save the output to a file. You can name this whatever you want. For example, you could run `pattern create 1000 find_eip`. No need to use the `>` operator as you would in normal bash.

After the pattern was generated, I ran the program and told it to use the contents of `pattern` as input by running `r < pattern`. The program crashed, this time saying it was because of instruction at `0x4e734138`. That doesn't look like it makes much sense, but don't worry. We can just copy that, and feed it to the command `pattern offset` and it tells us how many bytes into the pattern that this segment is.

Let's verify that we can overwrite EIP accurately:

![]({{site.baseurl}}/assets/images/foxhole/26.png)

Using python again, we print 516 "A"s, and then 4 "B"s. We pipe this into a file called `crash1` just so we don't have to copy-paste this string over and over.

We go back into GDB, and run it again, but with our new input:

![]({{site.baseurl}}/assets/images/foxhole/27.png)

This time, EIP was overwritten with `0x42424242`, which on the ASCII chart, is the letter "B".

## Abusing EIP control

