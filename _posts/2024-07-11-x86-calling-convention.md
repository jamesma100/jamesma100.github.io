---
layout: post
title: "Overview of the x86 call stack"
---

I had mostly forgotten how the x86 call stack works, and many things continue to confuse me, so join me and my scattered thoughts as I attempt to clarify my understanding.

In this post, we will look at a simple assembly program that takes in an integer and calculates the sum of all integers from 1 to `n`, then exits with the result as the return code. In essence:

```
int sum_to(int n) {
    int sum = 0;
    int i = n;
    while (i > 0) {
        sum += i;
        i--;
    }
    return sum;
}

int main() {
    return sum_to(10);
}
```
We will use the [NASM](https://www.nasm.us/) assembler running on standard 64-bit processor.

## The stack
Contrary to popular belief, the stack grows _downwards_[[^1]]. So, growing the stack would mean subtracting the stack pointer by the size of the value's type. Newly pushed items will reside at a larger memory address.

Below is an example of a 1Mb stack starting at 0x100000.
```
------------------  0x100000 (high address, bottom of stack)
| stack frame 1  |
|----------------| <- rbp              
| stack frame 2  |
|----------------| <- rsp
|                |
| empty          |
|                |
------------------  0x000000 (low address, top of stack)
```

There are two designated registers to keep track of the current stack frame - `rbp` points to the bottom and `rsp` points to the top.
`rbp` grows and shrinks as functions are called and returned, while `rsp` grows and shrinks _during_ a function call as local variables are created and freed. Thus, it should always be the case that `rsp <= rbp`.

Note that registers starting with "r" are simply extended 64-bit versions of their smaller counterparts. For instance, `rax` is an extended version of the 32-bit `eax`, which itself can be further segmented.

## Caller duties
Our assembly program will be separated into two phases, the caller phase and the callee phase.
The caller is responsible for setting up the stack with arguments, and the callee is responsible for taking over the stack, executing its function, then restoring the stack to its original state.

Below is the procedure for the caller.
```
push 10
call add
add rsp, 8
mov rdi, rax
mov rax 60
syscall
```

First, we push 10 to pass it as an argument to `add`, which extends the stack pointer by 8.
After `add` returns, we restore it by shrinking `esp` by 8.
Notice to keep our stack functional, each action has a counterpart that reverts it.
This can be seen more clearly if we replace `push` with its equivalent:
```
sub rsp, 8
mov qword[rsp], rax
```

Below is what our stack looks like after setting up the single argument.
```
--------------------- 0x100 <- rbp
| 10                |                
--------------------- 0xf8  <- rsp
```

Finally, to prepare for the system call, we need [60](https://chromium.googlesource.com/chromiumos/docs/+/master/constants/syscalls.md) for the exit syscall in `rax` and its only argument in `rdi`.


## Callee duties
When the function is called, the callee takes over the stack from the caller.
Since the caller expects that the stack remains the same, whatever we do to the stack we need to undo.

We start by pushing `rbp`. Remember that this is the value of the _caller_'s base pointer. But since we will be using it to keep track of the callee's frame, we need to save its old value before we can set it.
Registers that we _cannot_ modify unless we save its original value to be restored later are called "callee-saved" or "call-preserved[[^2]]."

Another such register is `rsp`, but its value after the function returns is just the value of `rbp`, or the base pointer of the function's stack frame. Thus, it does not need to be explicitly pushed onto the stack.

Next, we need to allocate local variables on the stack - in our case, just the `sum`.
We can achieve this by subtracting 8 from the stack pointer to make room for a 8 byte integer[[^3]].
```
push rbp
mov rbp, rsp
sub rsp, 8
```

Our stack now looks like this:
```
--------------------- 0x100
| 10                |                
--------------------- 0xf8
| return address    |
--------------------- 0xf0
| saved rbp (0x100) |
--------------------- 0xe8 <- rbp
|                   |
--------------------- 0xe0 <- rsp
```

Finally, we do some prep for our function to execute.
```
mov rdi, [rbp+16]          ; move parameter `n` into rdi
mov rsi, 1                 ; initialise loop counter `i`
mov qword[rbp-8], 0        ; initialise local var `count`
```

The prelude is now done, so we just have our function left to implement, which should be straightforward to understand.
```
loop
    add [rbp-8], rdi  ; sum += i
    dec rdi           ; i--
    jnz loop          ; if sum != 0; then restart loop
mov rax, [rbp-8]
```

Now that we're done executing the function, we need to restore the callee-saved `rbp` and `rsp`. Then we can return.
Remember earlier when we noted that the value of `rsp` after the function returns is just the base pointer of the function's stack frame.
So all we need to do to restore `rsp` is a single `mov` instruction instead of the standard `push`/`pop` duo.
```
mov rsp, rbp
pop rbp
ret
```

Our stack now looks identical to what it was before the callee took over!

```
--------------------- 0x100
| 10                |                
--------------------- 0xf8
```

Now, the whole program:

```
global _start

section .text

add:
    push rbp
    mov rbp, rsp
    sub rsp, 8            ; space for local var
    mov rdi, [rbp+16]     ; store arg 1
    mov rsi, 1            ; initialize loop counter
    mov qword[rbp-8], 0   ; store current sum in local var
    loop:
        add [rbp-8], rdi
        dec rdi
        jnz loop
    mov rax, [rbp-8]
    mov rsp, rbp          ; deallocates local vars
    pop rbp               ; restore caller's base ptr
    ret

_start:
    push 10
    call add
    add rsp, 8             ; deallocate params
    mov rdi, rax           ; exit code is return val
    mov rax, 60            ; exit syscall
    syscall
```
You can compile, link, and run the whole thing with `nasm -felf64 sum_to.asm && ld sum_to.o -o sum_to && ./sum_to`.

And that is it! Keep in mind that calling conventions are implemented differently from compiler to compiler, but the general principles are more or less the same.

---
[^1]: Some textbooks draw the stack in reverse, such that the stack can be seen to grow upwards. But this is confusing in its own right, since lower memory addresses would now be above higher memory addresses and futher obfuscates how memory is actually laid out. In this article, we followed Intel's own [reference manual](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html), which describes the stack as seen above.

[^2]: From the x86-64 System V ABI, callee-saved registers include `r12` to `r15`, `rbx`, `rsp`, and `rbp`, meaning they are preserved across function calls. The rest are caller-saved. However, a register being caller-saved does not imply that the caller _must_ save it. It simply means the caller cannot expect the value to be preserved across function calls. For this reason, the terms "call-preserved" and "call-clobbered" are often used in place of "callee-saved" and "caller-saved." 
[^3]: It's important to ensure this buffer space we create is as large as the data we intend to fill it with - in this case we only allocate 8 bytes because we know our `sum` variable will not exceed it (`qword`, or quad-word, means 8 bytes). If not, the local variable can end up overwriting data above the buffer. For instance, a user input that isn't bounds-checked can be carefully constructed to overflow and rewrite the return address so that when the function returns, the CPU jumps to the specified address in the payload instead of the expected next instruction.
