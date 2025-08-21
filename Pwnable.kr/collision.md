As usual I started by seeing what files are available and as usual the same files were given to me
```bash
col@ubuntu:~$ ls -l
total 24
-r-xr-sr-x 1 root col_pwn 15164 Mar 26 13:13 col
-rw-r--r-- 1 root root      589 Mar 26 13:13 col.c
-r--r----- 1 root col_pwn    26 Apr  2 08:58 flag
```

```c
#include <stdio.h>
#include <string.h>
unsigned long hashcode = 0x21DD09EC;
unsigned long check_password(const char* p){
        int* ip = (int*)p;
        int i;
        int res=0;
        for(i=0; i<5; i++){
                res += ip[i];
        }
        return res;
}

int main(int argc, char* argv[]){
        if(argc<2){
                printf("usage : %s [passcode]\n", argv[0]);
                return 0;
        }
        if(strlen(argv[1]) != 20){
                printf("passcode length should be 20 bytes\n");
                return 0;
        }

        if(hashcode == check_password( argv[1] )){
                setregid(getegid(), getegid());
                system("/bin/cat flag");
                return 0;
        }
        else
                printf("wrong passcode.\n");
        return 0;
}
```

Now I needed to reverse engineer the `hashcode` variable to create a hash collision so that I could fake the password and get the flag.

These were my parameters:
 - input - 20 bytes, 
 - output - 0x21DD09EC
 - the function worked by taking the converting the string to five chunks of ascii values one next to each other and adding those up
   
 First I wrote a python script that will hash using this algorithm
 ```python
 def check_password(p: bytes) -> int:
    ip = struct.unpack('<5i', p[:20])
    res = sum(ip)
    return res
```

I tried opening an ascii table and creating my own password that will equal this:

| 21  | PD  | 09  | EC  |
| --- | --- | --- | --- |
| \v  | p   | \t  | v   |
| \v  | m   | \0  | v   |
| \v  | \0  | \0  | \0  |
| \0  | \0  | \0  | \0  |
| \0  | \0  | \0  | \0  |

`ophirky@opc:~/code$ ./main \vp\tv\vm\0v\v\0\0\0\0\0\0\0\0\0\0\0`
but for some reason it returned `0x7d356ec2`

Now I took the final number I wanted (0x21DD09EC) and divided it by 5.
that gave me 0x06C5CEC8 and 4 in the remainder.
so I knew that I needed to use the following bytes: `0x06 0xC5, 0xCE, 0xC8` where in one of the 0xC8 bytes I'll need to add the remainder (4).
so my number to enter is:
`0x06C5CEC8 * 4 + 0x06C5CECC`

now I passed the numbers using python to the program
```
col@ubuntu:~$ ./col $(python -c "print(bytes.fromhex('06c5cec8'*4 + '06c5cecc'))")
passcode length should be 20 bytes
```

seeing this I wanted to see what's happening:
```
col@ubuntu:~$ python -c "print(len(bytes.fromhex('06c5cec8'*4 + '06c5cecc')))"
20
```

So the length of the bytes is 20 but the program is saying the their not...
I wanted to understand what was going on so I started to read the source code of the python print statement to see what it adds to the output. The code was not very helpful.

After doing some more reading I discovered that python3 encodes the given string using UTF-8 that might cause some bytes to change size.
Since in python3 print became a function and not a statement I tried using **python2** which worked perfectly. I tested the length using a C program that prints it

```
col@ubuntu:/tmp/testing$ ./main $(python3 -c "print('\x06\xc5\xce\xc8\x06\xc5\xce\xc8\x06\xc5\xce\xc8\x06\xc5\xce\xc8\x06\xc5\xce\xcc', flush=True, end=None)")
35

col@ubuntu:/tmp/testing$ ./main $(python2 -c "print '\x06\xc5\xce\xc8\x06\xc5\xce\xc8\x06\xc5\xce\xc8\x06\xc5\xce\xc8\x06\xc5\xce\xcc'")
20
```

Now that it worked it was time to give it to the original executable
```
col@ubuntu:~$ ./col $(python2 -c "print '\x06\xc5\xce\xc8\x06\xc5\xce\xc8\x06\xc5\xce\xc8\x06\xc5\xce\xc8\x06\xc5\xce\xcc'")
wrong passcode.
```

I had the correct length but the password was incorrect.
Since I knew that the bits were good I decided to try and take a look at the endian direction (little-endian or big-endian)
```
col@ubuntu:~$ file ./col
./col: setgid ELF 32-bit LSB pie executable, Intel 80386, version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux.so.2, BuildID[sha1]=48d83f055c56d12dc4762db539bf8840e5b4f6cc, for GNU/Linux 3.2.0, not stripped
```

LSB means little endian and can also be viewed in using the `readelf` command.

since it was little endian I converted it and reran the program
```
col@ubuntu:~$ ./col $(python2 -c "print '\xc8\xce\xc5\x06'*4+'\xcc\xce\xc5\x06'")
[flag🤫]
```