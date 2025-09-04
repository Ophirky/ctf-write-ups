This challenge is supposed to be easier because all it takes is to put input into a program. But let's see.
When reading the code in the server I saw that this challenge included 5 steps. each step contained a different type of program input.
1. argv
2. stdio
3. env
4. file
5. network

Let's look at every step alone
## **Level #1**
```c
// argv
if(argc != 100) return 0;
if(strcmp(argv['A'],"\x00")) return 0;
if(strcmp(argv['B'],"\x20\x0a\x0d")) return 0;
printf("Stage 1 clear!\n");
```

So I needed to make my argv have 100 arguments where the 65th argument was `\x00` and the 66 argument was `\x20\x0a\x0d` (just converted chars to ints).

First run with first payload
```bash
input2@ubuntu:~$ ./input2 $(python2 -c "print b'\x00 '*64 + b'\x20\x0a\x0d' + b' A' * 35")
-bash: warning: command substitution: ignored null byte in input
Welcome to pwnable.kr
Let's see if you know how to give input to program
Just give me correct inputs then you will get the flag :)
```

Then I looked at it for a little and realized that it compared bytes and not strings...

Second try:
```bash
input2@ubuntu:~$ ./input2 $(python3 -c "print('\\x00 '*64 + '\\x20\\x0a\\x0d' + ' A' * 35)")
-bash: warning: command substitution: ignored null byte in input
Welcome to pwnable.kr
Let's see if you know how to give input to program
Just give me correct inputs then you will get the flag :)
```

Now I tried just printing it as it is and seeing if the bytes were printed as a string or not.
```bash
input2@ubuntu:~$ python3 -c "print('\\x00 '*64 + '\\x20\\x0a\\x0d' + ' A' * 35)"

 A A A A A A A A A A A A A A A A A A A A A A A A A A A A A A A A A A A
```

The answer was no so I tried also escaping the `x` 
```bash
input2@ubuntu:~$ ./input2 $(python2 -c "print '\\\x00 ' * 64 + '\\\x20\\\x0a\\\x0d' + ' A' * 33")
Welcome to pwnable.kr
Let's see if you know how to give input to program
Just give me correct inputs then you will get the flag :)
```

Now I needed to see where it failed. so I copied the program to my local and added `printf`'s in some places

Ok. I have made a very dumb mistake... I forgot that when there is a substring then `strcmp` returns 0 and not 1... so now everything I have done was for nothing.
Also my indexes were wrong. Anyway I had to fix the entire payload.

Level Test Code:
```c
#include <stdio.h>
#include <string.h>

int main(int argc, char const *argv[])
{
    if(argc != 100) {
        printf("Fell in sub-level #1 argc=%d\n", argc);
        return 0;
    }
    if(strcmp(argv['A'],"\x00")) {
        printf("Fell in sub-level #2 argv['A'] = %s\n", argv['A']);
        return 0;
    }
    if(strcmp(argv['B'],"\x20\x0a\x0d")) {
        printf("Fell in sub-level #3 argv['B'] = %s\n", argv['B']);
        return 0;
    }
    printf("Stage 1 clear!\n");
    return 0;
}
```

So first I needed to fix my payload to actually be 100. or 99 to be more accurate. and my Index for 'A' was incorrect.
```bash
./tmp $(python3 -c "print('123 ' * 65 + '\\\x20\\\x0a\\\x0d' + ' A' * 33)")
```

Now I needed 'A' to be `\x00` which is automatically added to strings.
I commented out the first and third sub-levels so that I could focus on passing this one.
```c
// Current level Code
if(strcmp(argv[1],"\x00")) {
	printf("Fell in sub-level #2 argv[1] = %s, strcmp(argv[1],\"\\x00\" == %d)\n", argv[1], strcmp(argv[1],"\x00"));
	return 0;

}
```

First Run
```bash
user@pc:~$ ./tmp $(python3 -c "import sys; sys.stdout.buffer.write(b'A')")
Fell in sub-level #2 argv[1] = A, strcmp(argv[1],"\x00" == 65)
```

I have also tried by reading from a file and that also did not work.

I wanted a way to control the way I send the arguments and what is sent to that command. To do this I researched and found the `xargs` command that will send the arguments how I want them to be.
The xargs command just echos the input as it is to the appendix command.

So I decided to give it a try
```bash
user@pc:~$ printf '\x00' | xargs ./tmp
xargs: WARNING: a NUL character occurred in the input.  It cannot be passed through in the argument list.  Did you mean to use the --null option?
argc = 2
Stage 1 clear!
```

now I needed to make it at a larger scale. When writing the command I saw that it wouldn't work because the last input started with `\x20` which is a space...

So I went back and saw the warning from xargs and was interested
`xargs: WARNING: a NUL character occurred in the input.  It cannot be passed through in the argument list.  Did you mean to use the --null option?`

the `--null` option makes the the null byte the seperator and not the space key. so since the arguments in argv are strings they all end in \x00 and maybe giving it nothing will still work. so I modified the command to include this and tried it...
```bash
ophirky@opc:/mnt/d/CTFS$ python3 -c "import sys; sys.stdout.buffer.write(b'\x00'*65 + b'\x20\x0a\x0d' + b'\x00'*34)" | xargs --null ./tmp
Stage 1 clear!
```

And this worked perfectly!

Non Local run:
```bash
input2@ubuntu:~$ python3 -c "import sys; sys.stdout.buffer.write(b'\x00'*65 + b'\x20\x0a\x0d' + b'\x00'*34)" | xargs --null ./input2
Welcome to pwnable.kr
Let's see if you know how to give input to program
Just give me correct inputs then you will get the flag :)
Stage 1 clear!
```

