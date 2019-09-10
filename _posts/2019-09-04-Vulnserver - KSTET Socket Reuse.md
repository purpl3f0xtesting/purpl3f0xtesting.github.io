---
published: true
title: OSCE Prep - Vulnserver KSTET - Socket Reuse
---
-----
# Intro
-----
***Before I say anything...!***

---> [All credit goes to this awesome guy here!](https://rastating.github.io/using-socket-reuse-to-exploit-vulnserver/) <---

Without this post I'd have never even heard of socket reuse in buffer overflows. This was completely new territory for me and something I haven't really&nbsp; seen in anything I've run across in Exploit-DB, so I **heavily** referenced this while performing the work. I take no credit for coming up with anything that follows. I only reiterate everything here to try to help myself remember the process by trying to explain it in my own words (mostly), as well as simply cataloguing my OSCE prep.


-----
# Part 1 - Normal EIP overwrite
-----


No fuzzing this time; I mean c'mon, it's vulnserver. Just throw a lot of bytes at it and crash it already.

![](https://i.imgur.com/taPRA2f.png")

This is how I started the exploit. Of course I just cheated and looked at what Rastating had on his blog post so I could skip the unimportant stuff and get right into the meat of this exploit. Still, just for sanity, I ran this as-is instead of skipping right to finding the `JMP ESP` so I could make sure I get the right EIP overwrite. Sure enough, I did:

![](https://i.imgur.com/bAfNm67.png)

So that's in order, time to get the `JMP ESP`.

![](https://i.imgur.com/1arhwG8.png)

Pretty much another given if you've been doing vulnserver before.

As expected this lands me at the start of my C buffer, but that's when the problem here becomes apparent:
(borrowing this screenshot from rastating since I forgot to take my own)
![](https://rastating.github.io/assets/images/2019-06-21-using-socket-reuse-to-exploit-vulnserver/0001.jpg)

For whatever reason, this command is truncating the C buffer to something that's extremely short and practically useless. So first order of business is jumping up into the A buffer.

Now Rastating points out a very cool trick here. While he didn't use Immunity, this trick will still work in Immunity and most likely Olly.
Instead of trying to do maths to figure out the jump, we can just let Immunity do it:

![](https://i.imgur.com/HdD4xGm.png")

Scrolling up, I can see that there's a little weirdness going on with my A buffer. There are two A's above that last line there, so I take the address `0x0123F9C4` and just sub 2, and then tell Immunity that I want to jump to that address:

![](https://i.imgur.com/iFwuZm3.png")

![](https://i.imgur.com/KrPsqzl.png)

There, now I have my opcodes without having to open up NASM_shell or google or do maths. Might seem lazy but, considering that I anticipate cooking my brain during OSCE, I'm always open to cheap shortcuts like this.

Stepping into the instruction, I get:

![](https://i.imgur.com/6S5JAl9.png)

Perfect. I hit the very first A in the buffer. Now I have 70 bytes to play with.


-----
# Part 2 - Understanding sockets
-----


![](https://i.imgur.com/0N4O9Z3.jpg)

Again, borrowing this from Rastating's post, because it's a good visual aid. So when making a TCP connection, as Vulnserver does, the server has to set up the socket in this manner. The binding and listening occurs on a per-port basis, which is why this exploit is a socket *reuse*, and not just making a new socket, because that would mean opening up a new port. Once the socket and connection is established, programs can call `send()` or `recv()` to exchange data. `Recv()` will accept incoming data and write it somewhere to memory. The goal of this exploit is to invoke the `recv()` syscall, pass in the proper parameters to reuse Vulnserver's socket, and then send a second-stage payload that won't get truncated because it doesn't need to overflow the KSTET command (which is what I suspect is truncating the first payload).

To start figuring all this out, I have to restart vulnserver in the debugger and let it stay paused at the entry point. I start to scroll down until I see the `recv()` syscall:

![](https://i.imgur.com/thjsXIS.png)

Not terribly far down I find it. Looks like it's part of WS2_32, which makes sense. I've seen that being called when analyzing shellcode in the debugger, so I'd figured out that it was what Windows uses for sockets. I set a breakpoint here and run the program. Nothing happens since I haven't tried to connect yet, so I ran the exploit again. The breakpoint was hit, and the stack contained the parameters I needed:

![](https://i.imgur.com/dfkobV9.png)

Going off of what [what Microsoft says about recv()](https://docs.microsoft.com/en-us/windows/win32/api/winsock/nf-winsock-recv), these parameters basically mean:

```
	Socket = The socket descriptor
	Buffer = the location in memory that this incoming data will be written to
	BufSize = the allocated space for the incoming data
	Flags = Flags that can influence the behavior of the socket. None are set here.
```

It's important to note these because these need to be pushed onto the stack at the time that we make our own call to `recv()`. The trick is pushing them onto the stack correctly. There are a few obstacles in the way. The first should be obvious, the 0 we need for the flags parameter. We can't use null bytes in shellcode so that has to be pushed using other techniques. The less obvious one is the socket descriptor. This value will change every time the program is run, so it can't be hard-coded in the exploit. We have to find a way to dynamically find out what it is and reuse it.

A quick side note; Just for curiosity's sake, I followed the address pointed to by the "Buffer" argument in dump just to see if it really was pointing at my payload and:

![](https://i.imgur.com/oXnh9m1.png)

Probably obvious but I just felt like confirming that I was understanding things correctly.

So, before moving on, it's important to note the address of where `recv()` is by double-clicking the instruction:

![](https://i.imgur.com/X9qmyK7.png)

I put this in my notes and move on.

Letting the program execute until the NOP sled is hit, it appears that the parameters passed to recv() are no longer on the stack, overwritten by the overflow, so we really do have to manually push everything back onto the stack.


-----
# Part 3 - Time for some assembly
-----


Here's where knowing assembly becomes key. I've got a fair understanding but I learned some new stuff doing this too.

Restarting vulnserver inside of Immunity again, and then scrolling back down to the call to `recv()`, we need to analyze some of the assembly above the call. Once again my exploit is run and I hit the breakpoint at the call, and start looking at the assembly, and the registers.

`MOV DWORD PTR SS:[ESP], EAX`

Basically this means move the doubleword in `EAX` into `ESP` as a pointer to a memory address. If we look at `EAX` at this point in time, it's equal to `0x000000C8`, which is what we saw for the socket descriptor (I suspect it hasn't changed because I "restarted" it inside of Immunity, rather than shutting it down and re-attaching it. Above the MOV instruction is:

`MOV EAX, DWORD PTR:SS[EBP-420]`

Move the doubleword that is at the address stored in `EBP` - 0x420 bytes. Looking at EBP, it's pointing to `0x123FFB4`. Subtracting 0x420 gets me `0x123FB94`. I look at that address in the stack:

![](https://i.imgur.com/cy1CTZn.png)

Perfect~ I let the exploit run until I hit the first A in the buffer, and double-check that this hasn't been overwritten. It hasn't. Now I know where this value is.

This way, no matter what the socket descriptor is, I can always find it and use it.

Since this address can also change, it's time to do a little math. Sitting at the top of the A buffer we want to note where `ESP` currently sits:

![](https://i.imgur.com/phqhGQW.png)

0x0123FB94 - 0x0123FA0C = 0x188 bytes.

We'll use this information to do the prep work for pushing this onto the stack later. Since the stack grows downwards, args have to be pushed in reverse order, so this will actually be the last thing pushed. For now, we'll start building the assembly in Immunity to get the opcodes again, as well as test the instructions in real-time so we can watch it work:

```
	PUSH ESP - put the value of ESP on the stack
	POP EAX - put that value into EAX
	ADD AX, 0x188 - add 0x188 to get EAX to equal the address of the socket descriptor
```

![](https://i.imgur.com/YgY27iw.png)

Now before we really begin pushing anything, we need to look at something important:

![](https://i.imgur.com/W5kyYAi.png)

Remember that stack grows towards lower addresses. Right now, `ESP` is at a higher address than the stager we're building. We risk stepping all over the stager if we start pushing stuff on the stack now. We need to adjust `ESP` so that it's at an address well below the stager.

![](https://i.imgur.com/Zx4dd7f.png)

![](https://i.imgur.com/Hw05cyP.png)

This should work. The way the stager is being built, it's growing into higher memory, so the stager shouldn't step onto where we've just put ESP either. Now we're ready to go.

First arg to push is the flags arg. It's set to 0, but we can't use nulls in shellcode, so we'll just take advantage of a register that we're not using right now by changing it to 0 and pushing it onto the stack. The easiest way to 0 out anything is to XOR it against itself:

```
	XOR EBX, EBX - make EBX 0x00000000
	PUSH EBX
```

![](https://i.imgur.com/l59AfHA.png)

That's one argument down. The next is "BufSize". Shellcode should only be around 350~ish bytes, so we don't need much. Let's just use a round 1024 bytes, which is 0x400 in hex. To get around using nulls, we can just manipulat the "high end" of BX like so:

```
	ADD BH, 4
	PUSH EBX
```

![](https://i.imgur.com/hLZZQBW.png)

BH represents the 2nd byte in the `EBX` register, so by adding 4 to that, it's increased to `0x00000400` without having to use nulls.

Next arg to be set is the address where we want this payload to go. Again, hard-coding isn't an option, and we also want to make sure execution will be passed to the second payload. The easiest thing to do is to is calculate the distance look at where the A buffer ends, look at `ESP`, and figure out the difference.

`ESP` at this point is at `0x123F9A0`. The last A in my buffer is at `0x0123FA07`. My last A is 0x67 bytes away from `ESP`. Wanting to keep the stack an even number though, I just settle 0x64 byes:

![](https://i.imgur.com/hbirniw.png)

To get this value on the stack:

```
	PUSH ESP - put the value of ESP on the stack
	POP EBX - take that value OFF the stack and put it into EBX
	ADD EBX, 0x64 - increase the value by 0x64
	PUSH EBX - put that value onto the stack
```

![](https://i.imgur.com/gpFU3wV.png)

We're almost there. Only the socket descriptor is left. We can push that with a single instruction, but it's not as simple as `PUSH EAX`. `EAX` holds `0x0123FB94`. If we PUSH that, the stack will have `0x0123FB94` on it. The socket descriptor is `0x000000C8` though. So we use:

```
	PUSH DWORD PTR DS:[EAX]
```

This means "push the doubleword that is at the memory address `0x0123FB94` onto the stack"

![](https://i.imgur.com/SRhqc5N.png)

Now for another slick trick to avoid null bytes.

Recall that `recv(`) is at `0x0040252C`. We can't just put this in the shellcode because the null will terminate the rest of the payload, but there's a way around this:

```
	MOV EAX, 0x40252C90 - Move this value into EAX. This is our address minus the 00 at the front, and a NOP added on the end for padding.
	SHR EAX, 8 - shift the EAX register RIGHT by 8 bits. This will remove the 90 on the end, and add a 00 on the front.
	CALL EAX - make the syscall to recv()
```

![](https://i.imgur.com/0rab8WL.png)

At this point, our stager is complete. All of the arguments are on the stack, in the right order, and we're invoking `recv()`, which will take any additional buffer we send and write it into memory, at a location of our choosing. You can get the opcodes by highlighting all the assembly, right-clicking, and chosing "Binary Copy". Don't do "Copy to clipboard", that copies the addresses, opcodes, and assembly.

![](https://i.imgur.com/p94P8Rp.png)

Here's the payload so far. It's important to adjust the A buffer to accommodate the stager. Using python's `len()` function is a nice way to avoid doing the math yourself, and avoid mistakes.

Note that at the bottom I now have a sleep statement that pauses for 5 seconds before sending the second stage. This is just to make sure that the first stage executes and `recv()` is waiting for our buffer.

I set a breakpoint at `JMP ESP`, and then run the exploit:

![](https://i.imgur.com/3d6cvFB.png)

Perfection, the second stage lands in a spot where it'll get executed. Time to finish it off:

![](https://i.imgur.com/SayTID2.png) 

![](https://i.imgur.com/nDGZQ3C.png)

-----
# Conclusion
-----

It's weird. On one hand, this comes across to me as brilliantly simple, probably because I've been studying assembly outside of OSCE, but at the same time, it somehow feels remarkably complex. It feels like there's a million steps and it takes forever to do. But it's an amazing way to deal with such strict space limitations. I don't know how applicable this is in real world exploits, or if I'll see this on the OSCE exam, but I didn't want to omit such a useful technique just in case I do see it on the exam.

-----
# Links
-----

[Rastating's blog that made this possible](https://rastating.github.io/using-socket-reuse-to-exploit-vulnserver/)

[Microsoft documentation on recv()](https://docs.microsoft.com/en-us/windows/win32/api/winsock/nf-winsock-recv)
