# Memory Layout

Text is at bottom of memory (RO usually, X):
- all the functions have small addresses.
- S_ISVTX - if set, the OS can keep the text segment in swap space when the program terminates.
    That is, the text segment remains sticky in memory. (saved text!)

Initialized Data (.data)
- static variable initialized to nonzero value

Uninitialized Data (.bss)
- bss = block starting symbol.
- including all static variables initialized to zero
- initialized to 0 by exec

Heap
- shared by all threads, shared libraries, dynamically linked modules.
- grows upwards towards higher addresses

Stack
- grows downwards
- static variables in functions live in the data section.

argc, argv, envp are all at the top.

Stack overflow - too many stack frames

See /proc/self/map - memory map of the current proc.
See pmap.

# Program Startup

C standard wants main(void) or main(int argc, char *argv[]) with int return type. No prototype is defined.
Other implemenatation-defined manner.

When the exec function is called, kernel needs to load and run the program.

Special startup routine, make process space, put command line args in the right place, call entrypoint, etc.

POSIX guarantees `argv[argc] == NULL`

readelf:
entrypoint address doesn't match program output.

program starts at `_start`.

_start populates registers and then calls `___start`

`___start` calls `atexit`, `_libc_init`, and finally main

rdi is first reg
rsi is second reg

rax for return

after main, exit is called in `___start`.

Under /usr/src we can find architecture specific lib/csu (c start up) assembly for C runtime crt0.S

`____start` is found in crt0-common.c

struct ps_strings is used to locate argv and env. `extern char **environ` is set up this way.

particularly, for this system, envp is passed, so main actually takes 3 params.

Use something other than main!
- `cc -e foo` jump to foo instead of main. saved rip is 0x1, so we segfault.
- to avoid segfault, call exit.
- main returns an int that is given to exit.

# Program Termination

in C89, the return value of main is the value of the register. This can be set to anything by functions
called during main. In newer standards, the value is 0.

`objdump -d a.out` to see what is different in C89 vs C11 default
there is a `mov 0, eax` explicitly added.

normal termination:
- implicit return from main
- explicit return from main
- call to `exit(3)`
- call to `_exit(2)` (POSIX) or `_Exit(2)` (C std)
- return of last thread from its start routine
- calling pthread_exit(3) from last thread.

abnormal termination:
- `abort(3)`
- termination by signal
- response of the last thread to a cancellation request

`exit(3)`
- calls functions registered with `atexit(3)` in reverse order of registration
- flush open output streams, then close them
- unlink files created by tmpfile(3)
- call `_exit()`

`_exit(2)`
- terminates process immediately

`atexit(3)`
- register `void function(void)` functions and push onto a stack. `exit(3)` will pop them off.
- useful for cleaning up open files, freeing resources, etc.

to exit without handlers just call `_exit` or `abort`.

# Environment

Process gets access to the environment variable from `___start` on some systems.

```c
char *getenv(const char *);
int putenv(char *name);
int setenv(const char *name, const char *value, int overwrite);
int unsetenv(const char *name);
```

Unset removes the variable. Setting to empty string is different.

`extern char **environ` lives in bss. envp lives on stack.
they both point to the same place.

After setenv, the new value gets put on the heap.

```c
malloc();
calloc();
realloc();
free();
```
see `jemalloc` for more details

After putenv, envp and environ diverge. environ gets updated and the pointers
get copied to the heap.

# Proc Limits and PIDs

`ulimit` is a shell built-in for this shell.

`ulimit(3)` is obsolete. we should use `getrlimit` and `setrlimit`.

There are soft limits and hard limits.

When a soft limit is exceeded, the call may fail or receive a signal and be
allowed to continue. When a hard limit is exceeded, the resource may not
be increased.

A process may change its soft limit <= hard limit.
A process may lower its hard limit >= soft limit.

Changing soft limit also changes hard limit.

Only su can raise hard limits.

Changes are per process. (so it must be a built-in)

`ulimit builtin` gives us soft limit by default. `-H` for hard limit.

Careful:
cpu time limit counts time on cpu, not time sleeping.

PIDs

```c
pid_t getpid(void);
pid_t getppid(void);
```

Certain processes have fixed pids.

- swapper, sched, idle (Linux), system (nowadays) = pid 0 - responsible for scheduling
- init = pid 1 - bootstraps UNIX system, owns orphaned processes (systemd is a popular init on Linux)
- pagedaemon = pid 2 - on demand paging, responsible for Virt Mem system.

