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