So the command for stage 1 was this:
```bash
python3 -c "import sys; sys.stdout.buffer.write(b'\x00'*65 + b'\x20\x0a\x0d' + b'\x00'*34)" | xargs --null ./tmp
```

## **Level #2**
This level was about stdio, so again I copied to my local machine and added a bunch of printfs...

Level Code:
```c
// stdio
char buf[4];
read(0, buf, 4);
if(memcmp(buf, "\x00\x0a\x00\xff", 4)) return 0;
read(2, buf, 4);
if(memcmp(buf, "\x00\x0a\x02\xff", 4)) return 0;
printf("Stage 2 clear!\n");
```

My Code:
```c
// stdio
char buf[4];

read(0, buf, 4); // STDIN
printf("sub-level#1 -> buff = [%x, %x, %x, %x]\n", buf[0], buf[1], buf[2], buf[3]);

if(memcmp(buf, "\x00\x0a\x00\xff", 4)) return 0;
printf("cleared sub-level #1!!!\n");

read(2, buf, 4); // stderr
printf("sub-level#2 -> buff = [%x, %x, %x, %x]\n", buf[0], buf[1], buf[2], buf[3]);

if(memcmp(buf, "\x00\x0a\x02\xff", 4)) return 0;
printf("Stage 2 clear!\n");
```

Clearing the first sublevel was really easy:
```bash
user@pc:~$ ./tmp < <(python3 -c "import sys; sys.stdout.buffer.write(b'\x00\x0a\x00\xff')")
sub-level#1 -> buff = [0, a, 0, ffffffff]
cleared sub-level #1!!!

sub-level#2 -> buff = [a, a, 0, ffffffff]
```

for the second level every input I gave it in the stdin was the input sent to it
```bash
user@pc:~$ ./tmp < <(python3 -c "import sys; sys.stdout.buffer.write(b'\x00\x0a\x00\xff')")
sub-level#1 -> buff = [0, a, 0, ffffffff]
cleared sub-level #1!!!
AAAA
sub-level#2 -> buff = [41, 41, 41, 41]
```

So I tried just typing it in:
```bash
user@pc:~$ ./tmp < <(python3 -c "import sys; sys.stdout.buffer.write(b'\x00\x0a\x00\xff\n\x00\x0a\x02\xff')")
sub-level#1 -> buff = [0, a, 0, ffffffff]
cleared sub-level #1!!!

sub-level#2 -> buff = [a, a, 0, ffffffff]
```

Since that did not work I read about `stderr` but from what I've read there is not really a difference in the way that they receive input since they are all just file handles...
The reason it was not working was because the python was writing to `stdout` and then that was being sent to `stdin` so maybe if I send it to `stderr` then I'll be able to write to it directly.

I isolated the sub-level in the code and tried the following command:
```bash
./tmp < <(python3 -c "import sys; sys.stderr.buffer.write(b'\x00\x0a\x02\xff')")
```

Trials...
```bash
user@pc:/mnt/d/CTFS$ ./tmp < <(python3 -c "import sys; sys.stderr.buffer.write(b'\x00\x0a\x02\xff')")

�
sub-level#2 -> buff = [a, 7f, 0, 0]
user@pc:/mnt/d/CTFS$ ./tmp < <(python3 -c "import sys; sys.stderr.buffer.write(b'\x00\x0a\x02\xff\n')")

�

sub-level#2 -> buff = [a, 7f, 0, 0]
```

I remembered that using `< <` created a file with the python output. once the file was created the program would read it as stdin (fd 0) so I wanted to se e if I could redirect to the stderr (fd 2). I remembered that when reading about stderr someone did this `2>` to write to it so I tried doing the opposite to write to it.

```bash
user@pc:~$ ./tmp 2< <(python3 -c "import sys; sys.stdout.buffer.write(b'\x00\x0a\x02\xff\n')")
sub-level#2 -> buff = [0, a, 2, ffffffff]
Stage 2 clear!
```

Now I had to combine the two inputs and level two would be done.

Even though I did not think that it would work I still tried just putting the two inputs together:
```bash
$ ./tmp < <(python3 -c "import sys; sys.stdout.buffer.write(b'\x00\x0a\x00\xff\n')") 2< <(python3 -c "import sys; sys.stdout.buffer.write(b'\x00\x0a\x02\
xff\n')")
sub-level#1 -> buff = [0, a, 0, ffffffff]
cleared sub-level #1!!!
sub-level#2 -> buff = [0, a, 2, ffffffff]
Stage 2 clear!
```
But it did!!

***STDIN:***
`< <(python3 -c "import sys; sys.stdout.buffer.write(b'\x00\x0a\x00\xff\n')")`

***STDERR:***
`2< <(python3 -c "import sys; sys.stdout.buffer.write(b'\x00\x0a\x02\xff\n')`

## **Level #3**
Level 3 was all about the `env` - environment.

Level Code:
```c
if(strcmp("\xca\xfe\xba\xbe", getenv("\xde\xad\xbe\xef"))) return 0;
printf("Stage 3 clear!\n");
```

So I needed to create an env variable that was named `\xde\xad\xbe\xef` and contained `"\xca\xfe\xba\xbe"`.

## Stopping The challenge
This CTF consisted of following a pre-defined path rather than requiring independent research, which made the experience less engaging and was not what. I was looking for.

Maybe I'll come back to it in the future...