`ps -waxo pid,ppid,command`

```c
fork()
```
In the parent, you get the child pid. In the child you get 0.

Child gets copy of parent's file descriptors.
Child resource util stats are reset to 0.

On the shell, if we pipe, the output is not line-buffered.
So we get two copies of the output because the buffer
wasn't flushed even after we fork.

exec family is a frontend for `execve(2)`. they do not return.
v = argv is vector
l = argv is list
e = take envp
p = use PATH to search for file

ignored signals in the calling process are ignored after exec.
caught signals are reset to default.

real UID/GID is inherited. effective UID/GID is inherited unless SUID/SGID.

```c
wait();
waitpid();
wait3();
wait4();
```
suspend until child processes terminate.

pid/3 allow waiting for specific proc or proc group.
3/4 allow for inspection of resource usage.

```c
WIFEXITED();
WEXITSTATUS();
WIFSIGNALED();
WTERMSIG();
WCOREDUMP();
WIFSTOPPED();
WSTOPSIG();
```
pass NOHANG to avoid blocking.

You can't kill zombies. init will wait for orphaned processes. It will reap the zombies.

# Checkpoint

Consider the following program:

```c
int a;
char *s;
char buf[1024];

int
main(int argc, char **argv) {
    int b;
    char *string = "abcd";
}
```

For each variable, identify which memory segment (text, initialized data, bss, heap, stack, high address) it is allocated in.

a - bss
s - bss
buf - bss
main - text
argc - stack
argv - stack
argv[i] - high address
b - stack
string - stack
*string - data

What is the difference between a buffer overflow and a stack overflow?

A buffer overflow usually occurs within a single stack frame. You have
buffer allocated on the stack and use functions that do not perform
bound-checking. This allows you to overwrite values set up for you by
the compiler, such as the return address, frame pointer, stack pointer,
arguments, etc.

A stack overflow happens when you use up too much stack space and you run out.
Usually this happens because you didn't set your recursive base case properly
and you have infinite recursion. It could also happen if you allocate too many
huge buffers on a small stack. That's usually why we store huge and/or dynamically
sized data on the heap.

Go through the Program Startup Exercise.
How similar / different is the Linux startup compared to the one you've seen on NetBSD?

_start is defined in `sysdeps/<arch>/start.S`

it calls `__libc_start_main` in `csu/libc-start.c`

after some macro trickery, it defines `__libc_start_main_impl`. It calls a lot of
setup functions like atexit and then calls `__libc_start_call_main`

In `__libc_start_call_main` under `sysdeps/generic` it just calls exit with the
main function inside.

This is the main difference between the glibc implementation and the NetBSD implementation.


What is the return value of the following program:

```c
int func2() { _exit(0xcafe); }
int func() { exit(func2()); }
int main() { printf("%d\n", func()); }
```

The exit code is the LS byte of the exit status integer. It can be obtained
with the WEXITSTATUS() macro on wait status.

Since func() is called and it calls _exit(), printf() will never happen.

Exit code is 0xfe = 254.

Under what circumstances might malloc return NULL?

ENOMEM - Can't allocate new memory.

What system call does a parent process use to determine a child process's PID?

The pid is given when you fork() and you must keep track of it.

What's your favorite zombie movie?

Please note any particular (sub)topics or aspects of this week's materials that you would like me to revisit in class:

---

static variable does not get reinitialized.
if initialized to 0, it goes to BSS. if initialized to not 0, it goes to DATA.
the initialization must be set when the static variable is declared.

ulimit shows data and stack size limits.

ASLR causes a performance hit for safety. Some systems disable it for efficiency.
Stack canary - if you change it, then you break.
either compilers insert canaries or kernel injects canaries for you.

marking "W^X" for memory (write and execute perms are mutually exclusive)

Don't iterate envp to look for keys. Just use library functions.
Also, when you change the environment, envp continues pointing
to the initial environment while the environ gets copied to the
heap and updated.

vfork() does the Copy-On-Write (COW), but nowadays, systems implement fork as
copy-on-write.

you can find out how big the stdio buffer is by checking how big of a message
causes the flush when stdout is a pipe.

Leaking fds: forgetting to close file descriptors.

CLOEXEC - close fds on exec
