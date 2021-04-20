
## Bash

### Introduction

First, if you haven't already, see `linux-basics.md` for a primer on Linux. Fundamentally, all Linux commands take in the following input:

- Path of the executable to actually execute
- Arguments to the executable
- Environment variables
- Data from file descriptors

And all commands will output:
- Data to file descriptors
- A 32bit integer return code

And we'll be talking about what all of those input/outputs mean.

Of course, commands can also render to the screen, access the network, etc. But we won't count that as input/output from the operating system's point of view for that process. Instead, that's just what the program happens to be doing to your computer while its running.

### Arguments and return code

```c
#include <stdio.h>
// For any C program,
// argv is the array of arguments
// and argc is the number of arguments
int main(int argc, char** argv) {
  printf("Argc: %d", argc);
  for(int i = 0; i < argc; i++) {
    printf("Argv[%d]: %s", i, argv[i]);
  }
  return 0;
}
```

This executable, will print out argc (The number of arguments), all of argv (i.e., each arguments individually), and then give a return code of 0. Return codes of 0 are expected to mean that a function _succeeded_, if you want to express failure, you can `return -1;` or some other non-zero value. Try compiling this and running it from bash with various arguments, and see what it does, use `gcc myfile.c -O myexec; ./myexec abc 123`. Try running it from another path via `cd ../..; ./dir1/dir2/myexec arg1 arg2` and see what prints out. We'll learn how bash passes these arguments to the executable in the Chapter _Running a command_, and we'll also learn the reason behind a spooky fact, `printf("Beyond the Argv: %s", argv[argc]);` is in-fact valid C and it prints `Beyond the Argv: (null)`!

### Running a command

Now, we're ready to understand how bash allows you to interface with the three inputs and two outputs of every Linux executable. First, let's run a command:

```
$ echo Hello World!
Hello World!
```

Simple as that!

Under the hood, bash is doing the following:
- First, the command you typed into the terminal will be separated by spaces into an `argv` of `["echo", "Hello", "World!", NULL]`, aha there's that NULL!
- Then, bash decides at this point to search the filesystem for an executable with a name that matches `argv[0]`. In this case it will find one in `/bin/echo`.
- Bash will then finally call `execv("/bin/echo", argv)`, which is a Linux kernel syscall. Linux _requires_ argv to have a final element of NULL, that's how it knows how long argv is!

And yes, many people say that "`argv[0]` will be the command being called". This doesn't have to be true, but this is what bash decides to do, it just takes `argv[0]` and looks for that executable name when deciding which executable to run.

The C code that `/bin/echo` was compiled from is quite simply:

```c
#include <stdio.h>
int main(int argc, char** argv) {
   // Don't print argv[0]
   for(int i = 1; i < argc; i++) {
      fprintf(stdout, argv[i]);
   }
}
```

And hence, `argv[1] == "Hello"` and `argv[2] == "World!"` get propagated to the screen! The C language decided to abstract the `argv` NULL problem away for you by searching for the `NULL` and saving its index as `argc` for you. Awesome!

In-fact, any space-separated list of strings written `A B C D` you type into bash will turn into `argv` of `["A", "B", "C", "D"]`, so long as `"A"` isn't a bash keyword like `if` or `while`. And bash will always then search your filesystem for an executable by the name of `A`. If `A` is a relative or absolute path such as `./my_local_executable`, then it'll just look for exactly that file and try running. If it's a non-path string like `echo`, it'll inspect your entire binary file system. By "your entire binary file system", I mean your `PATH`, try `echo "$PATH"` to see where it'll search for executables as a list of directories. The command `printenv` will show you a lot of interesting configuration variables that bash uses, see if you can find `PATH` in there.

---

Addendum:

Why NULL-terminate argv, why doesn't the Linux execv just have an argc parameter? That's because you might _lie_ about argc and cause Linux to buffer overflow or hit undefined behavior if it believes you. So Linux doesn't believe you, it'll look for a NULL, and if there is no NULL terminator in the first ~32,000 elements of `argv` then `execv` will return an error. It'll also write all of the arguments to the same buffer, one after another, and the total length of that buffer can't be longer than ARG_MAX either. Otherwise `execv` will again just return with an error. Here's a fun one, try running the following program with various arguments so you can see the argument buffer itself:

