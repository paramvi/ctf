# problem reqs
when you login to machine

```
passcode@pwnable:~$ ls -l 

total 16
-r--r----- 1 root passcode_pwn   48 Jun 26  2014 flag
-r-xr-sr-x 1 root passcode_pwn 7485 Jun 26  2014 passcode
-rw-r--r-- 1 root root          858 Jun 26  2014 passcode.c

```

passcode.c

```
#include <stdio.h>  
#include <stdlib.h> 

void login(){
        int passcode1;
        int passcode2;

        printf("enter passcode1 : ");
        scanf("%d", passcode1);
        fflush(stdin);

        // ha! mommy told me that 32bit is vulnerable to bruteforcing :)
        printf("enter passcode2 : ");
        scanf("%d", passcode2);

        printf("checking...\n");
        if(passcode1==338150 && passcode2==13371337){
                printf("Login OK!\n");
                system("/bin/cat flag");
        }
        else{
                printf("Login Failed!\n");
                exit(0);
        }
}
void welcome(){
        char name[100];
        printf("enter you name : ");
        scanf("%100s", name);
        printf("Welcome %s!\n", name);
}

int main(){
        printf("Toddler's Secure Login System 1.0 beta.\n");

        welcome();  
        login();

        // something after login...
        printf("Now I can safely trust you that you have credential :)\n");
        return 0;   
}


```

# thought process
clearly scanf is at fault here. Like `&` is missing and when you try to put integer to it, program crasshes. So we would have to find another way to put "junk" values in passcode1 and passcode2.

We see that in the welcome function `char[100] name` asks for the input. It gives me the hunch that I could explore the buffer overflow route.

Let's explore welcome function with gdb
```
(gdb) disas welcome
Dump of assembler code for function welcome:
   0x08048609 <+0>:     push   ebp
   0x0804860a <+1>:     mov    ebp,esp
   0x0804860c <+3>:     sub    esp,0x88
   0x08048612 <+9>:     mov    eax,gs:0x14
   0x08048618 <+15>:    mov    DWORD PTR [ebp-0xc],eax
   0x0804861b <+18>:    xor    eax,eax
   0x0804861d <+20>:    mov    eax,0x80487cb
   0x08048622 <+25>:    mov    DWORD PTR [esp],eax
   0x08048625 <+28>:    call   0x8048420 <printf@plt>
   0x0804862a <+33>:    mov    eax,0x80487dd
   0x0804862f <+38>:    lea    edx,[ebp-0x70]
   0x08048632 <+41>:    mov    DWORD PTR [esp+0x4],edx
   0x08048636 <+45>:    mov    DWORD PTR [esp],eax
   0x08048639 <+48>:    call   0x80484a0 <__isoc99_scanf@plt>
   0x0804863e <+53>:    mov    eax,0x80487e3
   0x08048643 <+58>:    lea    edx,[ebp-0x70]
   0x08048646 <+61>:    mov    DWORD PTR [esp+0x4],edx
   0x0804864a <+65>:    mov    DWORD PTR [esp],eax
   0x0804864d <+68>:    call   0x8048420 <printf@plt>
   0x08048652 <+73>:    mov    eax,DWORD PTR [ebp-0xc]

```
We see `lea    edx,[ebp-0x70]`. This should be `char[100]`. Cool!
Let's disassembly login function
```
(gdb) disas login
Dump of assembler code for function login:
   0x08048564 <+0>:     push   ebp
   0x08048565 <+1>:     mov    ebp,esp
   0x08048567 <+3>:     sub    esp,0x28
   0x0804856a <+6>:     mov    eax,0x8048770
   0x0804856f <+11>:    mov    DWORD PTR [esp],eax
   0x08048572 <+14>:    call   0x8048420 <printf@plt>
   0x08048577 <+19>:    mov    eax,0x8048783
   0x0804857c <+24>:    mov    edx,DWORD PTR [ebp-0x10]
   0x0804857f <+27>:    mov    DWORD PTR [esp+0x4],edx
   0x08048583 <+31>:    mov    DWORD PTR [esp],eax
   0x08048586 <+34>:    call   0x80484a0 <__isoc99_scanf@plt>
   0x0804858b <+39>:    mov    eax,ds:0x804a02c
   0x08048590 <+44>:    mov    DWORD PTR [esp],eax
   0x08048593 <+47>:    call   0x8048430 <fflush@plt>
   0x08048598 <+52>:    mov    eax,0x8048786
   0x0804859d <+57>:    mov    DWORD PTR [esp],eax
   0x080485a0 <+60>:    call   0x8048420 <printf@plt>
   0x080485a5 <+65>:    mov    eax,0x8048783
   0x080485aa <+70>:    mov    edx,DWORD PTR [ebp-0xc]
   0x080485ad <+73>:    mov    DWORD PTR [esp+0x4],edx
   0x080485b1 <+77>:    mov    DWORD PTR [esp],eax
   0x080485b4 <+80>:    call   0x80484a0 <__isoc99_scanf@plt>
   0x080485b9 <+85>:    mov    DWORD PTR [esp],0x8048799
   0x080485c0 <+92>:    call   0x8048450 <puts@plt>
   0x080485c5 <+97>:    cmp    DWORD PTR [ebp-0x10],0x528e6
   0x080485cc <+104>:   jne    0x80485f1 <login+141>
   0x080485ce <+106>:   cmp    DWORD PTR [ebp-0xc],0xcc07c9

```
here I have shown only the relevant part. Notice the following lines

```
0x080485c5 <+97>:    cmp    DWORD PTR [ebp-0x10],0x528e6
0x080485cc <+104>:   jne    0x80485f1 <login+141>
0x080485ce <+106>:   cmp    DWORD PTR [ebp-0xc],0xcc07c9
```
338150 in hex is 0x528E6. Required value of passcode1
13371337 in hex is 0xcc07c9. Required value of passcode2
Beautiful!

So next thing that comes in mind is can we set the both values in login function using welcome function?
Let's see `char[100]` started from `epb-0x70` and passcode1 is at `ebp-0x10`.

if we check the difference between both of these 
```
(gdb) p/d 0x70-0x10
$1 = 96

```
It's 96. 96 bytes. sweet. So seems like I can control the value of passcode1. But how do I control the value of passcode2??
One point to note here is that in the original code `scanf("%100s", name);` only accepts 100 bytes only.
hmmmm....

Here I was out of my depths. So after researching on the internet, I found about GOT hijacking. GOT is global offset table. So basically when a C program is compiled into a binary, not all the functions are resolved. Especially the functions in shared libraries like printf, system(...), scanf() etc. It's linkers responsiblity to find them and once it finds them, then it places there address in the GOT. It works more like on-demand feature. When a function is required, linker will fetch it and place it's address in GOT and then if this function is required again, it'll be present there.
If you didn't understand, what I am talking about I encourage you to read about it on wikipedia and other tons of articles and come back.
Cool!

How do we exploit this?
If you note the scanf function of login function, we `&` is missing 

# solution
