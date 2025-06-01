---
layout: post
title: "Currying C functions using a heap-based trampoline"
---
Suppose you have a function and an argument that you want to send off to a worker thread.
The canonical way to do this in C is via `pthread_create()`.

```
$ man pthread_create
SYNOPSIS

       #include <pthread.h>

       int pthread_create(pthread_t *restrict thread,
                          const pthread_attr_t *restrict attr,
                          void *(*start_routine)(void *),
                          void *restrict arg);

```
However, it only accepts a single void pointer as the function argument `arg`.
What if we have multiple arguments?

A common way to get around this limitation is to define a struct with all your arguments and cast it to `void*` before passing it in.
Then, to unwrap them in your thread, cast the void pointer back to a struct pointer and retrieve its members.

But wouldn't it be nice to inline the arguments into our function so that we don't need an `arg` parameter?
This would eliminate the need for the whole struct-pointer-casting dance.
It might also make room for compiler optimizations.

In functional languages, we do this by currying.
In Haskell, for example, we can define the following function of type `Int -> Int -> Int` (read: a function that takes in an integer and returns another function that takes in an integer and finally returns an integer).

```haskell
addTwoNums :: Int -> Int -> Int
addTwoNums a b = a + b
```

To curry it, we simply supply a single argument and get back a new function of type `Int -> Int`:
```haskell
addSeven :: Int -> Int
addSeven a = addTwoNums 7 a
```

There is, however, no established way to do this in C.
The best I could come up with is a nasty trick (ab)using [GCC's support for nested functions](https://gcc.gnu.org/onlinedocs/gccint/Trampolines.html#Support-for-Nested-Functions).

## Constructing a trampoline
In a [blog post](https://nullprogram.com/blog/2019/11/15/), Chris Wellons discusses using an executable stack to create closures, allowing for nested functions to access the variables in their containing function. (The purpose of the post was not actually to glamorize it, but to shed light on how difficult it is to turn off the feature completely, as it is a security vulnerability.) The tl;dr is to store the function and the supplied argument on the stack, then jump to it via the `jmp` instruction. Hence, a trampoline.

Executable stacks are, however, disabled on the system I am on. Furthermore, I wanted to _return_ the new function pointer rather than just calling it in the containing function, so I need the code to persist even after the stack frame is freed. Moving the trampoline onto the heap solves both these problems.

To recap, the goal is to recreate the Haskell example from earlier. Namely, a function that takes in another function with signature `Int -> Int -> Int` and an integer, and returns a function with signature `Int -> Int`. In C function pointer notation, that looks like this:
```c
int (*curry(int (*f)(int, int), int arg))(int);
```
Then, we should be able to do the following to create function `add7()`:
```c
int add_two_nums(int a, int b) {
    return a + b;
}
int main() {
    int (*add7)(int) = curry(&add_two_nums, 7);
    assert(add7(10) == 17);
}
```
Following the blog post, to create our trampoline, we just need to create some executable code with the function pointer, store the argument from the containing function, and jump to the address of our code. The only difference is instead of allocating the trampoline on the stack, we allocate it on the heap via a memory mapping which we then mark as executable with `mprotect()`. 

```c
int (*curry(int (*f)(int, int), int arg))(int) {
  unsigned long fp = (unsigned long)f;
  unsigned char code[] = {
    0x48, 0xc7, 0xc6, arg >> 0, arg >> 8, arg >> 16, arg >> 24,    // mov rsi, arg
    0x48, 0xb8, fp >> 0, fp >> 8, fp >> 16, fp >> 24,              // mov rax, fp
    fp >> 32, fp >> 40, fp >> 48, fp >> 56,
    0xff, 0xe0                                                     // jmp rax
  };
  uint8_t *ptr = mmap(0, 1024, PROT_READ | PROT_WRITE,
      MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
  if (ptr == (void*)-1) {
    perror("mmap");
    exit(1);
  }
  memcpy(ptr, code, sizeof(code));
  mprotect(ptr, sizeof(code), PROT_READ | PROT_EXEC);
  return (int(*)(int))ptr;
}
```
A few things to point out here. First, remember that the function `f` already takes an argument, which goes in `rdi` as the first argument, so `arg`, the argument from the containing function, goes into `rsi` as the second argument, according to the [System V ABI](https://en.wikipedia.org/wiki/X86_calling_conventions#System_V_AMD64_ABI).

Secondly, the encoding is system-specific. Since x86-64 is little-endian, we first store the last (least significant) byte.
To decode it, then, you would do something like
```c
uint32_t arg = code[3] << 0 | code[4] << 8 | code[5] << 16 | code[6] << 24;
```
and similarly for the unsigned long function pointer.

Lastly, as a caveat, you probably shouldn't do any of this in real code, as heap allocation is slow and executing memory directly likely opens up security vulnerabilities. But alas, I think it was a fun exercise to explore what computers are capable of.
