---
layout: post
title: "Interpreter vs JIT: a practical illustration"
---

It's commonly known that there are three main ways to execute code - you can compile it, interpret it, or use a JIT (just-in-time execution).
The difference between compiling and interpreting/JIT'ing is easy to grasp - only the former has separate compilation and execution steps, while the latter does both in a single step, where each line is compiled and executed directly.
However, the difference between interpreters and JITs is a bit more confusing.
Wikipedia's definition of JIT'ing is:
> compilation (of computer code) during execution of a program (at run time) rather than before execution.

whereas an interpreter is defined as:
> a computer program that directly executes instructions written in a programming or scripting language, without requiring them previously to have been compiled into a machine language program.

It isn't clear to me, from the definitions alone, how the two differ, so I thought I would try to clarify the distinction between the two by looking at how they would be implemented in practice.
You can find the source code for a basic interpreter I wrote [here](https://github.com/jamesma100/bfint).

But first, we need to take a detour and talk about Brainfuck.

### Background: Brainfuck
Brainfuck is a super simple (though Turing complete!) programming language that consists of just 8 instructions, making it a popular language to build compilers and interpreters for.
The instructions are:
- `>` - move instruction pointer to the right by one
- `<` - move instruction pointer to the left by one
- `+` - increment (by one) current byte
- `-` - decrement (by one) current byte
- `.` - output current byte
- `,` - accept input from the user, store it in current byte
- `[` - if current byte is zero, move the instruction pointer forward to the matching `]`
- `]` - if current byte is nonzero, move the instruction pointer backward to the matching `[`

A couple things to note here:
- the instruction pointer is pointer that loops through each instruction in the code, moving forward by one every time (unless a `[` or `]` is encountered), and initially points to the beginning of some memory segment, which stores our data.
- the "current byte" is the data stored at the location the instruction pointer is currently pointing to.

### The easy way: an interpreter
With the basics out of the way, let's look at how we would execute Brainfuck code with an interpreter.

Interpreting source code is a pretty straightforward concept.
You simply write an interpreter program that loops through the input source code and execute each line _as part of the original interpreter program_.
As we will see later on, this is the key distinction between an interpreter and a JIT.
Let's see how we would interpret some Brainfuck code.

To start, we need two main data structures: one to represent the program's memory, and another to represent the instructions we are executing.

```
typedef struct {
  int *head;              // beginning of memory segment
  int *data;              // data in our memory
  int *ptr;               // pointer to next byte to be executed
  size_t initial_size;    // space allocated for memory segment
} Memory;

typedef struct {
  int num_rows;           // number of rows in source code
  int num_cols;           // number of columns in source code
  char **data;            // the instructions themselves
  int linenum;            // current line number (row)
  int pos;                // current position w/i `linenum` (column)
} Instructions;
```

Note that we're using a 2d array to represent the instructions for easier access and parsing.
So the instruction in row `i` and column `j` can be retrieved with `instructions->data[i][j]`.
Next, we initialize them as follows:
```
void init_memory(Memory *mem, size_t initial_size) {
  mem->data = (int*)malloc(initial_size * sizeof(int));
  mem->head = mem->data;
  mem->ptr = mem->data;
  mem->initial_size = initial_size;
  for (int i = 0; i < initial_size; ++i) {
    mem->data[i] = 0;
  }
}

void init_instructions(Instructions *instructions, int num_rows, int num_cols) {
  instructions->num_rows = num_rows;
  instructions->num_cols = num_cols;
  char **data = (char**)malloc(sizeof(char*) * num_rows);
  for (int i = 0; i < num_rows; ++i) {
    data[i] = (char*)malloc(sizeof(char) * num_cols);
  }
  instructions->data = data;
  instructions->linenum = 0;
  instructions->pos = 0;
  for (int i = 0; i < num_rows; ++i) {
    for (int j = 0; j < num_cols; ++j) {
  	instructions->data[i][j] = '\0';
    }
  }
}

Instructions instructions;
init_instructions(&instructions, 100, 100);
Memory mem;
init_memory(&mem, 100);
```

Note that C, unlike Java for example, does not initialize arrays, so we have to explicitly set each element to `'\0'` to avoid an invalid memory acccess.

Now, to interpret the source code stored in some initialized `Instructions` struct, we simply loop through each character, each of which is an instruction, then execute it.


To check each instruction, we might do something like this:
```
switch(instruction) {
  case '<':
    left(&mem);
    pos++;
    break;
  ...
  case '+':
    inc(&mem);
    pos++;
    break;
  ...
}
```
where `mem` is some initialized `Memory` struct.
Notice that for each instruction, we just map it to an equivalent C function and call it.
For instance, here is the `inc` instruction referenced above:
```
void inc(Memory *mem) {
  (*mem->ptr)++;
}
```
As noted before, the key point about an interpreter is that the CPU never leaves the original program stored on disk.

### Time to JIT
How would a JIT differ in this scenario?
The crucial difference between the interpreter we looked at and a JIT is that a JIT 1. _compiles_ each instruction into machine code, 2._emits_ the resulting binary into some dynamically allocated memory, then 3._executes_ that memory directly.
Notice that we are no longer contained within the original on-disk binary, as with the interpreter case.
We are instead creating a chunk of executable memory, then executing that memory directly as a separate program.

To make this more concrete, let's look at how we might implement a JIT variant of the same `inc()` function we saw earlier, which we will call `inc_jit()`.
We will need to allocate some executable memory (we'll use `mmap` to map a page), copy the machine code for `inc()` into the mapped page, then execute that memory directly.
```
typedef int (*incFuncPtr)(int); // function pointer typedef that
                                // takes an int and returns an int

void inc_jit(Memory *mem) {
  // allocate executable memory
  size_t len = 1024;
  void* exec_mem = mmap(0, len, PROT_READ | PROT_WRITE | PROT_EXEC,
  MAP_ANON | MAP_PRIVATE, -1, 0);

  if (exec_mem == MAP_FAILED) {
    fprintf(stderr, "error: mmap failed. exiting");
    exit(1);
  }

  // emit machine code into the memory page we allocated
  unsigned char code[] = {
    0x48, 0x89, 0xf8,                   // mov %rdi, %rax
    0x48, 0x83, 0xc0, 0x01,             // add $1, %rax
    0xc3                                // ret
  };
  memcpy(exec_mem, code, sizeof(code));

  // cast in-memory binary into function pointer
  incFuncPtr inc_func = exec_mem;

  // execute the function
  *mem->ptr = inc_func(*mem->ptr);
}
```
All this work for a single add instruction!
Reminder that our original `inc()` function is only:
```
void inc(Memory *mem) {
  (*mem->ptr)++;
}
```

### A note on security
Note that allocating executable memory pages opens us up to potential security vulnerabilities.
This is because data in this region can now be interpreted as valid computer instructions, i.e. the `eip` register can now point to some location in this allocated page.
If someone is able to write to this page, the CPU can be made to [execute arbitrary code](https://en.wikipedia.org/wiki/Heap_spraying).
For this reason, it's important for memory to never be [both writable and executable](https://en.wikipedia.org/wiki/W%5EX) at the same time.
So instead of doing what we did above, the safe approach would be to first do the memory-mapping with the writable bit set. Then, when our machine code is copied into the page, atomically give it executable permission while taking away its writable permission.

### Intermediate representations
Contrary to our example, JITs typically do not directly compile source code to machine code.
Often, it first compiles source code down to an [intermediate representation](https://en.wikipedia.org/wiki/Intermediate_representation), which can be further compiled to machine code at a _much_ faster speed.
For instance, JVM languages are compiled down to an intermediate representation known as bytecode, which is then compiled by the JVM into machine code.
Furthermore, LLVM is centered around the idea of a _common_ intermediate representation, which allows you to separate the frontend and the backend of a compiler and develop each individually.

However, an intermediate representation can also colloquially refer to any format that optimizes the original source code and makes compiling it simpler or faster.
For instance, the sequence of `>>>>>` can be transformed into `>5` (move instruction pointer right by five), which lets us simply do `add $5, %rax` instead of `add $1, %rax` five times.
Similarly, we can have a [preprocessing phase](https://github.com/jamesma100/bfint/blob/main/src/preprocess.c#L38) that parses through the source code and maps all `[`'s to `]`s (and vice versa), providing information such as line number and position within each line.
Now, when when jumping from one bracket to another, we can simply use the location of the first to look up the location of the second, then move the instruction pointer there directly.


