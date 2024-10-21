# Login Process

`dmesg -t` print boot messages

then you get daemon boot and login prompt in serial console

`getty` is responsible for configuring the tty you login to.

the getty exec's login after username and password is entered and login exec's shell.

Login process:
- init
- getty (all superuser privilege)
- login
    - turn off echo on terminal
    - getpass(3) to get your password. hash, then compare.
    - register login with systme databases (`w` and `who`).
    - read/display various files (like motd)
    - initgroups setgid
    - chdir to new home
    - chown terminal device
    - setuid and then invoke shell

# Process Groups

Every process belongs to a pgroup. A collection of processes within the same job or terminal

```c
pid_t getpgrp(void);
pid_t getpgid(pid_t pid);
```

each pgroup may have a leader. leader pid = pgroup id.
leader can create a new pgroup and new processes in the same group

a process can set its or its children's process group using `setpgid(2)`

Session - A collection of one or more process groups.

```c
pid_t setsid(void)
```

If you are not the pgroup leader, this function creates a new session.
- This process becomes the session leader of the new session.
- you become the pgroup leader of a new pgroup
- process has no controlling terminal. (to get one, you will need to request one via ioctl or open one)

Foreground groups may receive keyboard input and keyboard signals.
hangup from terminal will be sent to session leader.

`ps -o pid,ppid,pgid,sid,comm | ./cat1 | ./cat2`

shell pipeline spawns all three processes first.
pipeline gets new group but shell's group and this group are under the same session.

process groups are used to implement job control in the shell.
processes in the same group as the terminal are foreground and may read.

# Job Control

shell forks and then sets pgid of child to the child's pid.

when background processes finish, they send SIGCHILD, and then the shell should go check up on the background
processes and wait for it.

the login shell must have called tcsetpgrp to set the pgroup for the controlling terminal.

I/O interacts with the foreground process group. Generated signals get sent to the foreground groups.

`stty tostop` blocks output of background processes and makes it send a signal SIGTTOU.

SIGTTOU for output from background processes.
SIGTTIN for input to background processes.

being able to handle these signals is called job control.

`jobs -l` lists all jobs.

`bg %n` moves job n into background
`fg %n` moves job n into foreground

`kill -TSTP pid`
`kill -CONT pid`

You can kill individual commands in a pipeline, and that will impact the other processes in the pipeline.
Earlier processes will receive SIGPIPE because the readend disappeared.
Later processes will encounter EOF when the previous commands fail and close the pipe.

# Reentrant and interrupted functions

calling buffered I/O (`printf`) in a signal handler:
you may get output interleaving if you intend to separately print things without a newline to a terminal.
Use write syscall to get the correct order of output.

getpwnam() in an infinite loop with alarm(). alarm just sends a signal every n seconds.
you can get many sorts of failure modes: segfault, corrupted value, etc.

all things can go wrong if you don't call async-signal-safe functions in signal handlers.

Check manpages for async-signal-safe functions guaranteed by POSIX.

Some functions are even guaranteed to be atomic. Kernel cannot switch in until it completes.

Reentrant functions. Multiple invocations may safely run concurrently. Not necessarily thread-safe.

Many library functions set errno, so you should probably save errno in your sig handler and reset it at the end of the handler.

Blocking functions:
- read(2) from pipes, stdin
- write(2)
- open(2) on a device that waits until a condition occurs (like modems)
- pause(3)
- certain ioctls, IPC, file- or record-locking mechanisms.

interrupting a blocking call
- you return unsuccessfully with errno EINTR
- may return with data transfer shorter than requested
- syscall may automatically be restarted after sighandler returns.

Action depends on interface and how the handler was installed (SA_RESTART flag for sigaction).

Much of this is OS specific, so check the manpages.

---

# Checkpoint

Consider the login session:

```sh
$ echo $SHELL
/bin/sh
$ ls -R / | wc -l &
[2] 4526
$ date
Thu Oct 14 14:06:54 EDT 2021
$ jobs
[1] Stopped           find /
[2] Running           ls -R / | wc -l &
$ ps | sleep 30
```

How many process groups are there?
Which, if any, interact with the controlling terminal?

- If I understood correctly, 1 for the shell + 1 for each job. `ps | sleep 30` is another job.
  So in total 4 process groups. Only the foreground jobs directly interact with
  the controlling terminal input, so `ps | sleep 30`.

Common signals you may encounter include INT, KILL, SEGV, QUIT, or TSTP.
Identify three other signals and describe under what circumstances they might occur:

- SIGHUP (Hangup)
          This signal is generated by the tty(4) driver to indicate a hangup
          condition on a process's controlling terminal: the user has
          disconnected.  Accordingly, the default action is to terminate the
          process.  This signal is also used by many daemons, such as
          inetd(8), as a cue to reload configuration.
    - I came across the `nohup` command when I was first learning shell commands.
      I guess terminal multiplexers do a similar thing to keep your tty's alive when
      you log out. I know tmux uses a server that doesn't die when you log out.

- SIGILL (Illegal instruction)
          This signal is generated synchronously by the kernel when the
          process executes an invalid instruction.  The default action is to
          terminate the process and dump core.  Note: the results of executing
          an illegal instruction when SIGILL is blocked or ignored are
          formally unspecified.
    - I've heard debuggers use illegal instructions to implement breakpoints.

- SIGPIPE (Broken pipe)
          This signal is generated by the kernel, in addition to failing with
          EPIPE, when a write(2) call or similar is made on a pipe or socket
          that has been closed and has no readers.  The default action is to
          terminate the process.
    - I'm don't know of any special use cases for sigpipe.

What's the difference between blocking a signal and ignoring it?

- Blocking a signal means you want to finish doing something and process it later.
  The kernel holds it for you.
- Ignoring a signal means you want to handle the signal and do nothing

Run the program reentrant.c (repeatedly!) on different Unix versions (e.g., NetBSD, Linux, macOS).
What do you observe? Do they behave differently?

- On NetBSD, I only seem to get user _ not found! and sometimes the
  SIGALRM prints are separated by two newlines.
- On MacOS, SIGALRM gets printed and then it just hangs. with `repeat 10 (./a.out; echo)`
  I see two SIGALRM printed. Sometimes running a.out itself prints SIGALRM
  3 times. Looks like it hangs on the first a.out call.
- On Linux, again, it doesn't seem to get past the first a.out call. Weird...
  I'll consult the manpage at some point.

Please note any particular (sub)topics or aspects of this week's materials that you would like me to revisit in class:
