# File Descriptors

File descriptors = small nonnegative integers

Terminal by default sets up 0 1 and 2.

max open files per process:
- preproc macro OPEN_MAX
- sysconf(_SC_OPEN_MAX) - when running, we need to reset errno to correctly differentiate
between the unsupported and error possibilities.
- getdtablesize() - is equivalent to calling sysconf
- getrlimit(RLIMIT_NOFILE, ...) - per process limits.

- fstat() - can be used to check whether a file is open or not.
- open() - sets errno to EMFILE when hitting the max open file limit.

The true limit is found through getconf.
Under /usr/src/usr.bin/getconf/getconf.c we can see it uses SYSCONF
And the printvar calls sysconf. So, getconf(OPEN_MAX) is sysconf(_SC_OPEN_MAX)

`ulimit -n` shell builtin command to show open file limit
If we use `ulimit -n 64` the preproc macro OPEN_MAX remains incorrect while
the other methods update.

This limit can be get and set at runtime. The result might change from one invocation to the next.

Different OSes have different defaults or just don't define OPEN_MAX macro at all.

Exercises:
- `ulimit -n 0`
    - causes my terminal on MacOS to start doubling up characters.
       and commands don't go through correctly. I have lots of different
       software setup so that could contribute to the output.
    - on the NetBSD, if I am on zsh, commands like `ls` and `cat`
       fail and zsh says pipe failed. I was able to set it back to 1024.
       If I am on bash, ls tells me that shared object libutil.so.7 isn't found.
       Trying to call ulimit -n 1024 to set it back fails because
       operation not permitted.

- `ulimit -n unlimited` as root sets the value to 10000.

- `_POSIX_OPEN_MAX` seems to be defined as 20, like what was explained in the
   man page

- fd-exercise - it seems like stdio streams have an _file attribute. On my MacOS system it's a short.
   open under /tmp is weird on mac. On BSD I get the expected 3 and 4 for the consecutive
   open(2) and fopen(3) calls.
   When opening the same file twice, things get overwritten as there are two open files.
   With alternating reads and writes, the position keeps on getting incremented
   from where it was last set.
   I could keep reading until the word and seek backward the length of the word, and then write over it.
   Although this wouldn't insert new chars.

# open and close

```c
#include <fcntl.h>
int creat();
```
all file I/O can be done with the following syscalls:
- `open(2)` `close(2)` `read(2)` `write(2)` `lseek(2)`

`creat(2)` isn't part of it because it only returns you a fd in write-only mode. In early days,
you had to creat, close, then reopen, posing a race-condition. Creation and opening was not atomic.

Now it's obsoleted with the flags O_CREAT | O_TRUNC | O_WRONLY

## open(2)
- O_RDONLY, O_WRONLY, O_RDWR
- O_APPEND
- O_CREAT
- O_EXCL (atomic exclusive opening of a file and avoids TOCTOU)
- O_TRUNC
- O_NONBLOCK - don't block on open or for data to become available
- O_SYNC - wait for physical I/O to complete

There are non-POSIX flags. The above are POSIX.

openat(2) - used to handle relative path names from a different working dir in an atomic fashion.

Errors:
- EEXIST
- EMFILE - max file limit reached
- ENOENT - does not exist (no entry in the dir)
- EPERM
- ...

Always check return code.
```c
if ((fd = open(...)) < 0) {
    err();
}
// continue
```

## close(2)
closing a file releases record locks on that file

fds are closed by the kernel automatically if you forget,
but that is bad practice and could leak fds. So,
close in the same scope.

use `(void)close(fd);`

we don't really need to check the return code, even if it fails.
But, we want to make sure to explicitly cast to void to show that we don't want to check it.

stdout-exercise:
```c
int fd;

(void)close(STDOUT_FILENO);
if ((fd = open("./testfile", O_CREAT | O_APPEND, 0666)) < 0) {
    err();
}

#define MSG "This message goes to stdout.\n"
if (fd == STDOUT_FILENO) {
        write(fd, MSG, strlen(MSG));
}

(void)close(fd);
```

# read write and lseek

## read(2)

reads at current offset
increments offset by number of bytes actually read.
returns num bytes read, 0 on EOF, -1 on error.

- reading from a network - buffering can cause delays in arrival of data.
- record oriented devices (like magtape) may return data one record at a time.
- interruption by a signal