```c
#include <stdio.h>
#include <string.h>
int main(int argc, char** argv) {
  // Get total length of all arguments
  int total_len = 0;
  for(int i = 0; i < argc; i++) {
    total_len += strlen(argv[i]) + 1; // Length includes NULL at end of each string
  }
  // Now, print out total_len chars starting at argv[0],
  // which is both the first argument, and the start of the arg buffer
  for(int i = 0; i < total_len; i++) {
    char c = argv[0][i];
    if (c == '\0') printf("\nNull!\n");
    else printf("%c", c);
  }
}
```

### Variables

You saw a sneak-peak of variables with `echo "$PATH"`. What if you want to set your own variable? Very easily,

```bash
MYVAR="Hello World!"
echo "$MYVAR"
```

And, you can also set variables to the result of a command!

```bash
MYVAR="$(echo Helloooooooooo)"
echo "$MYVAR"
```

However, you probably will be disappointed to hear that MYVAR doesn't appear in `printenv`. Why is this? Well, there are bash variables, and then there are _environment_ variables. Environment variables get passed onto all child processes, and the Linux kernel is aware of them, while bash variables stay local to the current bash instance, and they just sit in bash's RAM to help you write bash scripts. Now for `printenv`, `printenv` is a simple executable that does the following:

```c
#include <stdio.h>
int main(int argc, char** argv, char** envp)
{
  // Print envp[i], until envp[i] == NULL
  for (int i = 0; envp[i] != NULL; i++)
  {
    printf("%s\n", envp[i]);    
  }
  return 0;
}
```

Here, the C language left you alone with no nice argc abstractions, so you'll have to find the NULL terminator yourself. Try compiling and running it. You might also be enticed to here that argv and envp are part of the same buffer (Try printing argv beyond the bounds of argv to see. Quite spooky looking beyond an array like that).

Okay so, how do you create an environment variable in linux? Specifically, how do we export our MYVAR variable that just isn't making it to the child process yet? Well, pretty simple, just write

```bash
export MYVAR
```
Or, for a single export-assign statement,

```bash
export MY_OTHER_VAR="Hello Other World!"
```


Now, after running the above two commands, try running that C code up above and perusing the result.  Cross-reference your result with `printenv` as well if you wish. Is your C code truly identical to `printenv`? If not, why, where?

Environment variables are a nice way to pass data into an executable. Many executables use environment variables extensively, `git` will look for `VISUAL` or `EDITOR` when deciding what editor to use when editing a git commit, and all graphical applications will look for `XAUTHORITY` and `DISPLAY` when finding the X11 Server and Display Device to render to. And bash itself, will look for `HOME` when deciding what the `~` key should alias in paths.

### File Descriptors

Okay, so, we have arguments, return codes, and environment variables. But what happened to file descriptors?

We first have to answer the question: what is a file descriptor? Very easy. A file descriptor, is an integer that acts as a reference to an actual file on the Linux filesystem. Every executable has 1024 of them, labelled from 0 to 1023. Most of them remain unused and thus `close()`'d since it's not bound to any actual file on the file system.

You might not realize it, but `FILE*` is just a wrapper around a file descriptor! When you write,

```c
FILE* my_file = fopen("log.txt", "rw");
fprintf(my_file, "Hello World!");
```

You can do the exact same thing with

```c
int file_descriptor = open("log.txt", 0, O_RDWR);
char buf[1024] = "Hello World!";
write(file_descriptor, buf, strlen(buf));
```

And if you ever want to swap between the two:

```c
// From int file descriptor to FILE* file pointer
FILE* my_file = fdopen(my_file_descriptor, "rw");
// From FILE* file pointer to int file descriptor
int my_file_descriptor = fileno(my_file);
```

But of course, the `fopen/fprintf` method uses libc and will thus work on any operating system that implements libc (i.e., it will work on Windows too), hence why a cross-platform `FILE*` wrapper was made for the libc specification. The direct `open/write` calls and the whole file descriptor system is Linux/POSIX-only, and will thus only work on Linux/Mac.

---

Now, I have a secret: `printf` writes to a file descriptor! But how? Why?

Well, by default, a command is expected to _read_ from file descriptor 0, _write_ to file descriptor 1, and write _errors_ to file descriptor 2. The `FILE*`'s for file descriptor 0 is known as `stdin`, the `FILE*` for file descriptor 1 is known as `stdout`, and the `FILE*` for descriptor 2 is known as `stderr`. `stdin/stdout/stderr` are simply `#define`'s to pre-made `FILE*`'s in `stdio.h`. Go ahead, try printing `fileno(stdin)` in a C program.

As a specific example, the following is the definition of the well-known `printf`:

```c
printf(const char* fmt, ...) {
  fprintf(stdout, fmt, ...);
}
```

On the same token, `scanf(...)` is simply a wrapper around `fscanf(stdin, ...)`

Of course, this is just convention, nothing stops you from making a C program called `./weird_one` that writes to file descriptor 0, and reads from file descriptor 1.  Simply write `fprintf(stdin, "Hello World!")`. It just won't be what people expect when calling your program. And, bash will be what connects `stdin` of your C program to the terminal of the person who ran the command. Bash is going to end up marking `stdin` as O_RDONLY / read-only to your C program, so `fprintf(stdin, ...)` will actually just crash if `./weird_one` is ran from a bash shell (Feel free to experiment). But, it won't necessarily crash if you have a C program that binds stdin to a file with write permissions, and proceeds to `execv("./weird_one", argv)`.

You can bind a new file to a specific file descriptor by doing the following:

```c
int fd = open("my_file.txt", 0, O_WRONLY);
close(0); // Close file descriptor 0
dup2(fd, 0); // Duplicate the file descriptor fd onto file descriptor 0
close(fd); // Close the no longer needed file descriptor fd
// Writing to stdin! Kind of weird though, but it works and will write to my_file.txt!
fprintf(stdin, "Hello World!");
```

---

And as a final note, something you might be confused by is: If file descriptors represent files on the filesystem, how does stdin take in my keyboard input, and how does stdout print to the terminal? Well, you actually do have a file representing your terminal. When you open up a terminal tab in Gnome Terminal (The default terminal UI on Ubuntu), that terminal will create a tty device at `/dev/pts/0`, which is indeed a file in your file system. A tty device is a filetype that can be written to and read from, and it represents a _teletypewriter_, 1960s lingo for what we now call a terminal (As they were once connected to actual typewriters, back when stdout would cause an electronic typewriter to type out the result on a piece of paper! If you like rabbit holes, investigate why the ASCII character `\r` is called a "Carriage-Return", look up what `unix2dos` does and why).

To see tty's in action, try opening up several Gnome Terminal tabs, and running `ls /dev/pts` after each one. Try running the command `tty` as well, to identify which one is which. When bash runs a command, it dup's the tty into the command's stdin/stdout/stderr, and marks stdin as O_RDONLY, and stdout/stderr as O_WRONLY. Then the spawned executable can read/write from the tty via stdin/stdout/stderr, while the tty can feed keyboard input and write program output onto the typewr- er, terminal. 

If you want to know how the `tty` command was made, try running following C code:

```c
#include <stdio.h>
#include <unistd.h>
int main(int argc, char **argv) {
  char buf[1024];
  size_t size = readlink("/proc/self/fd/0", buf, sizeof(buf));
  buf[size] = '\0';
  printf("%s\n", buf);
}
```

Compare the results to `tty`'s response. Try looking up what `readlink` does, or what `/proc/self/fd` means. It's interesting, the `/proc/self/fd` path has different contents based on who's reading from the file, doesn't it? The whole `/proc/` directory is a bit funny, none of it seems to actually exists on your hard drive, it seems awfully, `proc`edural. Try `cat /proc/cpuinfo`. Try calling `cat /proc/uptime` several times in rapid succession. What are those two numbers? `/proc` has a bunch of files, that Linux will give you the contents of whenever you ask. But, all of those "files" appear to just be ways to ask the Linux kernel different questions. It's an interesting concept, Everything is file. If it exists on the hard drive, `read` will grab its contents, but if it represents a question to the kernel like `/proc/uptime`, then `read` just runs some computations on the number of times a crystal oscillated. Go ahead, try it, `fopen("/proc/uptime")` then `fread` from it.

