---
layout: post
title: "Dynamic linking with lazy binding"
---
I recently had to debug a dynamic linking issue. This took me down a rabbit hole exploring how dynamic linkers work - in particular, how a dynamic linker resolves an external function at runtime.
How it does this is rather peculiar and interesting, so I decided to write down some of the details.

## Dynamic linking basics
When your program has a call to an external function in a shared library, for example `printf()`, the linker needs to somehow resolve the address of that function, since it isn't contained locally.
There are two ways it can accomplish this - statically or dynamically.
 - Static linking does symbol resolution at compile time, copying all library routines into your executable.
This essentially ensures your executable is portable and does not depend on finding libraries on the host system during runtime.
However, this wastes a lot of disk space since each program includes an entire copy of every dependency it needs.
Imagine if every hello world program on your computer contains an entire copy of libc!
Additionally, when a shared library updates, all your programs have to be re-linked against the new version.
- Dynamic linking, as its name suggests, accomplishes this dynamically at runtime.
At compile time, the linker only inserts some basic information, such as relocation and symbol table information.
The rest of the linking is done at runtime.
This has the advantage of allowing multiple programs to share the same copy of a shared library by mapping it into their own address space.

## Lazy binding
At first glance, this seems simple - just resolve all shared libraries the program needs when it is loaded.
One obvious downside is that you end up loading entire shared libraries when most programs will only ever call a few functions.

The solution is a clever approach called lazy binding - a function is only resolved the first time it is called.
This allows us to avoid a bunch of unnecessary relocations at load time.

Lazy binding involves two structures known as the GOT (Global Offset Table) and the PLT (Procedure Linkage Table).
The GOT is just a table of addresses.
The first couple entries store some basic information the dynamic linker needs to resolve functions.
After that, there is an entry for every external function that we call, e.g. `printf()`.
Below is what a GOT might look like:
```
GOT[0]: address of .dynamic
GOT[1]: address of relocation entries
GOT[2]: address of entrypoint for dynamic linker
GOT[3]: address of printf()
...
```

The PLT, on the other hand, is a table containing sets of instructions.
The first entry contains a special set of instructions that jumps to the dynamic linker.
Each entry after that specifies instructions for resolving an external function.
```
PLT[0] (dynamic linker)
   push   0x2fca(%rip)
   jmp    *0x2fcc(%rip)
   nopl   0x0(%rax)

PLT[1] (routine for printf)
   jmp    *0x2fca(%rip)
   push   $0x0
   jmp    0x555555555020

...
```
The tl;dr is that each time an external function is called, the CPU will jump to its PLT entry, which will jump to its corresponding GOT entry to find an address.
For example, when we call `printf()`, we will jump to PLT[1], which will jump to GOT[3].
The first time the function is called, GOT[3] will jump to the next instruction of PLT[1], allowing PLT[1] to complete resolution of the function address.
The linker will then modify GOT[3] to contain the actual resolved address.
The next time the function is called, the PLT[1] will again jump to GOT[3] but GOT[3] will now have the address, thus skipping the lookup process.
This means there is a runtime cost the first time the function is called, but subsequent invocations will be fast.

Let's see how this actually works.
Note that the following is specific to Linux x86_64 and ELF - if you don't have this environment you can follow along on <https://www.onlinegdb.com>.

Consider the following program that calls `printf()`, a dynamically linked function.
```c
#include <stdio.h>

int main()
{
    printf("Hello, world!\n"); // slow
    printf("Hello, world!\n"); // fast
    return 0;
}
```

Let's start gdb and break on main.
```
$ gdb main
(gdb) break main
Breakpoint 1 at 0x113d: file main.c, line 5.
(gdb) run
Starting program: /home/a.out

Breakpoint 1, main () at main.c:5
5           printf("Hello, world!\n");
```

