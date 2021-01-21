### Introduction 

Buffer overflow allow you to write memory you shouldn't.
Often it's cause because a programming error.


#### Overflow
Ex: 

```c
int main() {
  char buf[8];
  gets(buf)
  return 0;
}
```

A simple programm that read from stdin and put it into buf.
But here we do not verify the length. What happens if the input is more than 16 bytes?
Let's find out

```
gcc main.c -fno-stack-protector -no-pie -Wl,-z,relro,-z,now,-z,noexecstack 
$ ./a.out
1234567890
```

Nothing right? but we put more that the size, hum weird, let try to put more
```
$ ./a.out
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
[3]    3087 segmentation fault (core dumped)  ./a.out

```

This time we have a segmentation fault, which mean we try to access memory that we not suppose to.
Basically we override the instruction pointer, called eip for 32bits and rip on 64bits, and try to read somewhere we shouln't. We will dig into it later.

First let's play with that.

```c
#include <unistd.h>
#include <sys/types.h>
#include <stdlib.h>
#include <stdio.h>
     
int main()
{
 
  int var;
  int i = 42;
  char buf[8];
 
  fgets(buf);
  printf ("i = %d\n");
  return 0;
}
```

Let's try this
```
$ ./a.out
aaaa
i = 42
```

Now let's break it.
```
$ a.out
1111111111
check : 12593
```

We can see that doing a buffer overflow allow us to modify the stack.
Let dig a bit on what happens for the stack

#### Stack 

You need to know that the stack is a last in first out aka LIFO
When you add something it go on top and you remove something it's the one on top that will be removed.
The instruction to do it are `push` and `pop`.
`push` will add something, and `pop` will remove it


The following example will simplify some mechanism but should give a big picture of what happens
Value will be define in a readable way 

```c
int c = 42;
```
This instruction will push the variable on the stack
The stack go down in addr, meaning it begin at 0xffffffff, and then decrease, everytime we push something.

In our example the stack will have a size of 100 bytes, and depending on the architecture an integer take 4 bytes in memory.
So decreasing `100` by 4 will be `96`

| addr  | value |
|-------|-------|
|  96   |  42   |

The next instruction is :
```
char buf[8];
```
This time we will push on the stack an array of 8 bytes,
The stack will now be like:


| addr  | value |
|-------|-------|
|  88   |  0    |
|  96   |  42   |


Then the gets function come, and will write stuff in the buf varible, so on the stack, it will take the length give as input by the user since no verification have been done by the user. 
Let's imagine that user just enter `1111`
So get will write them into buf


Now retry this but with `1111111111`
This time we will have something like

| addr  | value |
|-------|-------|
|  88   |  1    |
|  89   |  1    |
|  90   |  1    |
|  91   |  1    |
|  92   |  1    |
|  93   |  1    |
|  94   |  1    |
|  95   |  1    |
|  96   |  1    |

So the variable at `96` have been overrided.
Wait what? my program wasn't printing `1` for `c`
Indeed, because a integer `int` is on 4 bytes and register works with hexadecimal values so when interpreting the `1` caractere it doesn't see it as the numeric number like we do.
It have it's own traduction table, called ASCII.
try: 
```
man ascii
```

#### Hex little endian and big endian

Why not changing the value of `c` but give it the value we want.
let's try with `1`.
We need to override the last 4 bytes to set them to the value hex of `1`
`1` is `0x01` so let's try this.
Since some caracteres aren't printable, if you check the `ASCII` table you already know.
A simple python script will help us.
So first we need to fills the `buf` variable and then the `c` 4 bytes.
`1` will be `0x00000001` 

```
python -c 'print "A" * 8 + "\x00\x00\x00\x01"' | ./a.out
c = 16777216
```

Wait what !!

We should check this one in hex, `0x1000000`, but we put `0x00000001` why is it the opposite?
Remember the title? big endian and little endian ? This is the order of how the low-order/high-order byte is store.
We can see the in the stack the low-order byte is store at the starting address, this is called little endian.
We, human, are more used to the big endian with the high byte at the staging adress.
So let's arrange our bytes to make it work

```
python -c 'print "A" * 8 + "\x01\x00\x00\x00"' | ./a.out
c = 1

```
