# forbidden documents, pwn

in this challenge we are given a ip and a port, if you connect with netcat you get a prompt for opening a file and for the reading offset and the size.

![alt text](https://raw.githubusercontent.com/quantumbracket/ctf_writeups/master/xmasctf2018/forbidden%20documents/zz.png)

obviouslly you cant read the flag directly because the program will exit if you have "flag" in your filename. we dont know the name of the executable either, I tried 'chall' and 'server' but none worked. but we do know the name of one file:

![alt text](https://raw.githubusercontent.com/quantumbracket/ctf_writeups/master/xmasctf2018/I%20want%20that%20toy/iwtt_8.png)

This screenshot was taken from my 'I want that toy' writeup. As we can see, there is a file called 'redir.sh', this file is in all challenges. Its used to keep the program listening to a port by using socat. But we can leak the executable name by opening this file  

![alt text](https://raw.githubusercontent.com/quantumbracket/ctf_writeups/master/xmasctf2018/forbidden%20documents/zz1.png)

Now we know that the name of the executable is 'random_exe_name' so we just have to open it and read it. I wrote a simple script that reads 4000 bytes at a time (tcp cannot handle more than 4096 bytes at a time) and like that it dumps the file.  

but even after dumping a considerable amount of the program, still the data section and some offsets where corrupted. After reversing part of the corrupted data I realized the file was opened in 'r' mode. in some systems you have to open your file in 'rb' mode if you want to read your original data, else all '\n' will get replaced with '\r\n'(edit: socat might have actually caused this). so what I just needed to do is to  replace all '\r\n' with '\n'. Now the binary is not corrupted,time to start reversing!!!

![alt text](https://raw.githubusercontent.com/quantumbracket/ctf_writeups/master/xmasctf2018/forbidden%20documents/zz2.png)

As we can see in the main function, it asks for the filename, then it checks if it contains 'flag' or 'stdin' and then asks you for your offset and then for the length of your file, and everyhing gets stored in a buffer (s) which is 512 bytes long. Clearly a buffer overflow. to exploit this I used the same approach as with the 'I want that toy': make a rop chain that prints a got address and then returns to main, then overflow again (and the rhymes come out of my brain ;) ) but this time with the libc gadgets. so I did it like this:  

rop gadgets:  
0x00000000004014f3 : pop rdi ; ret  
0x00000000004014f1 : pop rsi ; pop r15 ; ret  

one_gadget:
constraints:
  [rsp+0x30] == NULL  
  
leak rop chain:  
pop rdi -> "flag" -> puts  (this will print flag and a newline, I do this in order to separate the leaked address with the other output)  
pop rdi -> puts.got -> puts -> main

(overflow again and send this other chain)


one_gadget -> '\x00'*0x40 (for [rsp+0x30] == NULL constraint)  

if I test it locally it works perfectly and I get code execution, but when I tried with the server it closes the connection just after the memory leak. apparently it doesnt return to main. 

This made me struggle. but then I realized that in the redir.sh the socat uses a pty option which according to Wikipedia is:  

>In some operating systems, including Unix, a pseudoterminal, pseudotty, or PTY is a pair of pseudo-devices, one of which, the slave, emulates a hardware text terminal device, the other of which, the master, provides the means by which a terminal emulator process controls the slave.

So its like we are not writing our payload from a socket but rather like from our keyboard. that means that if we write '\x03' its like hitting Control-C. I was looking for resources about control characters but I didnt find any valid escape character (according to wikipedia https://en.wikipedia.org/wiki/C0_and_C1_control_codes#C0_(ASCII_and_derivatives) it should be '\x10'(Data Link Escape) but in this case doesnt work). Eventualy I discovered by experimenting with it a bit that '\x16' escapes the next character(like \\) and that the special characters where 1,2,3,4,8,13,17,18,19,21,22,23,26,27,28 and 127. so I wrote a function that escapes them.  

Now the next problem was the libc. Even though you could see the libc version by opening /proc/self/maps, I wasn't shure which libc was the right one (there are like 3 or 4 libc for the same version). I dont want to download twice after discovering one was not the correct one. So I decided to dump the whole libc using the script I wrote for dumping the program.  

Now once obtained the libc, edit our exploit with the correct offsets,then we just have to get a shell:

![alt text](https://raw.githubusercontent.com/quantumbracket/ctf_writeups/master/xmasctf2018/forbidden%20documents/zz3.png)