## write(2)

writes at current offset
increments offset by number of bytes actually written.
if file was opened with the O_APPEND flag, write atomically
updates position to the end and writes at the end of the file.
However, it has no effect on the position.

returns numbytes written and -1 on error
write will block until all has been written.
if in non-blocking mode, and the object represented by fd
is subject to flow-control, (i.e. socket), then write may
write fewer than requested, and we have to manually retry writing.

write(2) clears set-uid bits on files so that trying
to overwrite set-uid binaries with malicious
code is strictly prohibited.

try writing a prog to try to exploit the vuln if it existed.
```c

```

If we don't use O_APPEND with read write mode,
then writes will occur right after the read is done

vim shows the byte position as well.

Writes DO NOT insert data. It overwrites data.

## lseek(2)

```c
#include <sys/types.h>
#include <fcntl.h>
```

lseek has l because it used to return long before it was
changed to off_t.

- SEEK_SET = bytes from beginning
- SEEK_CUR = bytes from current file pos
- SEEK_END = bytes from end of file

Weird things:
- negative offset - with SEEK_SET, -1 is an Invalid argument.
- seek 0 bytes. (not quite a no-op) we can use this to check if a fd is seekable.
- seek past end of file.

Can you go backwards on a pipe?
- pipes and fifos are record oriented. can't seek back and forth.

What happens when you write past the end of the file?
- File holes - seek past eof and write.
when printing it out with hexdump, the kernel pretends it's null,
but it's actually just nothing on disk. It's a sparse file.
Copying the file with a hole actually fills the missing bytes with null.
Diff says the files are the same because kernel says there's null bytes.

Sparse files needs to be supported by the filesystem.

on Linux with the NFS share the file.hole increased in size
and the .nohole is smaller?
after another ls -ls, the blocks are the same. We observed
network latency.

With a local fs, Linux depending on the cp version, it can preserve
the hole in the file (sparse file).

cat doesn't detect sparse files so it will make the null bytes into real bytes.


# How efficient is simple-cat.c?

Buffer size efficiency

 >32k doesn't gain you much performance.

also the FS has a fixed block size, so you are bottlenecked.

`stat -f "%k" tmp/file1`
```man
k           Optimal file system I/O operation block size
                       (st_blksize).
```

These results differ between FSes and OSes.

# File Sharing

Each process table entry has a table of file descriptors,
which contain file descriptor flags and pointer to file table
entry.

Kernel maintains the file table. Each entry has
file status flags, current offset, pointer to vnode table
entry.

vnode structure contains vnode info and inode info (including current file size).

lseek merely adjusts the current file offset in the file table.

Before O_APPEND, you needed to lseek then write. This causes a race condition.

To atomically write anywhere else, we can use `pread` and `pwrite`.

Shell:
`echo bar > file` flags are `O_WRONLY | O_CREAT | O_TRUNC`
`echo bar >> file` flags are `O_WRONLY | O_CREAT | O_APPEND`

`ls -l file /nowhere 2>/dev/null`
`ls -l file /nowhere >file` shows file as 0 bytes because of the TRUNC
`ls -l file /nowhere >file 2>file` stderr seems to disappear. it gets written first.
`ls -l file /nowhere >file 2>>file` stdout does TRUNC, and stderr writes first in APPEND
`ls -l file /nowhere >>file 2>file` seems to fix the issue.

stderr is generally unbuffered. stdout is generally buffered.

`ls -l file /nowhere >file 2>&1` works.

Using & calls the dup(2) syscall.

`fcntl(2)` - catch-all function that does many things.
- F_DUPFD
- F_GETFD - get fd flags
- F_SETFD - set fd flags
- F_GETFL - get file status flags
- F_SETFL - set file status flags

O_SYNC guarantees I/O is flushed right away.

using
```c
flags = fcntl(fd, F_GETFL, 0);
flags |= O_SYNC;
fcntl(fd, F_SETFL, flags);
```

`ioctl(2)` - catch-all for device specific
like terminal I/O, magtape access, socket I/O, etc.

FS /dev
- char devices /dev/{stdin,stdout,stderr}

`echo two | cat first /dev/stdin third`
prints contents of first, then two, then contents of third.

in macos, /dev/stdin etc are symlinks to /dev/fd/0,1,2

under the other file descriptors, we get directories.

