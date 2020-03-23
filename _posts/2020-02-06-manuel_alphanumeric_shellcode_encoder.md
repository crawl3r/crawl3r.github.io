---
layout: post
title: "Manuel - Alphanumeric shellcode encoder"
---

## Introduction:
I am currently working through my OSCE prep material that I have gathered over the past few months, mainly generated after reading reviews and talking to friends who have taken the course. Without giving away any spoilers or specific information about the labs or the exam, I was able to highlight some key areas that I should study and prepare for. This post is specifically is about tackling character limitations within shellcode. Shellcode can sometimes be translated, mangled or misinterpreted during shellcode injection.

The easiest example is the NULL byte character ("\x00"). As shellcode is often injected into the process using buffer overflows via user input, the NULL character specifies the end of the user input. Naturally, this means having this character halfway through our shellcode would result in only half our payload being injected causing problems, and well, ultimately, a non working exploit.

## What is alphanumeric shellcode
To start with, I want to take a brief look into alphanumeric shellcode. Specifically what it is and why we should be interested in knowing about it. Reading through some OSCE reviews, I noticed a lot of talk about learning SEH overwrites and encoded shellcode to be make sure any bad character filtering or character translation in general does not break our payloads.

Now, up to this point - I hadn't had a chance to play with any on the weaknesses in vulnserver or create my own exploits. However, reading about alphanumeric shellcode in general, it seems to be filtering that only allows the hex characters ranging from 0x00 through to 0x7F. As we have already touched on, 0x00 is a NULL byte character, so naturally we will want to ignore that during our work, however it is important to note that all these characters are within the allowed scope. What's also important to note is that each hex value within this range is relative to an [ASCII character](http://www.asciitable.com/ "Shellcode").

An example situation where this sort of shellcode could come in handy would be user input that is specifically checked to be ASCII characters only (don't quote me on this, this is me guessing as to why something might specifically look and monitor for ASCII). For example, if a website asks for a some information about a user - they may sanitise the data to only allow characters that can be successfully rendered within the web page upon reflection. As you may have seen, if shellcode is printed to the terminal you sometimes receive broken icons. Where as ASCII characters would successfully print.

An example of this can be seen in the screenshots below. I used this [shellcode](http://shell-storm.org/shellcode/files/shellcode-827.php "Shellcode") within my testing. 

```
\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x53\x89\xe1\xb0\x0b\xcd\x80
```

The first screenshot shows the raw shellcode being printed directly to the terminal, note the broken icons throughout the output. The second screenshot prints the same shellcode, however this has been encoded into alphanumeric shellcode using Manuel (a small script I put together whilst learning the encoding process), and prints all working ASCII characters without any issues.

![Corrupt character output]({{ site.url }}/assets/images/alphanumeric/default_shellcode_output.png)

![Clean character output]({{ site.url }}/assets/images/alphanumeric/encoded_shellcode_output.png)

"But Gary, those two screenshots show two completely different chunks of shellcode. Infact one is massive compared to the other - so they can't be the same." 

Okay okay, technically, yes - you are correct. The shellcode isn't the same, that's obvious - however, they both do the exact same thing. They will both drop the user into a new shell instance on unix (if used correctly). The reason the sizes differ so much, is due to the approach required to get around the limited character set. Let's break down an example of using the first set of shellcode to pop our shell (these breakdowns won't include any forms of exploitation or getting data onto the stack, just what it should look like):

* We inject our shellcode payload into the target process
* The shellcode is pushed into the process memory (likely the stack) as is
* Gain control of EIP and redirect execution to our shellcode starting address

As we can see, that is pretty simple to understand (ignoring all the additional information required to actually get our shellcode into the processes memory. However, the second set of shellcode shown in the above screenshot requires a bit more explanation.

* We inject our shellcode payload in the target process
* The encoded shellcode is pushed into the process memory (likely the stack) as is <-- key point
* Jump to the encoded shellcode (I won't cover this here)
* Encoded shellcode executes, placing our decoded shellcode on the stack <-- key point
* Jump to decoded shellcode

You will notice that compared to the first breakdown, we actually jump to 2 seperate chunks of shellcode before gaining our required functionality. Within the second breakdown, I have highlighted two seperate points that we will look into closer to make sure we understand exactly what is happening.

## Our encoded shellcode
So, now we know we can encode our shellcode - what exactly does this mean? A pretty common approach of encoding shellcode to bypass certain restrictions is with the use of XOR'ing our bytes with a key value. This is a really quick way of bypassing basic validation, some anti-virus solutions etc. By encoding the shellcode and adding a decoder loop within the payload - malicious shellcode can go undetected before being decoded in memory and execution properly. The issue here is, XOR'd bytes can still (and will likely) include bad characters - in this case, anything outside of the "0x00 - 0x7F" character range.

In this case, we are just wanting to find a way to get our required shellcode on the target stack, allowing us to jump to it and pop our shell. A method of achieving this comes via subsidising our bad character shellcode with a series of instructions, performing simple mathematics on legal character values to obtain our original, required, value. At first, this logic seemed crazy to me - it took me a couple of reads of [H0mbre's post](https://h0mbre.github.io/LTER_SEH_Success/ "H0mbre") as well as a post he also recommended on [Vellosec](http://vellosec.net/2018/08/carving-shellcode-using-restrictive-character-sets/ "Vellosec"). I recommend you check these posts out as I won't be going into as much detail as they do, however, a breakdown of how a few math operations can be used in order to achieve our desired value can be seen below:

* Zero out our EAX register
* Create two (or three) values with ASCII friendly hex values
* ADD these values to EAX value
    * If we had 3 values, we want to SUB a fourth pre-defined value
* EAX should now hold our original (before encoding) shellcode hex values
* PUSH EAX onto the stack

The first thing we want to do is zero (clear) out our EAX register as this will be used as our accumulator, holding our results each time we complete our required steps. Normally we would zero out a register by XOR'ing it with itself, however due to the [table](https://www.cs.uaf.edu/2017/fall/cs301/lecture/09_29_machinecode.html "Machine code x86") shown below the register values are represented by opcodes that aren't within the ASCII character range.

![mod rm table]({{ site.url }}/assets/images/alphanumeric/mod_rm_table.png)

As we can see, the instructions "xor eax, eax" would equal the opcode value of "33C0" which includes C0, falling outside of our legal character range. For this reason we have to be a little more crafty, let's look at the AND method. As a disclaimer, I used the content from the vellosec blog post above to complete my work with AND operations. The post covers it very well and is definitely worth looking over if you are struggling - however, I will give a brief explanation below with similar content, but please go check that post out as well!

A basic understanding of AND is the following:
If both values equal 1, then the result is 1, otherwise the result is 0.

1 AND 1 = 1  
1 AND 0 = 0  
0 AND 1 = 0  
0 AND 0 = 0  

We can look at a few examples using binary:

The following values:  
11001100  
11000011  

Would equal:  
11000000  
  
Where:  
11001100  
00110011  
  
Would equal:  
00000000  

From the second example, we can get an idea of how we could use the AND operator to zero out a register. We also need to confirm that the opcodes used for the AND operator are within our allowed character set. Looking at the [table](https://x86.puri.sm/html/file_module_x86_id_12.html "Mod RM Table x86") on we can see that unlike the required opcodes for the XOR instruction (33C0), we can use the opcode 25 to complete our "and eax" operation. This value falls within our allowed character range and will allow us to use the above logic to zero out our registers. The next step is to find two 4 byte values that do not include any bad characters, and when used together in a logical AND operation give us the value of zero. Once again, we thank vellosec for the following two values:

554E4D4A and 2A313235

Looking at the following screenshots, we can see that our glorious calculator has given us the binary representation of our two values and when used in our AND operation, we are presented with our required zero value. 

![calc 1]({{ site.url }}/assets/images/alphanumeric/calc_1.png)
![calc 2]({{ site.url }}/assets/images/alphanumeric/calc_2.png)
![calc 3]({{ site.url }}/assets/images/alphanumeric/calc_3.png)

Now we have two values we can use to zero our EAX register, we can move on to using ADD and SUB to obtain our required shellcode value without ever having to inject any bad characters ourselves. So how do we do this? Looking back at the breakdown we see that we want to "Create two (or three) values with ASCII friendly hex values". There are obviously various ways to do this, I initially started to write a program that would start at the required value in hex, attempt to split that value in half (if it was even, great, if it was odd, I would minus one, half it, then add one back to one of the halves). From there I used an incremental offset that I would add to one half and subtract from the other in order to try and find two values that would equal the target value. This did work for some values (although it did take a while, so my code was probably awful) but there were some problems when it comes to values that included certain characters. This is where I took a step back and had a look around online.

Enter [Slink](https://github.com/ihack4falafel/Slink "Slink by ihack4falafel") by [@ihack4falafel](https://twitter.com/ihack4falafel "ihack4falafel Twitter"). Slink uses a really simple method of generating these values which was super quick and pretty much idiot proof (which was a bonus for me). After having a play around and just reading the code (thankfully, reading Python is like reading a book) I wrote down the following logic.

* Split the shellcode up into chunks of four bytes (pad with NOPs beforehand if it's not divisible by 4)
* Check the bytes for any known bad bytes
* If we don't have any of the specified bad bytes
    * Loop through each of the eight characters
    * Take the single character and find two decimal values that can be added together to equal this value (i.e 4 == 2 + 2, 5 == 3 + 2 etc)
    * Place these two values on seperate strings and continue the loop
* If we do have these specified bad bytes, we need to do something a little different
    * Loop through each of the eight characters
    * Take the single character, but this time we want to find three characters that add together PLUS an additional offset (i.e 5 == 2 + 2 + 2 (if the offset is 1), 8 == 4 + 3 + 3 (if the offset is 2)) - we will discuss this offset shortly and why it exists
    * Place these three values on seperate strings and continue the loop
* Once all eight charcters are finished, we will result in either two strings of eight characters that when added together will equal our original shellcode value or we will have three strings that will total the required value plus a string of eight of our required offset (i.e 11111111 or 22222222).

Before we move on, let's talk about this offset that has appeared in the lower half of the above loop logic. When checking for various characters, especially the 'f' character we are unable to calculate this using valid characters. As 'f' is equal to 15 in decimal, we would need to perform an addition such as 8 + 7. The value, combined with other hex values, could potentially create an illegal byte, depending on the order of the characters. For example, 0x80. As we can cannot exceed a certain value within our addition we can use a third value within our series of instructions, followed by a subtraction to help bring our overall value back down to the target. By using three seperate values, we overshoot our target - hence the requirement of the subtraction. There are probably other methods of completing this and getting the correct value, however during my time researching this topic and taking Slink apart (referenced above), this approach proved to be very effective. I used the hard coded value of 0x33333333 for my subtration, meaning each calculation used an offset of three within the addition. To finalise this, let's picture the following examples:

Target: F (15 + offset of 3 = 18)  
Possible values: 6 + 6 + 6, 7 + 7 + 4, 7 + 6 + 5 etc  

It is important to note (again) at this point, that the max value we can use within this set of instructions is 7 as the possible byte combinations will be within our legal range of ASCII characters (0x70, 0x71, 0x72, 0x73, 0x74, 0x75, 0x76, 0x77). Knowing this, we can now create or two seperate encoding processes:

* ADD, ADD
* ADD, ADD, ADD, SUB

After the correct encoding route (one of the above) has been complete, our desired value will be held in our EAX register. The final step of encoding this single chunk of shellcode is to push it onto the stack ready for usage. Luckily, 'push eax' can be accomplished with a legal character opcode 0x50 so we don't have to do anything else other than call that instruction.

Let's look at two chunks of shellcode that has been encoded and formatted by Manuel. The first chunk of shellcode follows the ADD ADD encoder, using two seperate hex values to accomplish the target value:

```
[*] Encoding chunk: 1 -> 9080cd0b

\x25\x4a\x4d\x4e\x55 => and eax, 0x554e4d4a
\x25\x35\x32\x31\x2a => and eax, 0x2a313235
\x05\x06\x67\x40\x50 => add eax, 0x50406706
\x05\x05\x66\x40\x40 => add eax, 0x40406605
\x50                 => push eax
```

The second chunk of shellcode follows the ADD ADD ADD SUB encoder, using three generated hex values and the hardcoded subtraction value:

```
[*] Encoding chunk: 4 -> 69622f68

\x25\x4a\x4d\x4e\x55 => and eax, 0x554e4d4a
\x25\x35\x32\x31\x2a => and eax, 0x2a313235
\x05\x34\x26\x32\x34 => add eax, 0x34322634
\x05\x34\x26\x32\x34 => add eax, 0x34322634
\x05\x33\x16\x31\x34 => add eax, 0x34311633
\x2d\x33\x33\x33\x33 => sub eax, 0x33333333
\x50                 => push eax
```

So, the good news is - we now know how to encode our shellcode into ASCII friendly bytes, preventing manipulation and incorrect injection results. However, the bad news is, we can't execute this shellcode in a similar fashion to the standard (non-encoded shellcode). We need additional instructions to re-align the stack and point execution towards our decoded shellcode. However, this post will not cover that as I personally have not covered that yet... soon.

To finish up this post, I will add a few usage videos of my tool Manuel which effectively automates the above into a handy cli tool. The tool and source code can be found on my Github in my [CTP/OSCE repo](https://github.com/crawl3r/CTP-OSCE "Crawl3r CTP-OSCE Github").

Standard usage
[![asciicast](https://asciinema.org/a/wctxFuqCnkM6gyPNn4KC3XFqo.svg)](https://asciinema.org/a/wctxFuqCnkM6gyPNn4KC3XFqo)

Usage with debug output
[![asciicast](https://asciinema.org/a/7VhMrAVPTm1W6V5P0afuxdUCt.svg)](https://asciinema.org/a/7VhMrAVPTm1W6V5P0afuxdUCt)

Usage with python ready output
[![asciicast](https://asciinema.org/a/sphDgkkyCrEuUonM3xNIEPuOd.svg)](https://asciinema.org/a/sphDgkkyCrEuUonM3xNIEPuOd)
