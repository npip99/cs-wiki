## Bash

### Introduction

First, if you haven't already, see `linux.md` for a primer on Linux. Fundamentally, all Linux commands take in the following input:

- Path of the executable to actually execute
- Arguments to the executable
- Environment variables
- A list of file descriptors

And all commands will return:
- A 32bit integer return code

And we'll be talking about what all of those input/outputs mean and how to modify them via bash.

Of course, Linux commands can also render to the screen, open a VM, play audio, restart your computer, etc. But we won't count that as input/output from the operating system's point of view for that process. Instead, that's just what the program happens to be doing to your computer while its running.

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

This executable, will print out argc (The number of arguments), all of argv (i.e., each arguments individually), and then give a return code of 0. Return codes of 0 are expected to mean that a function _succeeded_, if you want to express failure, you can `return -1;` or some other non-zero value.

Try compiling this and running it from bash with various arguments, and see what it does, use `gcc myfile.c -O myexec; ./myexec abc 123`. Try running it from another path via `cd ../..; ./dir1/dir2/myexec arg1 arg2` and see what prints out. We'll learn how bash passes these arguments to the executable in the Chapter _Running a command_.

Exercise ~ Try adding `printf("Argv[argc]: %s", argv[argc]);` to your code. This is valid, and will always give you the same answer!

### Running a command

Now, we're ready to understand how bash allows you to interface with the four inputs of every Linux executable. First, let's run a command:

```
$ echo Hello World!
Hello World!
```

Simple as that!

Under the hood, bash is doing the following:
- First, the command you typed into the terminal will be separated by spaces into an `argv` of `["echo", "Hello", "World!", NULL]`, aha there's that NULL!
- Then, bash decides at this point to search the filesystem for an executable with a name that matches `argv[0]`. In this case it will find one in `/bin/echo`.
- Bash will then finally call `execv("/bin/echo", argv)`, which is a Linux kernel syscall. Linux _requires_ argv to have a final element of NULL, that's how it knows how long argv is!

And yes, many people say that "`argv[0]` will be the command being called". This doesn't have to be true, but this is what bash decides to do, it just takes the first string you type into bash, and uses that for both `argv[0]` and the name of the executable it searches for.

The C code that `/bin/echo` was compiled from is quite simple as a matter of fact:

```c
#include <stdio.h>

int main(int argc, char** argv) {
   // Don't print argv[0]
   for(int i = 1; i < argc; i++) {
      fprintf(stdout, "%s", argv[i]);
      if (i < argc - 1) {
          fprintf(stdout, " ");
      }
   }
   fprintf(stdout, "\n");
}
```

And hence, `argv[1] == "Hello"` and `argv[2] == "World!"` get propagated to the screen! The C language decided to abstract the `argv` NULL problem away for you by searching for the `NULL` and saving its index as `argc` for you. Awesome!

Exercise ~ Try compiling the above, and running it with various arguments!

In-fact, any space-separated list of strings written `A B C D` you type into bash will turn into `argv` of `["A", "B", "C", "D"]`, so long as `"A"` isn't a bash keyword like `if` or `while`. And bash will always then search your filesystem for an executable by the name of `A`. If `A` is a relative or absolute path such as `./my_local_executable`, then it'll just look for exactly that file and try running it. If it's a non-path string like `echo`, it'll inspect your entire binary file system. By "your entire binary file system", I mean your `PATH`. Try `echo "$PATH"` to see the list of directories where bash will search for executables. The command `printenv` will show you a lot of interesting configuration variables that bash uses, see if you can find `PATH` in there.

---

Addendum:

Why NULL-terminate argv, why doesn't the Linux execv just have an argc parameter? Well, then it becomes not very simple to pass the array in. In-reality, to pass in ["echo", "Hello", "World!", NULL], Linux expects you to call execv as follows: `execv("/bin/bash", "echo\0Hello\0World!\0\0");`. Linux will know the string ends when two consecutive NULL bytes `\0\0` are found.

Have some fun exploring the following sample:

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

However, you probably will be disappointed to hear that MYVAR doesn't appear in `printenv`. Why is this? Well, there are bash variables, and then there are _environment_ variables. Environment variables get inherited by all child processes, and the Linux kernel is aware of them, while bash variables stay local to the current bash instance. Bash remembers them, but no one else does. Now for `printenv`, `printenv` is a simple executable that does the following:

```c
#include <stdio.h>
int main(int argc, char** argv, char** envp)
{
  // Print envp[i], until envp[i] == NULL
  for (int i = 0; envp[i] != NULL; i++) {
    printf("%s\n", envp[i]);    
  }
  return 0;
}
```

Here, the C language left you alone with no nice argc abstractions, so you'll have to find the NULL terminator yourself. Try compiling and running it. You might also be enticed to here that argv and envp are part of the same buffer (Try printing argv beyond the bounds of argv to see. Quite spooky looking beyond an array like that).

Note that this code captures an important quality of environment variables: When bash runs your C program, the environment of bash, gets automatically passed into the environment of the C program, even though bash and your C program are two different processes.

Okay so, how do you create an environment variable in bash? Specifically, how do we export our MYVAR variable that child processed can inherit it? Well, pretty simple, just write

```bash
export MYVAR
```

Or, for a single export-assign statement,

```bash
export MY_OTHER_VAR="Hello Other World!"
```

Now, after running the above two commands, try running that C code up above and perusing the result.  Cross-reference your result with `printenv` as well if you wish. Is your C code truly identical to `printenv`? If not, why, where?

Environment variables are a nice way to pass data into an executable. Many executables use environment variables extensively, `git` will look for `VISUAL` or `EDITOR` when deciding what editor to use when editing a git commit, and all graphical applications will look for `XAUTHORITY` and `DISPLAY` when finding the X11 Server and Display Device to render to. And bash itself, will look for `HOME` when deciding what the `~` key should alias to in paths.

### File Descriptors

File descriptors, as you've seen in `linux.md`, are pretty fundamental to Linux. They're so fundamental, that `bash` has very native built-in ways of working with them. This concept is called _redirection_, since it redirects where file descriptors will potentially read/write from.

```bash
echo hi 1>my_file
```

What this will do, is it will open up a file "my_file" with O_WRONLY permissions, and it will `dup` it into `echo hi`'s file descriptor "1" prior executing the `echo` itself.

```bash
echo hi 1<my_file
```

This will do the same thing, but using O_RDONLY instead of O_WRONLY. Since `echo` will write to file descriptor 1, this will crash.
However, what you can do is

```bash
cat 0<my_file
```

And this will create an O_RDONLY file descriptor to my_file, and dup it into file descriptor 0.

Of course, there are defaults. ">" will by default use file descriptor 1, and "<" will by default use file descriptor 0. Hence,

```bash
cat <in_file >out_file
```

Is a long-winded way to `cp in_file out_file`. Additionally, `>>` will pass in the O_APPEND flag, and `<>` will use O_RDWR. E.g.,

```bash
cat <in_file >>out_file
```

This will append in_file onto out_file

You can also be very cheeky, and _swap_ stdout and stderr by doing the following:

```bash
grep my_pattern 3>&2 2>&1 1>&3
```

Normally, redirection takes in a file name. But "&" acts like a reference here, and refers to _another_ filedescriptor. If you do `echo hi >2`, you've written to a file by the name of `2`. But if you do `echo hi >&2`, you've written to stderr. The way `A>&B` works under-the-hood is via `dup`s, and bash will always process redirections from left-to-right.

So now, let's understand our `3>&2 2>&1 1>&3` construction:

- Before anything happens, 0 refers to bash's stdin, 1 refers to bash's stdout, and 2 refers to bash's stderr. This is always the starting point.
- First, we dup stderr to file descriptor 3, so now file descriptor 3 directory points to bash's stderr as well.
- Second, we dup stdout to stderr. Now, stderr refers to bash's _current_ stdout.
- Third, we dup file descriptor 3 to stdout. But remember, file descriptor 3 refered to bash's _original_ stderr. Thus, stdout now also refers to bash's _original_ stderr.
- As an optional fourth, you can pass `3>&-`, which will close file descriptor 3. It reads visually like making file descriptor 3 point to a "null" file descriptor, but it translates fundamentally to the `close()` syscall.

Another interesting thing bash does for you is _piping_, which is an awesome feature of bash. You can see its usage here:

`find . -name "*.cpp" | xargs wc -l`

Here, the `find` will create a list of cpp files in your current directory. Then, `xargs` will read stdin and pass it in as args to `wc -l`. That causes `wc -l` (Reads "word count -l for lines"), to then give you the linecount of all of those cpp files as if you had passed them as arguments, along with the total line count of all of them!

