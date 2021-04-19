
## Filesystems

### History of ext{1,2,3,4}

In 1992, simultaneous with initial Linux development, the ext (Extended file system) format was created for filesystems, as an upgrade to the earlier minix filesystem format. This first iteration of ext on Linux was fairly unusable, a maximum storage limit of 2GB, only stored the file creation timestamp rather than the normal (Creation, Access, Modification) timestamp triplet, and the data allocated for a file was immutable so fragmentation was very difficult to address. Without the timestamp triplet, simple but crucial utilities like `Make` were impossible to use.

In 1993, ext2 came to solve all of these problems. Max filesize of 2TB, File timestamp triplet, and data fragmentation could be resolved easily due to simple mutability of file data location. It reigned for almost a decade, before one problem became evident: lack of journaling, in other words, it simply edited the file data live when writes are being made. Thus, with ext2, if the computer lost power or panicked while files were being written to, Linux would spend a rather painful 30seconds - multiple minutes checking filesystem integrity, and even then often the filesystem would be irreparably corrupted.

Thus, in 2001, ext3 came along. A very simple, but crucial, upgrade from ext2. ext3 was a _journaling_ filesystem. What that means, is that prior to a file being written to, it would add a copy of the file metadata (e.g. filename and filesize) to a special list of "actively changing files" at the top of the filesystem. This list is called the _journal_. When the file is done being written to, the file metadata would be removed from the journal. Now, if the computer crashes while a file is being written to, on the next boot, Linux will see the metadata copy at the top of the filesystem. It would restore the file metadata to its original state, and if the data has already been partially written to, the file will be wiped and have a size of 0 bytes. Yes your file might be corrupted, but the _filesystem itself_, is completely safe. And, file system checks remain in the milliseconds, even if something goes wrong, since the journal has all of the information necessary. Gone are the 200GB scans of the past! Of course, sometimes your program is _essential_, such as `apt`, and corruption of files is unacceptable. If you want to ensure that your program can't be corrupted, a journalism filesystem will give you 100% certainty of safety, simply write the following lines of code:

```c
// Example of how to update file.txt atomically
FILE* my_new_file = fopen("new_file.txt", "w");
fwrite(buf, sz, 1, my_file);
fsync(); // This will flush the cache to disk
rename("new_file.txt", "file.txt");
```

And you're done. Atomic updates of files. Yes this requires duplicating the file, but fundamentally it's impossible to create a more efficient method of atomically updating a file without simply duplicating its data prior to updating the data. And if you're fine having your file be truncated down to 0 bytes on improper shutdown, don't waste time with the fsync/rename idiom.

In 2008, we got ext4, which just updated file sizes to modern needs (16TB file size limit, 1 Exabyte filesystem limit), and got some internal updates to its algorithms to be more efficient in indexing and accessing large files. It didn't fundamentally change anything else. All modern Linux distros for the past decade will always install an ext4 filesystem, so if you're on Linux, you're probably running ext4 as you read this article.


