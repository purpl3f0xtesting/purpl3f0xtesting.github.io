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


The second way we can download the binary is through PowerShell. This is probably more realistic since we won't always find something with RDP enabled, and may have a low-priv shell through other means:
<center><img src="/assets/images/av/12.png" /></center>  
<center><i><small>Figure 12 - Downloading payload with PowerShell</small></i></center>

Curiously enough, when we attempt to run Meterpreter with PowerShell, it initially fires, but almost immediately dies as Defender catches it and shuts it down:  
<center><img src="/assets/images/av/13.png" /></center>  
<center><i><small>Figure 13 - Meterpreter calls back but then dies</small></i></center>

We can also see how Defender has deleted our payload once again:  
<center><img src="/assets/images/av/14.png" /></center>  
<center><i><small>Figure 14 - Now you see it, now you don't!</small></i></center>

-----
# Preparing to bypass Defender
-----  

Now that we have proven that Defender is on and is catching our Metepreter payloads, we'll begin work on bypassing it.  
For starters, let's generate shellcode in the C# format, and while we're at it, let's go ahead and use MSFvenom's built-in encoders. This encoding alone won't be enough, but it is a good first step:  
<center><img src="/assets/images/av/15.png" /></center>  
<center><i><small>Figure 15 - Generating a C# payload</small></i></center>

In the payload output, pay attention to the size of the `buf` variable. This will be important later, so take a moment to make note of this:  
<center><img src="/assets/images/av/16.png" /></center>  
<center><i><small>Figure 16 - Byte array size</small></i></center>

## Adding more encoding  

Remember above when I stated that MSFvenom's encoding won't be enough by itself? The biggest reason for this is due to the shellcode containing a `decoder stub` inside of itself. It has a small decoding loop it goes through when it executes, and most AV engines today can detect encoded Meterpreter payloads based on that decoder stub. So, to get around this, we'll add an extra layer of encoding ourselves, to encode the stub!  
We open up Visual Studio Community, and make a C# console project called `XOR_encoder`, and begin to build our custom XOR encoder. We start by just pasting the shellcode from MSFvenom into a new byte array variable. Make sure the size matches what MSFvenom gave you!  

<center><img src="/assets/images/av/17.png" /></center>  
<center><i><small>Figure 17 - Adding shellcode to the encoder</small></i></center>

Next, we need to add the code that is the meat of the executable. I'll break down each line individually and explain what's happening.  
We declare a new byte array called `encoded` and assigning it the length of our buffer: 
```C#
byte[] encoded = new byte[buf.Length];
```  
This is a loop that will iterate over every byte and XOR it with the `^` operator, and use `0xAA` as the "key". Afterwards, the bytes are subjected to a bitwise `AND` with a value of `0xFF` to prevent them from becoming larger than 8 bits:  

```C#
for (int i = 0; i < buf.Length; i++)
{
  encoded[i] = (byte)(((uint)buf[i] ^ 0xAA) & 0xFF);
}
```  
This is formatting the bytes to be printed out 2 digits at a time, prepended with 0x, and appended with a comma: 

```C#
StringBuilder hex = new StringBuilder(encoded.Length * 2);
foreach (byte b in encoded)
{
  hex.AppendFormat("0x{0:x2}, ", b);
}
```  
Then we print it with:  
```C#
Console.WriteLine("The payload is: " + hex.ToString());
```  

All of the code together will look like this:  
<center><img src="/assets/images/av/18.png" /></center>  
<center><i><small>Figure 18 - Custom XOR encoder</small></i></center>

With the code written, we compile the binary, and then go execute it:  
<center><img src="/assets/images/av/19.png" /></center>  
<center><i><small>Figure 19 - Building the executable</small></i></center>

<center><img src="/assets/images/av/20.png" /></center>  
<center><i><small>Figure 20 - The output of the encoder</small></i></center>

## Getting the shellcode to run in a C# wrapper  

We now have double XOR-encoded shellcode, but we have to get it to run somehow. C# can do this as well.  
We'll make a new project and call it `shellcode_runner` or whatever you want, as long as you know what it does.  
We'll need to interact with the Windows API to make this work. C# can do this in a very round-about way, but it's made simpler thanks to a Wiki called `pinvoke`:  
<center><img src="/assets/images/av/22.png" /></center>  
<center><i><small>Figure 22 - Pinvoke.net</small></i></center>

Pinvoke has templates for invoking Windows API calls in C#, allowing you to simply copy-paste the code into your own project.  
For this shellcode runner, we'll need `VirtualAlloc()`, `VirtualAllocExNuma()` (you'll see why later), `GetCurrentProcess()`, `CreateThread()`, and `WaitForSingleObject()`:  
<center><img src="/assets/images/av/21.png" /></center>  
<center><i><small>Figure 21 - Preparing API calls in C#</small></i></center>

