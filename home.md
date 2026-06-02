# Ret2libc Exploitation Technique on Kali Linux via Buffer Overflow in a vulnerable C program


## Introduction

In this report, we are going to analyze the return to libc attack technique, which we saw as a code reuse circumvention idea to the W^X mitigation technique.
This exploitation allows an attacker that identified potentially useful functions and data in the standard C libc library, to execute arbitrary functions overwriting the return address on the stack.
Our demonstration is going to be done by using a virtual machine running Kali Linux x86_64 and a precisely crafted C program, which uses the now deprecated gets() function to cause a stack overflow. The objective is to obtain an interactive shell (/bin/sh) abusing only code present in the running process, without any code injection.
We are going to be executing the attack in ideal conditions for an attacker, all mitigations, such as ASLR, stack canary and PIE are going to be disabled. We will then observe how the "normal" scenario with all the mitigations active prevent the attack, unless a proper exploit is developed.

## Environment & Setup

The exploit was developed and tested using the following:

- **Operating System**: Kali GNU/Linux, version 2026.1
- **Architecture**: x86_64
- **OS configuration**: ASLR mitigation disabled via
```
  echo 0 | sudo tee /proc/sys/kernel/randomize_va_space
```
- **Compiler**: `gcc` with the following security mitigations disabled:
  - `-fno-stack-protector` — disables stack canary
  - `-z execstack` — marks the stack as executable
  - `-no-pie` — disables Position Independent Executable

## Utilized Tools

- **GDB + pwndbg** — used for debugging and memory analysis
  ([github.com/pwndbg/pwndbg](https://github.com/pwndbg/pwndbg))
- **pwntools** — A python library for exploit development
  ([docs.pwntools.com](https://docs.pwntools.com/))
- **ROPgadget** — used to search for ROP gadgets in binaries
  ([github.com/JonathanSalwan/ROPgadget](https://github.com/JonathanSalwan/ROPgadget))
- **checksec** — used to inspect binary security mitigations

## The C program

The target of our exploit is a small C program which was pupusefully designed to cause stack-based buffer overflow. The vulnerability lies in the use of `gets()`, a depreciated unsafe function which unlike its "safer" alternatives (`fgets`, `read`), that can still be overflowed, `gets()` accepts input until a newline or EOF is encountered not limiting the number of characters read, which can lead to buffer overflows.

If we give this function more bytes than the buffer can store, the extra bytes spill into the stack and overwrite the saved return address — the value the CPU uses to know where to jump when the function ends. If we control that address, we control where the program goes.

### Source Code

```c
#include <stdio.h>

extern char *gets(char *s);

void read_message() {
    char buffer[80];
    printf("Enter message: ");
    gets(buffer);
    printf("Message received: %s\n", buffer);
}

int main() {
    read_message();
    return 0;
}
```

> The explicit `extern` declaration of `gets()` is required because modern glibc headers no longer expose the prototype, and compiling without it would result in an `implicit-function-declaration` error.

## Exploit


