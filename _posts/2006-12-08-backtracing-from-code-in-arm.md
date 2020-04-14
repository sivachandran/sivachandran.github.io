---
layout: post
title: "Backtracing in ARM through code"
date: 2006-12-08 17:31:00 +0530
tags: [embedded, c-cpp]
---
Recently I found a way to do backtrace from code in ARM processor. On a normal processor glibc itself provides a function called `backtrace` using which we can do backtracing, resolving symbol names and etc. But my glibc (ver 2.95.3) doesn’t support this, so I thought of writing my own backtrace function and the result is this.

The following two pictures shows the structure of activation record in ARM and how it is maintained

![Call Stack](/assets/arm-call-stack.jpg)

![Activation Stack](/assets/arm-activation-record.jpg)

So from the above pictures you could have found that the activation records are maintained like a linked list. All we need to find is the start of linked list and travel till there is no more record.

Glibc provides a built in macro `__builtin_frame_address` using which we can find the current frame(activation record) address at any time. So the steps for backtracing are

* Find current frame address (i.e.` __builtin_frame_address`)
* The first four bytes(i.e. `*fp`) of frame points to program counter (PC), so skip it
* The next four bytes(i.e. `*(fp – 1)`) since stack grow downwards) is link register or return address of current function
  * Using this we can find the parent/caller
* The fourth four bytes(i.e. `*(fp – 3)`) points to the previous/caller activation record
  * This forms a linked list of activation records and end when `*(fp – 3)` is `0`

```c
#include <stdio.h>;
void backtrace(void* fp)
{
  if (fp == 0)
    return;

  fprintf (stderr, "%p\n", *((int*)fp - 1));
  backtrace ((void*)(*((int*)fp - 3)));
}

int bar()
{
  printf ("bar\n");
  printf ("*** backtrack ***\n");
  backtrace (__builtin_frame_address (0));
  printf ("\n");

  return 0;
}

int foo()
{
  printf ("foo\n");
  printf ("*** backtrack ***\n");
  backtrace (__builtin_frame_address (0));
  printf ("\n");

  return bar();
}

int main()
{
  printf ("main at %p\n", main);
  printf ("foo at %p\n", foo);
  printf ("bar at %p\n\n", foo);

  printf ("main\n");
  printf ("*** backtrack ***\n");
  backtrace (__builtin_frame_address (0));
  printf ("\n");
 
  return foo();
}
```

```bash
Output:
main at 0x919a1160
foo at 0x919a1114
bar at 0x919a1114

main
*** backtrack ***
0x919a13e0
0x919a1418
0x919a1064

foo
*** backtrack ***
0x919a11b8
0x919a13e0
0x919a1418
0x919a1064

bar
*** backtrack ***
0x919a1144
0x919a11b8
0x919a13e0
0x919a1418
0x919a1064
```