Next is the meat of the executable, the part that will actually run the shellcode while bypassing AV.  
Our XOR-encoded payload should bypass some signature detection, but we also need to bypass `heuristics` as well. AV engines will typically "execute" programs in a sandboxed environment to analyze their behavior for anything suspicious. We'll have to fool the heuristic engine in Defender to make it think our program is legitimate.  
The first thing we need to do in the code is set up the heuristics bypass. Since heuristics engines typically "emulate" execution instead of actually running the binary, we might be able to bypass detection by trying to invoke an uncommon API call that the AV engine **isn't** emulating. This would cause that API call to fail, and we can tell our program to halt execution if it detects this failure.  
In this way, we can make the heuristics engine flag our program as "clean" by just exiting the program before anything malicious happens.  
We'll invoke the `VirtualAllocExNuma()` API call to do this. This is an alternative version of `VirtualAllocEx()` that is meant to be used by systems with more than one physical CPU:  
```C#
IntPtr mem = VirtualAllocExNuma(GetCurrentProcess(), IntPtr.Zero, 0x1000, 0x3000, 0x4, 0);
if (mem == null)
{
  return;
}
```  
What we're doing here is trying to allocate memory with `VirtualAllocExNuma()`, and if it fails (if (mem ==null)), we just exit immediately.  
Otherwise, execution will continue. We'll use a similar loop to before to decode the shellcode. Make sure the XOR key is the **same** as before:  
```C#
for(int i = 0; i < buf.Length; i++)
{
  buf[i] = (byte)(((uint)buf[i] ^ 0xAA) & 0xFF);
}
```  

Then, we'll allocate memory. If we look at [the MSDN for VirtualAlloc](https://docs.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-virtualalloc), we can see that the arguments, in order, are the memory address to start at, the buffer size, the allocation type, and the memory protection settings:  

<center><img src="https://i.imgur.com/cTUbwUQ.png" /><center>  
<center><i><small>Figure 22 - MSDN for VirtualAlloc</small></i></center>  

We'll set the parameters to 0 (to let the OS chose the start address), 0x1000 bytes in size, 0x3000 to set the Allocation type to `MEM_COMMIT` + `MEM_RESERVE`, and set the memory permissions to `PAGE_EXECUTE_READWRITE` with 0x40:  

```C#
IntPtr addr = VirtualAlloc(IntPtr.Zero, 0x1000, 0x3000, 0x40);
```  

Next, let's copy the shellcode into this newly allocated memory:  
```C#
Marshal.Copy(buf, 0, addr, size);
```  

Now it's time to run the shellcode. We'll spawn a new worker thread, point it to the start of the shellcode, and let it run.  
Looking at [the MSDN for CreateThread](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createthread), we learn that the required arguments are the thread attributes, the stack size, the start address, additional parameters, creation flags, and thread ID.  

<center><img src="https://i.imgur.com/Fz5fMUM.png" /><center>  
<center><i><small>Figure 23 - MSDN for CreateThread</small></i></center>  
  
For most of these arguments wi'll supply 0s to let the API chose it's default actions, except for the start address, which will be the result that `VirtualAlloc()` returned to us earlier:  
```C#
IntPtr hThread = CreateThread(IntPrt.Zero, 0, addr, IntPtr.Zero, 0, IntPtr.Zero);
```  

Lastly, we make a call to `WaitForSingleObject()` to keep the thread alive; otherwise it will die. To make this call wait **indefinitely**, we give it a value of all hexidecimal Fs:  
```C#
WaitForSingleObject(hThread, 0xFFFFFFFF);
```  

The complete code:  
<center><img src="/assets/images/av/23.png" /></center>  
<center><i><small>Figure 24 - The shellcode runner completed</small></i></center>

With all this work done, we'll compile this binary. I keep my projects on a Samba share on Kali, and just run Visual Studio on my Windows host, so I'll switch back to Kali, and copy the compiled binary to my Apache web server:  
<center><img src="/assets/images/av/24.png" /></center>  
<center><i><small>Figure 25 - Copying the binary to Apache</small></i></center>  

Back over on the victim, we'll download the binary again:  
<center><img src="/assets/images/av/25.png" /></center>  
<center><i><small>Figure 26 - Downloading the new payload</small></i></center>  

Now, let's cross our fingers and run the binary again and see what happens!  
<center><img src="/assets/images/av/26.png" /></center>  
<center><i><small>Figure 27 - New payload still gets caught</small></i></center>

<center><img src="https://media1.tenor.com/images/b9a5f8f27fa8248cef817873d3bfc503/tenor.gif?itemid=16003613" /></center>  
<center><i><small>Figure 28 - Disappointment</small></i></center>
