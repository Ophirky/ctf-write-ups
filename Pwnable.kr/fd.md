viewed all files and read the code of fd.c

Seeing that I read about file descriptors and understood that 0 means to read.
Seeing the code I saw that I needed to enter 0 +0x1234 but since atoi works only with integers I converted 0x1234 to a decimal number and got 4660.

I called the program with the sysargv of 4660. this requested input so I entered
LETMEWIN and new line and got the key
```bash
fd@ubuntu:~$ ./fd 4660
LETMEWIN
good job :)
[flag🤫]
```