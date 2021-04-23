## Linux

This document explains all the ins-and-outs of the Linux Operating System. It's recommended that you read `linux-basics.md` first.

- The Virtual File System
    - Hardlinks and Softlinks
    - Permissions
- File Descriptors

### The Virtual File System

Before we talk about anything else, we should first distinguish two different types of filesystems. There's the standard physical filesystem that we're used to understanding. And, there's the _Virtual_ File System (VFS) that you've likely interacted with far more often.

When we talk about the physical filesystem, we often understand it quite well. But let's see why one is created: When Linux is installed onto a partition, the first thing we give it is a (hopefully) unused partition with nothing on it. So the first thing the linux installer does, is it splits the contiguous partition into physical files on disk, where each/file/path refers to a specific region of bytes on the partition to read/write from, and it stores metadata to keep track of which regions of bytes are for which files. Linux prefers the ext4 format for such data management on the physical filesystem.

But, when Linux actually boots, Linux internally in memory creates a _Virtual_ file system, which is what all of the executing code will interact with, and _that_'s what you end up seeing in bash itself as you peruse various directory. The Virtual File System has paths with files in it like usual, and some of those paths can indeed map to physical files on the ext4 filesystem. But, not always. For example, everything in `/tmp` will actually internally get stored on a separate filesystem data structure that's held in RAM, not on the actual hard disk. Additionally, the files in [/proc](https://tldp.org/LDP/Linux-Filesystem-Hierarchy/html/proc.html) don't actually exist anywhere, but are in-fact used to communicate with the Linux Kernel. For example, `cat /proc/cpuinfo` will tell you hardware information about your CPU. 

As you can see, Linux strongly adhers to the "Everything is a file" paradigm. What this means is that, in general, Linux heavily utilizes the read/write abstraction of files to manage all types of different things, and it does so via the VFS. Many C functions in-fact communicate over Linux files under-the-hood. For example, networking in C is implemented via the [socket](https://man7.org/linux/man-pages/man2/socket.2.html) syscall, which asks Linux to create a file on the filesystem that represents a TCP/UDP communication endpoint over a network socket. Reading/writing to that file will communicate to and from the internet! You can also mount entire partitions of your hard drive as `block device` files, and reading/writing to them will directly write bytes to the partition (Potentially destroying the metadata behind an ext4 filesystem that used to exist on that partition!). This is what `sudo mkfs -t ext4 /dev/your_partition` does to write a new ext4 filesystem to a partition, and that command is what the Ubuntu Linux Installer uses to install Linux onto your computer! Additionally, Linux can create a `tty` file that refers to a terminal window, and it's what bash uses for terminal input/output! You can learn more about bash terminals and `tty`s in `bash.md`.

#### Hardlinks and Softlinks

(**Hardlinks** `ln original_file new_file`) It's also possible to have multiple paths refer to the same file. For this conversation, the word "file" is getting ambiguous. Really, what we mean is "inode". An "inode" refers to a block of data on ext4 / RAM for tmp / Network for sockets / etc. However, a "file" is a "./human/readable/string" that points to an inode. It is possible for multiple files to point to the same inode, and when that happens, we say that the files are _hard links_ to eachother. You can create a hardlink by doing `ln ./file1 ./file2`, which takes in args like "cp", e.g. with a "from" and "to". Then, deleting "./file1" will indeed delete that filepath, but it won't delete the underlying data because the "./file2" filepath still points to that inode. However, now that "./file1" is gone, if you delete "./file2", the underlying inode will be freed since nothing points to it anymore. At that point, the file will be unrecoverable. However, there's one big downside to hardlinks: Hardlinks can only be made with files, not folders. This is because making a hardlink with folders could potentially create cycles. And even if Linux instead tried to check cycles, then _moving_ a folder from A to B could potentially create cycles between hardlinks inside of A and some ancestor of B. Checking for cycles every time you move a folder would make moving a folder very expensive, so instead hardlinks on folders is just not allowed by the Linux Kernel, and not checking could cause the Linux Kernel to be hung in an infinite loop, forcing a reboot.

So for that reason, I recommend you just never use hardlinks. If you never use hardlinks, you can use the words "file" and "inode" pretty interchangeably, because they map 1:1. Instead of hardlinks, I recommend you use symlinks, also known as symbolic links or soft links.

(**Symlinks** `ln -s "original_path" new_file`) Symlinks are special files that internally just store a string to the file that they refer to, but with a "symlink" marker so that Linux remembers that they're symlinks. And most importantly, they work the same with files as they do with directories! When you read/write a symlink, it will read/write to the underlying filepath described in the symlink. So they feel and act exactly like hardlinks would work, but now filesystem traversal is easy. "grep" will by default not follow symlinks when you grep recursively `grep -r my_pattern`. If you want to force the following of symlinks, use `grep -R my_pattern`, and if you get stuck in an infinite loop, that's your problem, not `grep`'s (Since the infinite loop is user-level and not kernel-level, you can just ctrl+c to exit). You can make a symlink with `ln -s "original_path" new_file`, again noting that it "feels" like "cp" in usage, but doesn't actually copy any data, it just marks new_file as a symlink to original_path. Note that `"original_path"` will be stored verbatim as a string! It doesn't even have to exist, symlinks are allowed to be broken, e.g. if `original_path` isn't a folder/file, or if you move the original file after making the symlink. `"original_path"` as a string can either be an absolute directory, or a relative directory. I recommend using absolute directories for things like `/usr/bin/gcc`, but relative directories for symlinks between files in your projects. Remember that "~" will expand prior to being passed into `ln`, so that's an absolute directory, but you can use it for things like `~/.vimrc` that you know aren't moving. A nice property is that rlative symlinks will still work if you move a parent folder that both the symlink origin and target are children of. (And now you see the importance of no cycles for hardlinks, so that we can have meaningful conversations about "parent" and "children" with respect to the filesystem)

Try perusing your `/usr/bin` directory, you'll probably find a good deal of symlinks there! E.g. if you have multiple gcc installs,

```
npip99@razer-blade:~$ file /usr/bin/gcc
/usr/bin/gcc: symbolic link to /usr/bin/gcc-7
```

#### Permissions

The Linux VFS has a few rules for interacting with it. For an executable to read to a file, the user who ran the executable needs to have read permissions for that file. It also needs to have read permissions for every parent directory of that file. There are two types of paths however: Absolute, and Relative. If you use an absolute path like "/home/myuser/some/path/to/file", then you must have read permissions for "/home", "/home/myuser", "/home/myuser/some", "/home/myuser/some/path", "/home/myuser/some/path/to", and "/home/myuser/some/path/to/file". However, if your current working directory is "/home/myuser/some/path", then you can instead try reading the relative path "./to/file", where "." refers to your current directory. Then, you only need read permissions for "./to" and "./to/file". When bash runs as a user, it has the limitations of the user who first logged in. Additionally, every command ran by bash is limited to the same restrictions as bash itself. Permissions can't increase when you call a child, unless you use something like `sudo` to escalate to a higher level of permission. The same restrictions hold for write permissions and execute permissions. Also, in order to "ls" a directory, you need execute permissions on that directory. If you don't have execute permissions on the directory, then you can still "cd" into a child, but you'd need to brute-force the directory names, you can't simply "ls" them.

All files have an "owner", which is the user that owns that file. Additionally, you the owner of your file can give a "group" access to your file. If you want to know what groups you yourself are a part of, just execute `groups`.

Now, we can look at how permissions for files in the VFS work:

```
npip99@DESKTOP-QG304U7:~/programming/cs-wiki$ ls -alh
total 92K
drwxr-xr-x  4 npip99 npip99 4.0K Apr 23 01:01 .
drwxrwxrwx 19 npip99 npip99 4.0K Apr 22 23:14 ..
drwxr-xr-x  8 npip99 npip99 4.0K Apr 22 23:14 .git
-rw-r--r--  1 npip99 npip99   46 Apr 22 23:14 .gitattributes
-rw-r--r--  1 npip99 npip99 1.6K Apr 22 23:14 README.md
-rw-r--r--  1 npip99 npip99  952 Apr 22 23:14 apache.md
drwxr-xr-x  3 npip99 npip99 4.0K Apr 22 23:14 assets
-rw-r--r--  1 npip99 npip99 9.4K Apr 23 00:04 bash.md
-rw-r--r--  1 npip99 npip99 3.5K Apr 22 23:14 filesystems.md
-rw-r--r--  1 npip99 npip99 7.9K Apr 22 23:14 heaps.md
-rw-r--r--  1 npip99 npip99 4.5K Apr 22 23:14 linux-basics.md
-rw-r--r--  1 npip99 npip99  11K Apr 23 01:01 linux.md
-rw-r--r--  1 npip99 npip99  463 Apr 22 23:14 mysql.md
-rw-r--r--  1 npip99 npip99  13K Apr 22 23:14 optimizing-compilers.md
```

As you can see, there's a lot to look at. The first column we'll talk about shortly. The second column is how many "hard links" there are of that file, it's just a reference counter. When the reference counter goes to 0, Linux will free the file (And delete its data if the file exists on the underlying ext4 filesystem). The third column is the owner. The 4th column is the group (Which can also refer to the owner if there is no group). The 5th column is filesize (Though folders are just lists of filenames, so folders will show up only as 4096 even if the underlying files are GB large). The 6th column is the last modification time. And the 7th column, is the filename.

Permissions for files are organized in the following way:

`drwxrwxrwx`

The "d" is the directory bit, it will appear as a "d" for directories, and "-" for files.

Now, for each "rwx" triplet, this refers to the standard read/write/execute permissions.

The first "rwx" triplet refers to what permissions you the owner have over your own file. You can obviously change this at will, since you own the file. This is mostly just for your own protection. By not having the "x" bit, you won't accidentally execute a file that you didn't want to. And if you remove the "w" bit, you won't accidentally overwrite a file you didn't mean to. For example, "rm -r" will ask you if you're sure you want to delete the file that's write-protected (i.e. no write bit), but "rm -rf" will forcefully delete all files you have the ability to delete. There's, not very much of a reason to prevent yourself from reading your own file.

The second "rwx" triplet refers to what permissions members of the file's group have. So if you allow a file to have the group "UCF", and give it "r--" privileges, then all users who are a part of the group "UCF" will be able to read your file. This is pretty useful when you have many people in a University, and you have "grous" which can refer to clubs for example. Or you have a "professor" group and you can let professors read your work without letting anyone else read your work.

The third "rwx" triplet is what permissions you give to the average random user who isn't apart of any useful groups or anything. By giving a file "r--" permission, you're letting the world read it. But of course, if the parent directory doesn't have "x--", then it's not like the user could've navigated to that directory anyway.

In order to modify permissions, use the `chmod` command. You can do `chmod u-x` to remove executable permissions from yourself (The user), and `chmod g+w` to give write permissions to the group owner of that file. You can change the owner/group with `chown USER:GROUP ./file`. But of course, only `root` can change the user. However, you're allowed `chown yourself:some_group` so long as you're apart of that group (via the `groups` commands). Only `root` can change what groups a user is a part of.

---

Sometimes, you'll notice the following triplet:

`-rwsr-xr-x`

With an `s` instead of an `x`.

When the setuid bit is set, then the executable will run _as if the owner ran it_, even if the person executing the file is some other user with lesser permissions. The only executable that really uses the setuid bit, is `sudo`, and you may note that `root` owns the `sudo` command. This is how `root` lets other people run commands as if they were `root`, even though they aren't. The `sudo` command just checks if you're apart of the `sudo` group, and if you are it'll run the command as root, otherwise, it'll exit. It'll also ask for your user's password, to ensure that you didn't just leave your terminal unattended. `chown` also uses the setuid bit, and it'll check that you're a member of a group before changing the group of a file to that group. (If you see `-rwS` instead of `-rws`, that's because you set the setuid bit without setting the executable bit, which isn't very helpful!)

If you want to have fun with the setuid bit on a University network, you can make an executable `leave-message` that takes stdin and appends it to a `messages` file in your directory. Then you can give everyone in your University `--x` permissions on `leave-message` (Either via the "all" triplet, or the "group" triplet if there's a group for e.g. all University Students). You can mark `messages` with `-rw-------`, and now you can collect messages! (And without letting the message-leavers read other people's messages as well). Go ahead, ask people to use it! Maybe after a while, someone might exploit this by spamming your inbox. If you want to make this strong, you can save in a `messages-usage` file a mapping between USER_IDs and total message data sent in bytes. And don't let them send more than some number of bytes per day ;) Perhaps you can add a whitelist/blacklist system as well if you wish. Lots of opportunity.

### File Descriptors

So, we've been talking a lot about files, but how exactly do we use files? Well, it would be very expensive to have to do something like `write("./file", data, len)`, since each time it would have to inspect the entire VFS for what that file points to, and _then_ most likely access the ext4 filesystem if it's a normal harddisk file, and _then_ calculated all of the complex permissions systems to ensure that you even have access to the file in the first place!

So, in order to make everything a lot faster, instead, before you can use a file, you have to open a _file descriptor_. You can do that via the following:

```
int fd = open("log.txt", 0, O_RDONLY);
```

This will open up "log.txt", but with read-only permissions. Of course, if you don't have the ability to read "log.txt", then open will return `-1` and `errno` will equal `EACCES`. Now, `fd` can be used as a "reference" to that file. The Kernel keeps track of all of the file descriptors you have own, and future `read`/`write` calls will use these file descriptors. Then 2nd argument to `open` is for flags, e.g. `O_CREAT` will create the file if it doesn't already exist, and `O_NOFOLLOW` will cause the `open` to fail if you're trying to open a symlink (E.g., you should use `O_NOFOLLOW` when traversing the filesystem in order to avoid infinite loops)

For general information on file descriptors:

- There are up to 1024 file descriptors that a process can use effectively
- Each opened file descriptor can be marked as read-only, write-only, or read-write. The `read`/`write` syscalls will fail if the file descriptor is missing the correct permission.
- A file descriptor points to an actual file with a path on your operator system, but fundamentally files themselves can refer to anything. 
- Multiple file descriptors can point to the same file, even if each individual file descriptor has different read/write privileges.
- A "reference" to a file is closed when no more file descriptors point to that file. At that time, the linux will likely deallocate any RAM that the file was using.

---

Usage of file descriptors is fairly simple. You've already seen the `open` command. But here's an example of a few more Linux commands:

```c
// This will duplicate my_fd into new_fd, and both will now point to the same underlying file
int new_fd = dup(my_fd);

// This will close my_fd,
// but the Kernel won't actually close or clean-up anything because new_fd is still pointing to it
close(my_fd);

// This will close(10) if 10 happens to be open,
// And it'll make 10 and new_fd point to the same VFS file
// So essentially, it "duplicates" my_fd _onto_ file descriptor 10.
dup2(new_fd, 10);

// This will now close file descriptor 10 and all of its resources because no-one points to the underlying file anymore
close(10);

// This will create a file descriptor refering to a network socket
// After this call, you can use `sendto` to send packets over UDP
int udp_socket = socket(AF_INET, SOCK_DGRAM, IPPROTO_UDP);
char buf[1024];

// Read up to 1024 characters from the UDP socket. This command works for _all_ files.
// If you want to know who _sent_ you the UDP packet, you'd have to use recvfrom though, which is socket-specific
read(udp_socket, buf, sizeof(buf));
```

`dup2` between two integers is often a bit hard to understand. If you're used to some C-pointer intuition, it seems like dup(A, B) should make B point to A. And then, updating A would update B. But it doesn't, _A and B have nothing to do with eachother once you dup them_. Fundamentally, all open file descriptors point to _an actual file_ in the VFS. When you do dup(A, B), you _force_ B to point to the _same underlying file_ that A used to point to. Then, at that point, B does not care one bit about A at all, B instead just has a direct reference to the underlying file.

---

Now, you might not've realize it, but `FILE*` is in-fact just a wrapper around a file descriptor! Which, makes sense now. When you write,

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

And if you ever want to swap between the two to wrap/unwrap the `FILE*` abstraction,

```c
// From int file descriptor to FILE* file pointer
FILE* my_file = fdopen(my_file_descriptor, "rw");
// From FILE* file pointer to int file descriptor
int my_file_descriptor = fileno(my_file);
```

But of course, the `fopen/fprintf` method uses libc and will thus work on any operating system that implements libc (i.e., it will work on Windows too), hence why a cross-platform `FILE*` wrapper was made for the libc specification. The direct `open/write` calls and the whole file descriptor system is Linux/POSIX-only, and will thus only work on Linux/Mac.

---

Now, here's an interesting tid-bit: `printf` writes to a file descriptor! But how?

Well, by default, a command is expected to _read_ from file descriptor 0, _write_ to file descriptor 1, and write _errors_ to file descriptor 2. The `FILE*`'s for file descriptor 0 is known as `stdin`, the `FILE*` for file descriptor 1 is known as `stdout`, and the `FILE*` for descriptor 2 is known as `stderr`. `stdin/stdout/stderr` are simply `#define`'s to pre-made `FILE*`'s in `stdio.h`. Go ahead, try printing `fileno(stdin)` in a C program and see what you get. Try the same for `stdout` and `stderr`

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
close(0); // Close file descriptor 0, the reference to normal stdin, since we don't want to listen to stdin anymore.
dup2(fd, 0); // Now, duplicate the my_file.txt file descriptor fd onto file descriptor 0, so that both point to my_file.txt
close(fd); // Close the no longer needed file descriptor fd
           // Closing this won't actually detach the file from our process, because the file is _also_ bound to fd 0 now.
           // The kernel-level reference onto the file won't be freed until no more file descriptors point to it
// Writing to file descriptor 0! Kind of weird though, but it works and will write to my_file.txt!
char* str = "Hello World!";
write(0, str, strlen(str));
// We can also try to "write" to stdin,
// but the `FILE*` struct is marked by libc as read-only,
// so this won't work even though fd0 to write-only.
fprintf(stdin, "Hello World!");
// Now, we can close fd 0, and our lock onto my_file.txt will be freed
close(0);
```

---

And as a final note, something you might be confused by is: If file descriptors represent files on the filesystem, how does stdin take in my keyboard input, and how does stdout print to the terminal? To answer this question, we need to learn more about how bash and `tty`'s work. If you want to learn more, you can take a view of `bash.md`.
