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
