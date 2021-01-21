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

Nothin right? but we put more that the size, hum weird, let try to put more
```
$ ./a.out
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
[3]    3087 segmentation fault (core dumped)  ./a.out

```

This time we have a segmentation fault, which mean we try to access memory that we not suppose to.
Basically we override the instruction pointer and try to read somewhere we shouln't. We will dig into it later.

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
When you 


```c
int c = 42;
```
This instruction will push the variable on the stack
The stack go down in addr, meaning it begin at 0xffffffff.

In our example the stack will go until 0xff

| addr  | value |
|-------|-------|
|  0xfb |  42   |
