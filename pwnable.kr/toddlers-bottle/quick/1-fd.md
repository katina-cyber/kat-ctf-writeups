# Toddler's Bottle: 1 - fd (Quick)

First Written: 5.21.2020
Last Updated: 5.21.2020 

# Steps to pwn

A *file descriptor* is an nonnegative integer that represents an open file in Linux. File descriptors attached to every process in Linux include 0 for *Standard In* (keyboard input), 1 for *Standard Out* (console output), and 2 for *Standard Error* (console error output). 

We can pass 4660 as an argument to *fd* to pass a file descriptor of 0 to the read functon.

```
fd@pwnable:~$ ./fd 4660
LETMEWIN                                                                                             
good job :)                                                                                          
mommy! I think I know what a file descriptor is!!  
```

Awesome! 

