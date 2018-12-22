# I want that toy, pwn
First we get a binary and a libc, when reversing its clear that the program binds to a port, forks and then the parent closes
the fd, but the child serves the request and the respond parses and responds to the client. 


![alt text](https://raw.githubusercontent.com/quantumbracket/ctf_writeups/master/xmasctf2018/I%20want%20that%20toy/iwtt_1.png)

![alt text](https://github.com/quantumbracket/ctf_writeups/raw/master/xmasctf2018/I%20want%20that%20toy/iwtt_2.png)

if we run the server and access it in our browser we get something like this:

![alt text](https://github.com/quantumbracket/ctf_writeups/raw/master/xmasctf2018/I%20want%20that%20toy/iwtt_3.png)

lets encode hello world as base64 and send it trough the toy parameter:

![alt text](https://github.com/quantumbracket/ctf_writeups/raw/master/xmasctf2018/I%20want%20that%20toy/iwtt_4.png)


also the server logs some stuff:

![alt text](https://github.com/quantumbracket/ctf_writeups/raw/master/xmasctf2018/I%20want%20that%20toy/iwtt_5.png)

Now lets try a huge string to check if there is any overflow:

![alt text](https://github.com/quantumbracket/ctf_writeups/raw/master/xmasctf2018/I%20want%20that%20toy/iwtt_6.png)


Ok, there is a stack smashing detected, we are getting somewhere. Apparently the base64decode function just decodes into the buffer with no checks,its technically a strcpy ,now how do we leak the stack cookie?.  
one slow approach would be to try every byte of the cookie because our input its not null terminated and also because the cookie its the same between forks.  but there is an easier way, in the route() function this one responds with all the html, but remember it prints our user agent? Well there is a format string vulnerability there, so we can leak a cookie and the program base(remember its PIE compiled)

![alt text](https://github.com/quantumbracket/ctf_writeups/raw/master/xmasctf2018/I%20want%20that%20toy/iwtt_7.png)
(stderr and stdout are dup2'ed to the socket fd before the route() function is called)

so now with $p and %7$p wa can leak the program base and the stack cookie, but there is no libc leak,what do we do now? easy, just 
make a rop chain that calls puts with some got address and then returns back to respond, this way we get the leak and have the
opportunity to craft a new rop payload this time with libc gadgets. so our leak ropchain first looks like this:

gadgets:  
0x0000000000001d9b : pop rdi ; ret  
0x0000000000001d99 : pop rsi ; pop r15 ; ret  

rop chain:
pop rdi -> 4 -> pop rsi; pop r15 -> 1 -> dummy -> dup2  
pop rdi -> puts.got -> puts  
pop rdi -> 0 -> fflush  
pop rdi -> 4(our socket fd is always 4) -> respond  

or in a simplified way:  
dup2(4,1)  
puts(puts.got)  
fflush(0)  
respond(4)  

dup2 is used because parse_query_string() is called before the dup2() and the route(), so our stdout will still point to stdout and not  to our socket, also fflush is used to make sure puts prints the string, respond takes as parameter our socket number (its always 4)  


now we have to overflow again but this time with our rop shellcode:  

one gadget rce:  
0x3f306	execve("/bin/sh", rsp+0x30, environ)  
constraints:  
  rax == NULL  
    
rop chain:  
pop rdi -> 4 -> pop rsi; pop r15 -> 3 -> dummy -> dup2  
pop rdi -> 4 -> pop rsi; pop r15 -> 0 -> dummy -> dup2 (this will return 0(stdin) in rax so its just the constraint we need for the one_gadget) 
one_gadget  

or in a more simplified way:  
dup2(4,3)  
dup2(4,0)  
one_gadget  

now we execute it and get the flag:  

  
![alt text](https://github.com/quantumbracket/ctf_writeups/raw/master/xmasctf2018/I%20want%20that%20toy/iwtt_8.png)


this task took me like 3 days to complete, this was my reaction when I got the flag:  
https://www.youtube.com/watch?v=VZSvVkJZ8xI


