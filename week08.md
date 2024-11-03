# IPC

Sync vs Async
- read and write timing

Unidirectional vs Bidirectional
- one way or both ways

Related vs Unrelated
- must share ancestor

Network communication


# Means of IPC

Using files on FS:
- async
- bidirectional
- related and unrelated
- variable data

Signals
- async
- unidirectional
- related and unrelated

Semaphores
- async
- unidirectional
- related and unrelated

Shared Memory and Message Queues
- async
- bidirectional
- related and unrelated
- variable data

Pipe
- sync
- unidirectional
- related
- variable data

FIFO
- sync
- unidirectional
- related and unrelated
- variable data

Socket Pairs
- sync
- bidirectional
- related
- variable data

Sockets
- sync
- bidirectional
- related and unrelated
- network
- variable data

# System V IPC (Semaphores, Shared Memory, Message Queues)

Async IPC:

All 3 use IPC structures, referred to by an identifier and a key.

We need special system calls (msgget, semop, shmat) and special userland
commands (ipcrm, ipcs)

## Semaphores

```c
semget();
semctl();
semop();
```

Use `ftok` to create IPC id from pathname.

Use `initsem` to initialize a semaphore.

Use `semop` to grab a permit or release based on sem_op.

`ipcs` reveals information about the semaphore.
there are permissions, owner and group for the semaphore.

only the owner may remove the semaphore set

`ipcrm` to remove the semaphore set.

## Shared Memory

reduce the amount of times we switch between userspace and kernel space.

fastest form of IPC

access is often controlled by semaphores

`shmget` obtains shared mem id
`shmat` attaches shared mem segment to address space
`shmdt` detaches it
`shmctl` for other ops

process can terminate but shared memory stays shared.

`ipcs -m` shows info about it.

LPID is last process that accessed the shm.

Printing the pointers of shared memory shows that it is between the stack and the heap.

Try to find the limits of shared memory and what happens when you run out.

## Message Queues

Linked list of messages stored in kernel space.

`msgget`
`msgsnd`
`msgrcv`
`msgctl`

message can be a user defined structure.
first element must be a long defining message type.

`ipcs -q` shows queues

shows data Last Submitted PID and Last Read PID.

So if you don't have write perms on the mqueue you can't write to it.

if two processes are waiting to read a message from the queue,
the consumers are waiting in order.

## POSIX Message Queues

`mq(3)` provides a realtime IPC interface
- don't need ftok. named identifier may appear under /dev/mqueue
- `mqsend(3)` and `mqreceive(3)` allow blocking and non-blocking calls
- allows priority of messages.
- `mq(3)` provides async notification `mqnotify(3)`

need to link `-lrt` realtime lib

# Pipes and FIFOs

```c
int pipe(int fildes[2]);
```

Need to fork to get IPC with pipes.

You need to close the right ends so there's only a unidirectional communication.

PAGER env var is what is used to show stuff like man pages. Default is `more`

we can arbitrarily set `argv[0]` with exec.

popen creates a pipe connected to command and returns FILE *.
Usually implemented with a socket

command is passed to `sh -c`. This is dangerous if you aren't careful

`popenve` is nonstandard on NetBSD but allows you to pass an environment.

you can pass an arbitrary command sequence to the shell

NEVER USE `popen` and always verify input.

```c
int mkfifo(const char *path, mode_t mode);
```

named pipe. it gets a file on the filesystem.

How does tee work?
- tee outputs to a file and passes it on.



pipes require ancestor
not line buffered
can have multiple readers and writers

read will see EOF once the write end closed and all data has been read.
write will cause SIGPIPE if the read end is closed. if caught or ignored,
write sets errno to EPIPE.

---

Week 08 Checkpoint

Provide three examples of IPC that allows for asynchronous communication between unrelated processes:

- signals, shared mem, message queues

What is an advantage of using Shared Memory over a pipe? What's a disadvantage?

- Shared memory is the fastest IPC method. You need to use concurrency
  primitives to coordinate multiple processes trying to access the memory.

What is an advantage of using Message Queues over Shared Memory? What's a disadvantage?

- Message queues implement priority for you but are slower than shared
  memory because there is extra work that needs to be done.

What's the difference between a FIFO and a pipe?

- A FIFO is a named pipe. It gets a path on the filesystem.
  You can write and read from it in unrelated processes.
  Pipes on the other hand must be handled between related processes.

popen(3) makes it easy to set up IPC with a new process. What is a risk in using this library function?

- It allows you to pass any arbitrary shell code to the /bin/sh interpreter.
  This arbitrary code could be set to anything and it won't be checked, so you
  could remove all your files.

---

Is directory creation atomic in POSIX

Some system daemons use pid files to assume whether they are running.

For temporary files, use a temp directory and then put files in there.
Recommend against temporary files.

pub-sub (publication and subscription)
- producer-consumer

Networking
Socket
Async I/O
Queueing
Distribution
Load Balancing

If you happen to be the sysadmin for a school for cs students, you need to clean up your
IPC.

By logical design we probably just want to reuse semaphore then we will remove it.

2001-2006 Stevens sysadmin. we had a CS pubnix.

been 4-5 years since the last pubnix.

We used to have a HPC with 65 machines...

Shared memory persists across muliple process lifetimes.

msg struct data field max isn't 512.
MSGMAX is the system limit for the body.

The high priority can break if you aren't busy looping.


Why does init not log to dmesg?
- dmesg is the kernel buffer.
- syslog can be used to send logs to dmesg buffer but since init is a shell script runner,
  those things don't really go to kernel buffer.
- systemd is a different beast. it is different from rc and init.

Diff between reentrant, thread-safe, async-signal-safe?
`man 7`
- async-signal-safe - safe to resume wherever they are.
- thread-safe - coordinate between threads using concurrency primitives.

Why does SIGWINCH exist if there isn't even a window manager?
- it was added in BSD 4.3 about 1986. X window system was created around 84/85.
  Diff UNIX versions may or may not come with a window manager.

Are message queues full-duplex?
- This concept doesn't really apply.

Since the pipe is shell-specific, it can be implemented in any way.

Async network communication coming up!

What is __DARWIN_UNIX03 for?
- on macOS, `union semun` is defined.

System V said the IPC should be OS-specific.

How does shmdemo work?
kernel details...

don't choose popen because you can easily hijack environment variables.

One way to avoid popenve is to use posix_spawn.

shquote will format the strings to make it safe for your shell



linux really only uses posix ipc nowadays.

can be observed using lsof and lsipc.


