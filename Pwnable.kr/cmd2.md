First I needed to get the cmd1 flag again so I did that quickly and got started
```bash
cmd1@ubuntu:~$ ./cmd1 "/bin/cat f*"
[flag🤫]
```

I opened the c file `cmd2.c`
```c
#include <stdio.h>
#include <string.h>

int filter(char* cmd){
        int r=0;
        r += strstr(cmd, "=")!=0;
        r += strstr(cmd, "PATH")!=0;
        r += strstr(cmd, "export")!=0;
        r += strstr(cmd, "/")!=0;
        r += strstr(cmd, "`")!=0;
        r += strstr(cmd, "flag")!=0;
        return r;
}

extern char** environ;
void delete_env(){
        char** p;
        for(p=environ; *p; p++) memset(*p, 0, strlen(*p));
}

int main(int argc, char* argv[], char** envp){
        delete_env();
        putenv("PATH=/no_command_execution_until_you_become_a_hacker");
        if(filter(argv[1])) return 0;
        printf("%s\n", argv[1]);
        setregid(getegid(), getegid());
        system( argv[1] );
        return 0;
}
```

reading the c code in the server I saw that I needed to access the flag after the PATH variable was discarded...
```c
putenv("PATH=/no_command_execution_until_you_become_a_hacker");
```

I also could not use any of these in my commands: =, PATH, export, /, \`, flag
so my problem was to get to the commands.
My first thought was to create an alias for the command that I wanted.
```bash
cmd2@ubuntu:~$ alias docs='/bin/cat flag'
cmd2@ubuntu:~$ ./cmd2 docs
docs
sh: 1: docs: not found
```
But the alias had a scope of the current terminal session and was not recognized by the system.
I tried creating a global alias but I did not have permission...
