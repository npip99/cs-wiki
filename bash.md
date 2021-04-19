## Bash

### Introduction

First, if you haven't already, see `linux-basics.md` for a primer on Linux. Fundamentally, all commands take in the following input:

- Path of the executable to run
- Arguments to the executable
- Data from file descriptors

And all commands will output:
- Data to file descriptors
- A 32bit integer return code

Of course, they can also open files, access the network, etc. But we won't count that as input/output from the operating system's point of view. That's just what the program is doing to your computer.

#### Arguments and return code

```c
#include <stdio.h>
int main(int argc, char* argv[]) {
  printf("Argc: %d", argc);
  for(int i = 0; i < argc; i++) {
    printf("Argv[%d]: %s", i, argv[i]);
  }
  return 0;
}
```

This executable, will print out argc, all of argv, and then give a return code of 0. Return codes of 0 are expected to mean that a function _succeeded_, if you want to express failure, you can `return -1;` or some other non-zero value.

#### File Descriptors

What is a file descriptor? Very easy. A file descriptor is a device on Linux that you can either send data to, or read data from. Every command has 1024 of them, labelled from 0 to 1023. Most of them remain unused though and thus `unbound` since it's not connected to any device. File descriptors can be bound to your terminal window, so you can see what the output of a command is or send input to the command via typing. Or, they can be bound to files, so that commands can directly write their output to a file. By default, a command is expected to _read_ from file descriptor 0, _write_ to file descriptor 1, and write _errors_ to file descriptor 2. In libc, you're guaranteed to get the following `#define`'s in `stdio.h`,

```c
#define stdin  0
#define stdout 1
#define stderr 2
```

And, the following is the definition of the well-known `printf`:

```c
printf(const char* fmt, ...) {
  fprintf(stdout, fmt, ...);
}
```

On the same token, `scanf(...)` aliases to `fscanf(stdin, ...)`

Of course, nothing stops you from making a C program that writes to file descriptor 0, and reads from file descriptor 1, it just won't be what people expect when calling your program.

---

Now, for files. You might not realize it, but every time you do

```c
FILE* my_file = fopen("log.txt", "rw");
fprintf(my_file, "Hello World!");
```

this is in-fact just a wrapper around a file descriptor. You can do the same thing with

```c
int file_descriptor = open("log.txt", 0, O_RDWR);
char buf[1024] = "Hello World!";
write(file_descriptor, buf, strlen(buf));
```

But of course, the `fopen/fprintf` method uses libc and will thus work on any operating system that implements libc (i.e., it will work on Windows too). The `open/write` calls are Linux/POSIX system calls, and will only work on Linux/Mac.

But wait you ask, `stdout` is the integer 1, and `my_file` is a `FILE*`, how can we pass both into the first argument of `fprintf`? Okay well, I lied, the `#define stdin/stdout/stderr` will really `#define` it to a `FILE*` that wraps file descriptors 0/1/2. If you ever want to swap between the two:

```c
// From int file descriptor to FILE* file pointer
FILE* my_file = fdopen(my_file_descriptor, "rw");
// From FILE* file pointer to int file descriptor
int my_file_descriptor = fileno(my_file);
```

### Running a command

Now, we're ready to understand how bash allows you to interface with the three inputs and two outputs of every Linux executable. First, let's run a command:

```
$ echo Hello World!
Hello World!
```

Simple as that!

- First, the command you typed into the terminal will be separated by spaces into an `argv` of `["echo", "Hello", "World!"]`, and an `argc` of 3.
And yes, many people say that "`argv[0]` will be the command being called". This doesn't have to be true, but this is what bash decides to do.
- Thus, bash decides at this point to search the filesystem for an executable with a name that matches `argv[0]`. In this case it will find one in `/bin/echo`.
- Bash will then finally call `execv("/bin/echo", argv)`.

The C code that `/bin/echo` was compiled from is quite simply:

```c
#include <stdio.h>
// For any C program,
// argv is the array of arguments
// and argc is the number of arguments
int main(char** argv, int argc) {
   // Don't print argv[0]
   for(int i = 1; i < argc; i++) {
      fprintf(stdout, argv[i]);
   }
}
```

And hence, `argv[1] = "Hello"` and `argv[2] == "World!"` get propagated to the screen! But wait, how did it know `argc` if we only passed it `execv("/bin/echo", argv)`? Well, I lied again, bash actually generates an `argv` of `["echo", "Hello", "World!", NULL]`, since `execv` expects a NULL-terminated array. The C language decided to abstract this away for you by searching for the `NULL` and saving its index as `argc` for you. Try printing out `argv[argc]`, it's legal C!

In-fact, any space-separated list of strings written `A B C D` you write into bash will turn into `argv` of `["A", "B", "C", "D"]`, and Bash will then search your filesystem for an executable by the name of `A`. If `A` is a relative or absolute path such as `./my_local_executable`, then it'll just look for and run exactly that file. If it's a non-path string like `echo`, it'll inspect your entire file system. By "your entire file system", i mean your `PATH`, try `echo "$PATH"` to see where it'll search for executables.

