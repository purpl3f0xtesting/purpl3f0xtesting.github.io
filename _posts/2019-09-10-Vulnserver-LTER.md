---
published: false
---


-----
# Intro
-----
This was probably one of the more complex Vulnserver exploits that I made, requiring lots of jumping around, stack adjustments, and the infamous alphanumeric character restrictions. I took this as an extra opportunity to practice some manual encoding but also sped things up with [this nice script here](https:github.com/ihack4falafel/Slink/blob/master/Slink.py)

While on one hand I found this more complex than the HP NNM exercise in the CTP course, I simultaniously found it easier, probably just because I was familiar with the concepts at this point, and also knew how to fix a mis-aligned stack (not to mention how to spot it).

By the way, for anyone who may be confused about the new look in the screenshots, I started using [x64dgb](https://x64dbg.com/) because of some of the nicer features here and there.

-----
# Part 1 - Starting with the SEH overwrite
-----

Long-time readers of my original blog should be familiar with the basics of an SEH overwrite, so I'll skip over the finer details and just get right to the important bits.

SEH overwrite occurs when sending 4000 "A"s, and the offsets for SEH and nSEH are around 3500:

![]({{site.baseurl}}/assets/images/lter/01.png)

![]({{site.baseurl}}/assets/images/lter/02.png)

![]({{site.baseurl}}/assets/images/lter/04.png)

![]({{site.baseurl}}/assets/images/lter/05.png)

So right away I've got a basically functioning SEH overwrite in the works. From here on out, the alphanumeric restrictions will start affecting how the exploit is written.

-----
# Part 2 - Alphanumeric shellcoding
-----

So now we get to the interesting part. A jump that doesn't use restricted characters. Remember that alphanumeric means we can only use 0x01 thru 0x7F. The usual `JMP SHORT` opcode, `\0xEB`, is outside of that range, so we can't use it.

There are a few different ways to go about jumping with alphanumeric characters, but I chose to do it with a conditional jump, `JZ`, or `Jump if 0`. I chose this because I noticed that the ZF flag was set to 1.

![]({{site.baseurl}}/assets/images/lter/06.png)

After taking the jump:

![]({{site.baseurl}}/assets/images/lter/07.png)

Now I'm in the buffer of "D"s. There isn't a ton of room to work with here so I want to start jumping backwards into the "A" buffer. The biggest jump I can make with a `JMP SHORT` is 127 forwards or backwards, so for now I have to settle on "carving" out the shellcode. To exercise my skills in doing this manually, I did the math on my own to make the shellcode.

Starting out, I need to look at the address of the last "D", and compare that to ESP. I need to make ESP point to the bottom of the "D"s so that any carved shellcode will appear there.

Bottom of the "D"s: 0x00E8FFFE
ESP after the jump: 0x00E8EC8C
A difference of 0x1372, or 4978 decimal bytes.

Fortunately, setting up ESP can be done with alphanumeric-friendly opcodes, so this won't have to be encoded.

```nasm
54          PUSH ESP        ; Push the value of ESP onto the stack
58          POP EAX         ; Pop that value into EAX
66057013    ADD AX, 0x1370  ; Add 0x1370 to the AX register to make it 0x00E8FFFE
50          PUSH EAX        ; Push this new value onto the stack
5C          POP ESP         ; Pop this value into ESP
```
