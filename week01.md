# Week 1

What is this class about?
- gain an understanding of UNIX operating systems
- gain systems programming experience
- understand fundamental OS concepts

Comment the why, not the how.

Read through the style guide

Question about formatting:
- can I use clangformat formatter? and is there an already existing clangformat file that matches your style preferences?
- what's the linting workflow like? what specs?

Answers to above:
- found a [basic clang-format configuration](https://github.com/NetBSD/src/blob/9b7af74de2474e659ae4504bab7d5cd205a5ef1b/share/misc/dot.clang-format)
- all the rest need to be done manually I think.

BSD History can be found under /usr/share/misc/

I probably won't take notes on history.

Notes for Unix Basics

Manual pages: `man [page] [name]`
1 - General commands
2 - System calls
3 - C library functions

POSIX spec [POSIX 2024](https://pubs.opengroup.org/onlinepubs/9799919799/)

C Programming:
- Unix best practices
    - 0 exit status = success
- Use errno!
- Use Wall Werror to make bad code not compile
- Use sysexits!

"Consistency underlies all principles of quality." - Frederick P. Brooks Jr.

Unix programs
- are simple
- follow the element of least surprise
- read from stdin and write to stdout
- meaningful error messages to stdout
- meaningful exit code
- have a manual pages
...

Fun fact:
- Unix predates the Unix epoch

system time + user time != clock time

blocking time is not counted

-----

## During Lecture

2-8 KLOC

post quantum encryption
https://en.wikipedia.org/wiki/Kyber

If something is obviously wrong, Jan will step in, but for the most part he will wait in the background.

Notes aren't 100% necessary but they are still useful.

Fun facts:

Old unix keyboards had hjkl with arrow keys.
And tilde is the home key.

This is why vi has hjkl and tilde as home directory.

The reason getlogin compiles is that the compiler guesses the prototype. It guesses that it returns an int.
It segfaults because printf wants to print a string, but we got an assumed int.

Headers give prototypes. C files have definitions.

Question: why does printf cause the compiler to print stdio.h in warning. but getlogin doesn't print unistd.h?
- probably a compiler feature

Style differences compared to BSD
- if blocks with one statement

Look up: "Goto fail security vulnerability"

---
## Questions

[Linus Torvalds vs Andrew Tannenbaum](https://groups.google.com/g/comp.os.minix/c/wlhw16QWltI?pli=1)
- Linus built Linux.
- Tannenbaum built MINIX.

USENET is kinda like Reddit back then.

Tannenbaum said Linux is OBSOLETE. The USENET was archived. And now it's on google groups.

At the time, Linux was only designed for i386.

NetBSD is famous for being portable. It has one code base.

Linux works on many more machines but also many different patches.

Linus Benedict Torvalds

---

What do all the things under / mean?

/altroot - useful for swap roots

in the old days disk space was small, and different binaries were
on different disks.

/bin - basic user commands (coreutils)
/sbin - system binaries

/usr/bin -
/usr/sbin -

/usr/local/bin - tools that don't belong to the OS. manually installed
/usr/local/sbin -
/usr/pkg/bin - tools installed by the package manager.
/usr/pkg/sbin -

/boot - tends to be boot partition
/cdrom - mountpoint for CD

/dev - device/fds

man hier - describes filesystem

---

`find | xargs` is faster than just `find -exec`.

---

In buffered I/O, where are buffers are stored?
But it depends.

---

why is ifndef included for simple-cat for BUFFSIZE?

Well, BUFSIZ is defined on systems in the library.

but for BUFFSIZE, we use ifndef because we can do this:
// compile time option for setting macro values.
cc -DBUFFSIZE=512 simple-cat.c

---

why is forking and cloning the process the only way to
start a new process?
- UNIX way is ONE WAY to make a new process.

Modern OSes use Copy-on-Write when forking.

---

How to program in C in a very safe manner in the age of
automated (and sometimes AI powered) fuzzing and vulnerability
searching?
- well fuzz your own program first
- test driven programming

---

exit is a built-in

in some cases [ and test may be a shell built-in
not necessarily executables

---

advantages of function prototypes.

---

K&R just said that functions without prototypes will probably
define everything as int as default.

libc >c99 allows a fall off without a return 0 in main.





