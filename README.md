<!---
{
  "id": "279a01d6-7696-46b7-9cb7-2c44773ad06b",
  "teaches": "From `printf` to `fprintf`",
  "depends_on": ["a2596a91-c7de-477a-bfbb-b08867f1aa89"],
  "author": "Stephan Bökelmann",
  "first_used": "2025-04-01",
  "keywords": ["fprintf", "streams", "C", "output"]
}
--->

# From `printf` to `fprintf`

## 1) Introduction
In a [previous exercise](https://github.com/STEMgraph/a2596a91-c7de-477a-bfbb-b08867f1aa89) we looked into the C-function `printf`, which gave us the ability to print constant strings into the terminals output. 
Furthermore the output-buffer has also been discussed in an [earlier exercise](github.com/STEMgraph/missing).
We have to understand, that `printf` as well as `fprintf` are functions, that are not directly implemented by the C' standard library, but are rather adapters to functionalities of the hosting operating system. This is due to the fact, that our program itself only knows its own RAM. Every thing outside of this, has to be managed by the operating system.

So when we want to write to the terminal' output from within our program, we'd have to prepare a string, hand it over to the operating system and hope for it to forward it to the terminals output buffer. The following piece of code shows an example implementation of how this can be achieved:

```asm
01 section .data
02       msg     db 'Hello, World!', 10
03       len     equ $ - msg
04
05 section .text
06       global _start
07
08 _start:
09      mov     rax, 1          ; syscall number for write
10      mov     rdi, 1          ; file descriptor 1 = stdout
11      mov     rsi, msg        ; pointer to message
12      mov     rdx, len        ; message length
13      syscall
14
15      ; exit(0)
16      mov     rax, 60         ; syscall number for exit
17      xor     rdi, rdi        ; exit code 0
18      syscall
```

From line 01-03, a string is prepared and its length is memorized.
From 09-13 the actual operation happens. 
First the operations number for `write to a location` is moved into `rax`. After that, the location-identifier also called `file descriptor` is moved to `rdi`. Then the address of the first character of our string in RAM is moved into `rsi` and the amount of bytes which shall be copied into the `file descriptor` is moved to `rdx`. 
When the program executes `syscall`, the operating system knows that it's supposed to take over and look into `rax`, `rdi`, `rsi` and `rdx` for instructions. 

As you can see in this example, the `file descriptor` that we are using is `1`. In the **POSIX**-standard, the operating system opens three standard `file descriptors` for each process automatically:

| File Descriptor   |   Symbolic Name    |   Explaination   |
|--------------------|--------------------|-----------------|
| `0` | `stdin` | Standard in, usually the keyboard-buffer|
| `1` | `stdout`| Standard out, usually the terminal-emulators output-buffer|
| `2` | `stderr`| Standard error, usually also connected to the terminals output buffer|

---

Let's now look into an example, where we want to open a different `file descriptor` to a different location:

```asm
01  section .data
02      filename    db '/home/user/file.txt', 0
03      message     db 'Hello, World!', 10
04      msg_len     equ $ - message
05  
06  section .text
07      global _start
08  
09  _start:
10      ; syscall: open(filename, O_WRONLY | O_CREAT | O_TRUNC, 0644)
11      mov     rax, 2              ; syscall: open
12      lea     rdi, [rel filename] ; const char *filename
13      mov     rsi, 577            ; flags: O_WRONLY | O_CREAT | O_TRUNC
14      mov     rdx, 0644           ; mode: rw-r--r-- (octal)
15      syscall
16      mov     r12, rax            ; save returned fd
17  
18      ; syscall: write(fd, message, msg_len)
19      mov     rax, 1              ; syscall: write
20      mov     rdi, r12            ; file descriptor
21      lea     rsi, [rel message]  ; pointer to message
22      mov     rdx, msg_len        ; length
23      syscall
24  
25      ; syscall: close(fd)
26      mov     rax, 3              ; syscall: close
27      mov     rdi, r12
28      syscall
29  
30      ; exit(0)
31      mov     rax, 60
32      xor     rdi, rdi
33      syscall
```

Firstly, we are storing the filename in line 02. From 11-15 we set up our `syscall` to open a new `file descriptor` with flags and permissions. 
`O_WRONLY`, `O_CREAT`, and `O_TRUNC` are macros, that expand to bitpatterns. Namely `0b00000001`, `0b01000000`, and `0b1000000000`. The `|` between them stands for the boolean `OR` operation, which creates a new bitpattern:

```
 0b0000000001
+0b0001000000
+0b1000000000
-------------
 0b1001000001
```

This tells the operating system, that we want to open the file with the following functions:

|  macro  | pattern  | function |
|--------|----------|-----------|
| `O_WRONLY` | `0b0000000001` | Write and Read Only Mode |
| `O_CREAT` | `0b0001000000` | Create new file, if it doesn't exist |
| `O_TRUNC` | `0b1000000000` | If it exists, shorten it to 0 byte first |

Line 14 sets the POSIX permissions on the new file, in this case: 
|Kategorie |	Bedeutung |	Oktalwert |
|------|-----|------|
|Owner	| read + write	| 6 |
|Group	| read	| 4 |
|Others |	read	| 4 |

After the `syscall` in line 15 is executed, the operating system will write an identifier to this stream which was now opened into `rax`. That's why `rax` is moved into the work-register `r12` in line 16. That way, we can use the descriptor when we need to write to it. 

---

So while `printf` writes to `stdout` per default, we can use `fprintf` to write to any other file-location, that we can access via our operating system. 

## 1) Introduction

In a [previous exercise](https://github.com/STEMgraph/a2596a91-c7de-477a-bfbb-b08867f1aa89), we explored the C function `printf`, which allows us to print constant strings to the terminal's output. Additionally, the concept of the output buffer was discussed in an [earlier exercise](https://github.com/STEMgraph/missing).

It’s important to understand that both `printf` and `fprintf` are **not** implemented directly by the C standard library in the sense of standalone functionality. Instead, they are **convenience adapters** that rely on services provided by the underlying operating system. This is because a user-space program only has direct access to its own memory (RAM); anything beyond that—like files, terminals, or devices—must be accessed via the operating system.

So, if we want to write to the terminal output from within our program, we must prepare a string in memory, request the operating system to forward it to the terminal, and hope that it gets buffered and rendered appropriately. The following code demonstrates how this can be done using the `write` system call directly in Assembly:

```asm
01  section .data
02      msg     db 'Hello, World!', 10
03      len     equ $ - msg
04
05  section .text
06      global _start
07
08  _start:
09      mov     rax, 1          ; syscall number for write
10      mov     rdi, 1          ; file descriptor 1 = stdout
11      mov     rsi, msg        ; pointer to message
12      mov     rdx, len        ; message length
13      syscall
14
15      ; exit(0)
16      mov     rax, 60         ; syscall number for exit
17      xor     rdi, rdi        ; exit code 0
18      syscall
```

- In lines 01–03, a string is prepared in memory and its length is calculated.
- In lines 09–13, the actual write operation takes place:
  - Line 09 loads `1` into `rax`, which tells the kernel that we want to invoke the `write` system call.
  - Line 10 sets `rdi` to `1`, which corresponds to the file descriptor for `stdout`.
  - Line 11 sets `rsi` to the address of the message.
  - Line 12 specifies the number of bytes to write via `rdx`.

When `syscall` is executed, control is passed to the kernel, which looks at `rax`, `rdi`, `rsi`, and `rdx` to decide what to do.

In this example, we're writing to **file descriptor 1**, which is predefined by the **POSIX** standard. Every process launched in a POSIX-compliant system receives three default file descriptors:

| File Descriptor | Symbolic Name | Description |
|-----------------|----------------|-------------|
| `0`             | `stdin`        | Standard input (usually the keyboard buffer) |
| `1`             | `stdout`       | Standard output (usually the terminal's output buffer) |
| `2`             | `stderr`       | Standard error (usually also directed to the terminal) |

---

Now let’s look at an example where we want to **open and write to a file** using a file descriptor different from the standard ones:

```asm
01  section .data
02      filename    db '/home/user/file.txt', 0
03      message     db 'Hello, World!', 10
04      msg_len     equ $ - message
05  
06  section .text
07      global _start
08  
09  _start:
10      ; syscall: open(filename, O_WRONLY | O_CREAT | O_TRUNC, 0644)
11      mov     rax, 2              ; syscall: open
12      lea     rdi, [rel filename] ; const char *filename
13      mov     rsi, 577            ; flags: O_WRONLY | O_CREAT | O_TRUNC
14      mov     rdx, 0644           ; mode: rw-r--r-- (octal)
15      syscall
16      mov     r12, rax            ; save returned fd
17  
18      ; syscall: write(fd, message, msg_len)
19      mov     rax, 1              ; syscall: write
20      mov     rdi, r12            ; file descriptor from open()
21      lea     rsi, [rel message]  ; pointer to message
22      mov     rdx, msg_len        ; length
23      syscall
24  
25      ; syscall: close(fd)
26      mov     rax, 3              ; syscall: close
27      mov     rdi, r12
28      syscall
29  
30      ; exit(0)
31      mov     rax, 60
32      xor     rdi, rdi
33      syscall
```

Line 02 prepares the filename string (null-terminated). Lines 11–15 set up the `open` syscall with flags and permissions:
  - `O_WRONLY`, `O_CREAT`, and `O_TRUNC` are macros defined in `<fcntl.h>`, expanded to bit patterns:
    - `O_WRONLY` → `0b0000000001`
    - `O_CREAT`  → `0b0001000000`
    - `O_TRUNC`  → `0b1000000000`
  - Bitwise OR combines them into:

```
    0b0000000001
  | 0b0001000000
  | 0b1000000000
  --------------
    0b1001000001  → 577 (decimal)
```

  This combination tells the OS: "Open the file in write-only mode. If it doesn't exist, create it. If it exists, truncate it."

- Line 14 sets the **POSIX file permissions**:

  | Category | Meaning        | Octal |
  |----------|----------------|--------|
  | Owner    | read + write   | 6      |
  | Group    | read           | 4      |
  | Others   | read           | 4      |

- After the `open` syscall (line 15), the returned file descriptor is stored in `r12` (line 16) for later reuse.
- The write and close operations (lines 18–28) follow the same syscall logic as before but use the new file descriptor.

---

While `printf` writes to `stdout` by default, `fprintf` allows us to target any stream—such as a file, a network socket, or a pipe—provided we have an appropriate file descriptor. What you're seeing in Assembly is the **lowest level version** of what `fprintf` does internally: preparing registers and invoking the right system call.

 
### 1.1) Further Readings and Other Sources
- [Operating Systems: Three Easy Pieces](https://pages.cs.wisc.edu/~remzi/OSTEP/) — A foundational (and free!) book that covers file I/O, system calls, and the role of the OS in managing processes and resources.
- *The Linux Programming Interface* by Michael Kerrisk — The definitive book on Linux system calls, including detailed coverage of `open`, `read`, `write`, and file descriptor mechanics.
- *Programming from the Ground Up* by Jonathan Bartlett — A beginner-friendly introduction to Assembly and how it interfaces with Linux system calls like `write`, `open`, and `exit`.


## 2) Tasks
1. **Look into fprintf**: Navigate into the `./assets/` subdirectory. Inspect the file `fprintf.c`. Compile and run it.
2. **Exchange with printf**: Open `./assets/fprintf.c` again and exchange `fprintf` with `printf`. Compile and run it again. 
3. **Into a new File**: Open the `./assets/to_file.c`-file. Inspect the code carefully, compile and run it. 
4. **Compare to ASM**: Look at the second `asm` example in this exercise and compare it to the C-code carefully.

## 3) Questions

1. What is the difference between `printf` and `fprintf`, and why might you use one over the other?

2. What are file descriptors `0`, `1`, and `2` in POSIX systems, and what streams do they correspond to?

3. How does the `write` system call work at the register level, and what values must be placed in which registers before calling `syscall`?

4. What do the flags `O_WRONLY`, `O_CREAT`, and `O_TRUNC` mean, and how are they combined when opening a file?

5. How does `fprintf` internally relate to system calls like `write` — what layers exist between user code and the kernel?


## 4) Advice

Understanding how high-level functions like `printf` and `fprintf` translate down to system-level calls such as `write` is a powerful step toward mastering systems programming. Don't just memorize function signatures — explore what happens under the hood. Use tools like `strace`, `xxd`, and even your own minimal Assembly code to connect abstract programming concepts to real machine behavior.

The more you understand how data flows from your code to actual hardware or system components, the more control and confidence you'll have as a developer. Keep experimenting, keep digging, and don’t be afraid to break things in the process — that’s where the real learning happens.
