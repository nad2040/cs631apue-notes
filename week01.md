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