Disassembling the main function, we see
```
(gdb) disas main
Dump of assembler code for function main:
   0x0000555555555139 <+0>:     push   %rbp
   0x000055555555513a <+1>:     mov    %rsp,%rbp
=> 0x000055555555513d <+4>:     lea    0xec0(%rip),%rax        # 0x555555556004
   0x0000555555555144 <+11>:    mov    %rax,%rdi
   0x0000555555555147 <+14>:    call   0x555555555030 <puts@plt>
   0x000055555555514c <+19>:    lea    0xeb1(%rip),%rax        # 0x555555556004
   0x0000555555555153 <+26>:    mov    %rax,%rdi
   0x0000555555555156 <+29>:    call   0x555555555030 <puts@plt>
   0x000055555555515b <+34>:    mov    $0x0,%eax
   0x0000555555555160 <+39>:    pop    %rbp
   0x0000555555555161 <+40>:    ret
End of assembler dump.
```
First, the compiler noticed that we are printing a constant string so replaced `printf()` with `puts()`.
Next, we see that instead of calling `puts()` directly, we call into `puts@plt`.
Let's examine this routine:
```
(gdb) disas 'puts@plt'
Dump of assembler code for function puts@plt:
   0x0000555555555030 <+0>:     jmp    *0x2fca(%rip)        # 0x555555558000 <puts@got.plt>
   0x0000555555555036 <+6>:     push   $0x0
   0x000055555555503b <+11>:    jmp    0x555555555020
End of assembler dump.
```
The first line shows an indirect jump to `0x555555558000`, the entry for `puts()` in the `got.plt` section, which is the part of the GOT that we care about.
Let's see what this address contains.

```
(gdb) x/gx 0x555555558000
0x555555558000 <puts@got.plt>:  0x0000555555555036
```
`0x0000555555555036` is simply the next instruction in our PLT!
So calling `printf()` led us to the first instruction in our PLT entry, which jumped to a GOT entry, which led us back to the second instruction in our PLT entry.

Moving on, we push `$0x0` onto the stack and jump to `0x555555555020`.
```
(gdb) x/4i 0x555555555020
   0x555555555020:      push   0x2fca(%rip)        # 0x555555557ff0
   0x555555555026:      jmp    *0x2fcc(%rip)        # 0x555555557ff8
   0x55555555502c:      nopl   0x0(%rax)
   0x555555555030 <puts@plt>:
    jmp    *0x2fca(%rip)        # 0x555555558000 <puts@got.plt>
```
This is actually PLT\[0\], which is responsible for calling the dynamic linker.
We push another argument (`0x555555557ff0`) onto the stack and do an indirect jump to `0x555555557ff8`.
If we trace this jump, we arrive at the dynamic linker resolve routine.
```
(gdb) x/gx 0x555555557ff8
0x555555557ff8: 0x00007ffff7fd8eb0
(gdb) x/gx 0x00007ffff7fd8eb0
0x7ffff7fd8eb0 <_dl_runtime_resolve_xsavec>:    0xe3894853fa1e0ff3
```
`dl_runtime_resolve_xsavec` will then use the two stack entries to resolve the actual address of `puts()`.

This seems like a lot of work just to resolve a single function address, but fortunately we only have to do this one time.
Let's step past the first `printf` statement:
```
(gdb) n
Hello, world!
6           printf("Hello, world!\n");
```

Once again, we'll disassemble the `puts@plt` entry.
```
(gdb) disas 'puts@plt'
Dump of assembler code for function puts@plt:
   0x0000555555555030 <+0>:     jmp    *0x2fca(%rip)        # 0x555555558000 <puts@got.plt>
   0x0000555555555036 <+6>:     push   $0x0
   0x000055555555503b <+11>:    jmp    0x555555555020
End of assembler dump.
```
Just like before, the PLT entry points to a GOT entry.
Examining it, we see

```
(gdb) x/gx 0x555555558000
0x555555558000 <puts@got.plt>:  0x00007ffff6f09910
```
This is different from what we got before!
Recall that the last time around, it was pointing to the second instruction in our PLT entry, a.k.a the jump instruction stored at `0x0000555555555036`.
But now it directly points to the function address!
So the after the dynamic linker resolved the address, it modifies the GOT entry to point to the resolved address of `puts()`.

Let's verify by trying to call it.

The `puts()` function takes in a `const char*` and has return type of void, so we can cast the address to a function pointer of type `void(*)(const char*)` and try to call it.
```
(gdb) call ((void(*)(const char*))0x00007ffff6f09910)("Hello, world!\n")
Hello, world!
```
Success! We've found the address of `puts()` through the GOT and successfully called it.

This whole process seems quite intricate but is a rather neat little trick to ensure we only attempt to resolve a function when it is called and that we only do so the first time.
I hope this helped demystify some of the magic dynamic linkers do behind the scenes.

## References
- Computer Systems A Programmer's Perspective 3rd Edition---Randal Bryant & David Hallaron
- [GOT and PLT for pwning](https://systemoverlord.com/2017/03/19/got-and-plt-for-pwning.html)
- [Global Offset Table (GOT) and Procedure Linkage Table (PLT) - bin 0x12](https://www.youtube.com/watch?v=kUk5pw4w0h4)

