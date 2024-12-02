# syslog(3)

syslogd(8)

do one thing and do it well

central logging interface.

- kernel can use log(9)
- userland can use syslog(3)
- UDP to port 514

```c
#include <syslog.h>

void openlog(const char *ident, int logopt, int facility);
void syslog(int priority, const char *message, ...);
```

prepend ident to each message
opts: `LOG_CONS | LOG_PERROR | LOG_PID`
facilities: `LOG_DAEMON, LOG_MAIL`

priority is facility + level
log levels: `LOG_DEBUG, LOG_WARNING, LOG_EMERG`


editing /etc/syslog.conf:
- matches higher levels

logger(1)

emergency gets broadcast to all logged in users in the terminal.

syslog was big in 1980s

documented RFC3124
standardized RFC5424

various newer versions like syslog-ng and rsyslog with (TCP, TLS)

large scale message relay services (elasticsearch, solr, flume, fluentd)

# Non-Blocking I/O

errno would be set to EWOULDBLOCK or EAGAIN.

open or call fcntl with
O_NONBLOCK

writing into the pipe:
we reach the limit of 64k.

writing into socket:
16k chunks
TCP buffers data and network I/O may at times block

# Resource Locking

ways to exclusive access.
- create | excl and then unlink
- create a lockfile
- use a semaphore

```c
int flock(int fd, int operation);
```

add advisory locks on files.

operation can be LOCK_NB and
LOCK_SH (shared)
LOCK_EX (exclusive)
LOCK_UN for unlocking

locks apply to the entire file.

flockfile(3) can be used to lock stdio streams.

all locks are released on termination.

fcntl record locking

F_GETLK, F_SETLK, F_SETLKW

struct flock allows you to specify an offset.

this allows you to lock a region of a file.

lockf(3)
- library func for locks a region given offset and size.

Locks are
- not inherited across forks
- inherited across exec unless close-on-exec is set for fd.
- release on termination
- release if a fd is closed
    - remember a lock is specific to a file and process pair. not fd.

You need to keep track of all the library functions that make any changes on the fd.
This is problematic and manual page warns about it.

"Mandatory" Record Locking
- some systems overload group permissions (setgid but no group exec)
- problems: lock on file doesn't prevent removal and rename
- basically don't work. do not rely

can you lock regions past the end of a file?

how do flock and fcntl locks interact?

# Asynchronous I/O

wait for a signal!

select/poll were asynchronous blocking i/o

AIO is async non-blocking

System V had true async i/o but it was limited to STREAMS (never adopted and now obsolete)

BSD async I/O:
- terminals and network only
- enabled by open/fcntl O_ASYNC, F_SETOWN
- uses SIGIO and SIGURG

POSIX AIO aio(7)
- kernel manages queued I/O events
- notification of a calling process happens via signal or sigevent callback function.
- choice to block

glibc aio
libaio

# Memory Mapped I/O

```c
#include <sys/mman.h>

void *mmap(...)
```

mmap maps len bytes of data into a region.

protections
- PROT_READ = can read
- PROT_WRITE = can write
- PROT_EXEC = can exec
- PROT_NONE = cannot be accessed

Specify MAP_SHARED or MAP_PRIVATE

memory mapped i/o should be better than normal read/write.

cp(1) uses mmap but not for all I/O. when/why do we use mmap?

---

Checkpoint

Describe each one log level and facility not mentioned in the videos:

- LOG_CRIT      Critical conditions, e.g., hard device errors.

- LOG_CRON      The cron daemon: cron(8).

Use the nonblock.c program to write data in non-blocking mode (./a.out
nonblock) into a pipe, a fifo, and over the network (e.g., via nc(1)).
Describe and explain your observation:

- pipe - i see the number of bytes fluctuates, but 65536 is the max.
  write error: Resource temporarily unavailable
- fifo - i see 8192 and a lot more write errors this time.
- socket - using `netcat --exec` on macOS, I don't see any errors.
  piping output into nc on NetBSD, I see it freezes afer a while.
  Max is still 65536 but errors seem more frequent.

Rewrite flock.c to use fcntl(2).
Note any observations here:

- I had to manage the flock struct and set the l_type field to
  change the lock. I'm not sure if I did things correctly as after
  running a second instance a few seconds after, the first just kept
  printing Unable to get an exclusive lock, even after the first one
  was done with the shared lock.

Give a practical example of when it might be useful to use asynchronous I/O:

- It's useful for accessing resources over the network or generally for things
  that take a long time to access, like databases. Some programming languages
  have keywords async await for asynchronous operations.

Review the NetBSD source code for cp(1) - why/when is mmap(2) used here?
Why is it not used for all I/O?

- mmap is used for files smaller than 8MB. We don't want to map too much memory into
the process image.

---

Questions:
At what point does memory mapping get slower than direct I/O operations?
I'm guessing that 8MB was achieved from testing.

---

different systems choose to deal with dualstack differently. BSD has dualstack on by default.
Linux doesn't.
