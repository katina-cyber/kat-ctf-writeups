# Toddler's Bottle: 1 - fd (Detailed)

First Written: 5.21.2020

Last Updated: 5.21.2020 

[pwnable.kr](http://pwnable.kr/) is a lovely set of pwn challenges. 

Be sure to try to solve the problem on your own first without looking at any writeups. Trust me, you'll feel a lot more confident that way!

Let's take a look at the first problem, fd. 

# Steps to pwn
First, ssh into the server. The password is *guest*

**SSH FROM A VIRTUAL MACHINE (VM), SET UP A (VIRTUAL PRIVATE NETWORK) VPN. Plus having a VM ready will help you keep your CTF files separate from your personal ones (you're taking notes while you tackle challenges, right?**

```
ssh fd@pwnable.kr -p 2222
```

Where:
- ssh is the command we use to ssh into a server
- fd is the user
- pwnable.kr is the host
- -p 2222 specifies that we are connecting through port 2222

List the files in the user fd's home directory using **ls** after you successfully login.

```
fd@pwnable:~$ ls
fd  fd.c  flag
```

We see a C source file *fd.c* , and two other files, *fd* and *flag.*

Run the **file** command with the *fd* and *flag* filenames respectively to get learn their filetypes.

```
fd@pwnable:~$ file flag
flag: regular file, no read permission
fd@pwnable:~$ file fd
fd: setuid ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), dynamically linked, interpreter /lib/ld-, for GNU/Linux 2.6.24, BuildID[sha1]=c5ecc1690866b3bb085d59e87aad26a1e386aaeb, not stripped
```

We see that *flag* is a regular file, but we don't have permission to read it.

We also see that *fd* is an ELF executable, so let's run it and see what happens. 

To run an executable in Linux, run **./name_of_executable_to_run**

```
fd@pwnable:~$ ./fd
pass argv[1] a number
```

This means that *fd* wants a number as an argument. 

To pass an argument to an executable, run **./name_of_executable_to_run myarghere** where you put a space after the name of the executable and type your argument.

Let's try 0 as argument and see what happens. 

```
fd@pwnable:~$ ./fd 0
learn about Linux file IO
```

Ahh, a passive aggressive statement, but no flag. 

*Side note: You may be wondering why you see square brackets on argv[1] indicating that argv is an argument array. Don't arrays start at index 0? But then the program is telling us to pass an argument to argv[1], and we're only passing one argument! The argument is passed to argv[1] because argv[0] contains the name of the program*

There is one file we haven't looked at yet, *fd.c* - which contains the source code of the *fd* executable (in some pwning problems, authors may feel nice enough to include the source code of the executable for you to comb through). 

To view the contents of a file, run **cat file_to_see_contents_of** 

Let's view the contents of *fd.c*

```
fd@pwnable:~$ cat fd.c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
char buf[32];
int main(int argc, char* argv[], char* envp[]){
        if(argc<2){
                printf("pass argv[1] a number\n");
                return 0;
        }
        int fd = atoi( argv[1] ) - 0x1234;
        int len = 0;
        len = read(fd, buf, 32);                                                                     
        if(!strcmp("LETMEWIN\n", buf)){                                                              
                printf("good job :)\n");                                                             
                system("/bin/cat flag");                                                             
                exit(0);                                                                             
        }                                                                                            
        printf("learn about Linux file IO\n");                                                       
        return 0;                                                                                    
                                                                                                     
}   
```

If you have never seen any C code in your life, don't fret! Take the time to break down each part of the source code. 

Google the different functions you see in the source code and learn what they do. 

After some research (feel free to use the links under the "Resources for the brain" section), check that your understanding of what the source code does matches with mine. I have added annotations as comments below. 

```
/*include directives here*/
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

/*make space for a string named buf that can hold up to 32 characters*/
char buf[32];

/*main function*/
int main(int argc, char* argv[], char* envp[]){
	/*if the length of the argv array is less than 2, print a message and return 0*/
        if(argc<2){
                printf("pass argv[1] a number\n");
                return 0;
        }
	
	/*convert the user's passed argument into an integer*/
	/*then subtract from it 0x1234 (which is 4660 in decimal)*/
        int fd = atoi( argv[1] ) - 0x1234;
	
	/*define the integer len to be 0 at first*/
        int len = 0;
	
	/*set len equal t the result of read and the following arguments*/
	/*read(file_descriptor, string_to_write_file_contents_to, number_of_bytes_to_write*/
        len = read(fd, buf, 32);

	/*if the result of the string "LETMEWIN\n" and buf are equal, print the flag file*/
	/*the flag file is printed using system("/bin/cat flag")*/                                           if(!strcmp("LETMEWIN\n", buf)){                                                              
                printf("good job :)\n");                                                             
                system("/bin/cat flag");                                                             
                exit(0);                                                                             
        } 
	
	/*otherwise, print a passive agressive message*/                                                     printf("learn about Linux file IO\n");                                                       
        return 0;                                                                                    
                                                                                                     
}       
 ```

Two C concepts to note:
- strcmp compares two strings. If the two strings are equal, it returns 0. Otherwise, it returns a nonzero integer
- The typical "true" and "false" boolean values in other languages are represented in C as a 1 for "true" and a 0 for "false" - this may seem confusing when considering that strcmp returns 0 if two strings are equal

The **read** function is the ticket to printing our flag. It needs a *file descriptor* as its first argument. 

A *file descriptor* is an nonnegative integer that represents an open file in Linux. File descriptors attached to every process in Linux include 0 for *Standard In* (keyboard input), 1 for *Standard Out* (console output), and 2 for *Standard Error* (console error output). 

Knowing that the file descriptor 0 allows for keyboard input, and that 4660 is subtracted from the argument we pass to *fd*, what is passed an argument such that the file descriptor passed to the **read** function is 0? 

We can pass 4660 as an argument to *fd*

```
fd@pwnable:~$ ./fd 4660

```  

Notice how the file seems to hang... 

You know that the file descriptor passed to read is 0. 

What if we typed LETMEWIN and hit enter (to represent our the \n newline character)?

```
fd@pwnable:~$ ./fd 4660
LETMEWIN                                                                                             
good job :)                                                                                          
mommy! I think I know what a file descriptor is!!  
```

Awesome! 
                                                                                                 
## Resources for the brain
- [SSH protocol](https://www.ssh.com/ssh/protocol/)
- [About the ssh command in Linux](https://www.ssh.com/ssh/command/)
- [About the cat command](https://linuxize.com/post/linux-cat-command/)
- [How to execute a file in Ubuntu](https://howtoubuntu.org/how-to-execute-a-run-or-bin-file-in-ubuntu)
- [Atoi - C Library Function](https://www.tutorialspoint.com/c_standard_library/c_function_atoi.htm)
- [Read (C System Call](http://codewiki.wikidot.com/c:system-calls:read)
- [Strcmp - C Library Function](https://www.tutorialspoint.com/c_standard_library/c_function_strcmp.htm)
- [Decision and Branching Concepts in C](https://www.cs.uic.edu/~jbell/CourseNotes/C_Programming/Decisions.html)
- [What is a File Descriptor? StackOverflow](https://stackoverflow.com/questions/5256599/what-are-file-descriptors-explained-in-simple-terms)
- [File Descriptors - bottomupcs](https://www.bottomupcs.com/file_descriptors.xhtml)

### Author's thoughts on this problem

I felt **really salty** as salty as salt could be after figuring out the solution to this problem. For a lot of CTF problems I've done, I have definitely looked at writeups and then felt pretty dumb afterwards. After taking a couple days off from working on web development, and CTFs entirely (no laptop except for the occassional journal entry, bad internet, screen time mainly consisted of YouTube videos and video games), I felt refreshed so I decided to try to gain some CTF momentum again.

I learned that CTFs are 90% a mental battle (staying confident and persistent even if you don't get it right away or have to look at writeups, writeups help you LEARN), and 10% technical ability and time. 

With the right mindset, you can learn a lot, and that can help you build the endurance to put in more time to do better. But in the end, the knowledge earned is more valuable than a scoreboard or "looking good" or whatever. 

Seriously, this problem have to have taken me about 4-6 hours, but I learned a lot. This writeup that you see here, is a polished, cut down, trimmed version of the amount of time and work I put into this problem. I refrained from looking at a writeup for this one. 

 