But how does bash allow the output of `find`, to become the input of `xargs`? Very simple, and using a file of course! Linux does most of the abstraction for us though.

```c
int fds[2];
pipe(fds);
```

The above code will ask Linux to create a `pipe`. Linux will then create a file of type "pipe" in its VFS, and set fds[0] to an O_RDONLY version, and fds[1] to an O_WRONLY. The "pipe" file is handled using an in-memory filesystem known as "pipefs", and it's optimized specifically for passing "write" calls of one process into "read" calls of another. It'll also have a buffer of 64kb between the two processes to make sure that both are always running at all times if possible. But if e.g. one process writes faster than the other reads, and the buffer fills up, then then `pipefs` will block when the writing process tries to write past the `64kb` buffer, it'll only stop blocking once the receiving process reads some more.

The only thing bash will do after calling `pipe`, is dup fds[1] into find's stdout, and dup fds[0] into xargs's stdin. Then bash'll close its own fds[0]/fds[1], and now find/xargs are linked. It'll fork both of those processes out, and both will move along doing their thing. Bash won't continue onto the next line until both are done. If `xargs` finished first, fds[0] will `close` on exit, and fds[0] will be deallocated because no-one references it anymore. If fds[0] is deallocated, fds[1] will now be considered a "broken pipe". When that happens, if `find` tries to write to stdout, Linux will kill it with SIGPIPE because it wrote to a broken pipe and there's no-one there to read it. Bash doesn't care though and will completely ignore when the unimportant child dies due to SIGPIPE, Bash only cares if `xargs` dies since that's the end of the pipe train. You can do `set pipefail` if you care about a command failing due to a pipe failing, however.

---

One thing you might be asking is: When bash runs, how does bash setup stdin to take in my keyboard input, and how does stdout print to the terminal? The terminal isn't a file, right? Well, you actually _do_ in-fact have a file representing your terminal. When you open up a terminal tab in Gnome Terminal (The default terminal UI on Ubuntu), that terminal will create a tty device at `/dev/pts/0`, which is indeed a file in your file system. A tty device is a filetype that can be written to and read from, and it represents a _teletypewriter_, 1960s lingo for what we now call a terminal (As they were once connected to actual typewriters, back when writing to a `tty` would cause an electronic typewriter to type out the result on a piece of paper! If you like rabbit holes, investigate why the ASCII character `\r` is called a "Carriage-Return", look up what `unix2dos` does and why). In the good-ol days of the 70s-90s, when you would `write` to an `tty` device, Linux/(Unix) would reference internal bitmaps of what pixels those ASCII characters look like, and it would render those pixels directly onto the Monitor. You would be able to use `ctrl+alt+F{1-9}` to swap between `tty` devices. In-fact, Linux still supports this mode. Most Ubuntu distributions use `ctrl+alt+f4` to access your primary tty device 1980s-style, and `ctrl+alt+f1` to access a non-existant tty device that the Gnome Windowing system will render on-top of. When you use Gnome Terminal however inside of the Gnome Windowing System however, this is a _Terminal Emulator_, _not_ on actual hardware terminal you get from `ctrl+alt+f4`. A Terminal Emulator is a graphical program that reads what you `write()` to the `tty`, and it "emulates" what the terminal would have looked like, had the Linux kernel been rendering it full-screen (But often with colors too!). Those writing to the `tty` are none-the-wiser.

To see tty files in action, try opening up several Gnome Terminal tabs, and running `ls /dev/pts` after each one. Try running the command `tty` in each tab as well, to identify which one is which. When bash starts up, it dup's the tty into its own stdin/stdout/stderr, and marks stdin as O_RDONLY, and stdout/stderr as O_WRONLY. When bash runs a command, it has the child inherit bash's stdin/stdout/stderr. Then, the spawned executable can also read/write from the tty via stdin/stdout/stderr, while the tty can feed stdin keyboard input and write program output onto the typewr- er, terminal. 

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

Compare the results to `tty`'s response. Try looking up what `readlink` does, or what `/proc/self/fd` means. It's pretty interesting how Linux serves different values for the contens of `/proc/self` depending on which executable is the one asking. `/proc/self/fd` is also usually the only way to actually find the pipe files made by the `pipe` command. You can use `/proc/PID/fd/FD_NUMBER` to find the fd's for any process ID, "/proc/self" is just a virtual file that linux translates into /proc/PID whenever you try reading/writing to a file in it.