in macos, /dev/fd/3 seems to be the cwd.

pipes also show up

in linux, /dev/stdout is symlinked to the /proc/self/fd/1. /dev/fd also points to /proc/self/fd.

pipe could be implemented through socket.

pid of current shell `echo $$`. some shells could have open files for history.

`echo foo | cat /dev/stdin` - error no such device or address. linux appears to impl pipe using socket. can't open() sockets.

---

week2 checkin (not answered above)

a flag for open(2) I found on MacOS:

    The O_EVTONLY flag is only intended for monitoring a file for changes
    (e.g. kqueue). Note: when this flag is used, the opened file will not
    prevent an unmount of the volume that contains the file.


Run the hole.c program on different Unix systems (NetBSD, macOS, Linux, ...).
Does the behavior differ? Why?
- Yes. The NetBSD filesystem supports the sparse file. Meanwhile on macOS,
the hole just gets filled with actual null bytes. This is because the
filesystem on macOS (APFS as indicated in Disk Utility) doesn't support the
sparse files. For Linux, it seems that the filesystem I have supports the
sparse files as well, and I checked with the ls -ls command to see the block
count wasn't 10000.

You end up at offset 10 because the file is of size 10 bytes after
successfully writing 10 bytes. If my understanding is correct,
the file offset will already be at 10 if you don't use O_APPEND.

Yes.

First redirects stdout to file, updating fd table entry 1, then redirects stderr to go wherever stdout has been changed to, updating fd table entry 2.

Second redirects stderr to go to wherever stdout is, updating fd table entry 2, but then redirects stdout to the file, updating fd table entry 1. So, the stderr message goes to the original stdout, and the stdout message goes to the file.

---

Questions:

Can we go over O_SYNC and fcntl(2) again?

- Remember basic I/O is on file descriptors.
- fcntl set the file status flags.
- sync means the writes all immediately go to disk
- usually kernel is efficient by being lazy.

---

OPEN_MAX = 128 can be a lower limit for the actual max number of open files.

You cannot raise the limit. Only the superuser can raise the limit!

`ls -s` - num blocks.

What are sparse files for?
- sparse files are used in databases, os images, etc.

To detect if a file is a sparse file, just ask the fs the number of blocks
being used.

vnodes are implemented in most UNIX systems.

File locking mechanisms aren't easy and light to implement. UNIX by default doesn't
implement locking mechanisms.

time:
- wall clock
- user space time
- kernel space time
Actual I/O does not get kept track of in kernel or user space.

for synchronous I/O the time for I/O adds up to the wall clock time.

Try SYNC on pipes from either side or both.

GNU du(1) and NetBSD du(1) give different numbers.
- It actually depends on different filesystem.

C standards
- gets() is deprecated since c99 (you just get a warning)

There are resource limits because our computers don't have infinite resources.

why does bash implement /dev/tcp?
- it's an interface for a TCP socket. it makes it easy to write to a port.
- can be used in malware and setting up a reverse shell.
- you can implement a web server in just shell.

security of sparse files
- not sure of any direct security flaws

Is EOF automatically added to the end of files?
- files don't have to end in newline. it is only UNIX convention.
- EOF is defined as some special value when you reach the end of the file. or by the terminal driver with ^D.

Everything is a file
- we covered the 5 basic syscalls. they work on most things. even dirs are files that map inodes to names.
fifo, sockets, char devs, block devs.
- NOT COMPLETELY TRUE because some shells implement pipes as sockets.

Sockets
- UNIX domain - doesn't go through the tcp or udp stack but it still has the same api.
- network domain

Plan 9 OS - rob pike
- took everything is a file much further. window system uses file i/o.

O_EVTONLY means I don't want to be bothered to do I/O when there's nothing there. Only react to events.
O_DIRECT write to fs without using buffer cache. Linux has this
O_NOATIME don't touch the last access time.
O_TMPFILE create a temporary file that only one process has access to. create dir, create file, open, unlink file.
- the flag guarantees the operations are done in one atomic step.
O_ASYNC async io.
O_REGULAR must open a regular file. instead of calling stat to check if it's regular then calling open, this does it atomically.
O_SHLOCK atomically acquire a shared lock.
O_VERIFY kernel will verify the file before it's open. used for the runtime linker. verification is implementation specific.

---

for homework:
we get a test script and we should always consider "what can go wrong?"



