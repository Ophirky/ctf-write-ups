I decided to try the `crackmes.one` platform and learn more high level things and have challenges that are on windows and not only Linux like pwnable.kr.
Keygenme was the first challenge that came up so I decided to go ahead and try it.

My first big challenge was actually to understand whether the password to the zip file was part of the challenge or I should've known it. I researched a little bit and no... it was not part of the challenge.

After I figured out the password to the file (which was crackmes.one) I had a file called `keygenme.exe` .

First thing I opened it up and saw what was the program. The program contained a simple login page that required a username and registration key.

At first I just started to tinker with it and see if anything worked
 - I sent an empty form - did nothing
 - Tried using the zip file name as the registration key - did nothing
 - Delete placeholders and submit - did nothing

So now I decided to disassemble it and see what I can find. To disassemble it i needed to learn how to use IDA and only then I will be able to go back.