# Week 5

## Editor

- `:make`
- `:cl :copen` to list errors. copen leaves them open. cclose.
- `:cc [n]` to jump to the error in line n of the `make` output
- `:cn :cp` jump to next or previous error
- nK gives us the correct manpage from section n.
- ^] jumps to definition. ^t brings us back.

## Compiler Toolchain

preprocessor
lexical analysis
syntax analysis
semantic analysis
code generation
code optimization
assembly
linking

Having a second compiler can help you catch non-portable code!

The toolchain consists of
`cpp` - C Preprocessor
`cc` - C Compiler - invokes preprocessor
`as` - Assembler
`ld` - Linker

The preprocessor pastes headers and removes all directives and expands macros, etc.
`cpp -P` to leave out annotations. Convention to output `.i` files.

`cc -E` stop after preprocessor.
`cc -v` verbose details about compilation

`cc -S` yields assembly
- LFB = Local Function Begin
- LFE = Local Function End

`cc -On` for code optimization

`cc -c` yields object files
`strings` to show the valid ascii strings in any binary.

Linking:
`ld .o -lc` for libc. crt0 = C runtime. This has the _start for running C programs.

Instead of using `grep` to search for symbols, use `nm`.
`nm /usr/lib/crt0.o`. T = defined in text section. U = undefined, check elsewhere.
We find `_fini` in crti.o

Still doesn't compile.
`ld.elf_so` is a runtime link editor. Tell ld to use it `-dynamic-linker /libexec/ld.elf_so`?
Still didn't work.

Check what `cc` does with -v.

We need to provide bookend objects for processes. `crtend.o crtn.o`

`ld --verbose` shows the defaults for what ld looks for.

Order of command line flags may matter!

## Project Management (make)

```make
ls: ls.o main.o cmp.o ...
    cc ${CFLAGS} *.o -o ls

ls.o: ls.c
    cc ${CFLAGS} -c ls.c

main.o: main.c
    cc ${CFLAGS} -c main.c

cmp.o: cmp.c
    cc ${CFLAGS} -c cmp.c
```

Suffix rules and variables:
.c.o converts .c to .o
\$< file ending in original suffix
\$@ file ending in destination suffix

```make
PROG=   ls
OBJS=   ls.o main.o cmp.o
CFLAGS= -Wall -Werror -Wextra

.c.o:
    ${CC} ${CFLAGS} -c $< -o $@

all: ${PROG}

${PROG}: ${OBJS}
    @echo $@ depends on $?
    ${CC} ${OBJS} -o ${PROG} ${LDFLAGS}

clean: ${PROG}
    rm -f ${OBJS} ${PROG}
```

Running make with the variables
`make CFLAGS=""`

Builtin variables:
`CC = cc`

Using `mkdep` to handle header file updates.
```make
depend:
    mkdep -- ${CFLAGS} *.c
```

mkdep creates .depend

Final trick: don't need the .c.h suffix rule because that is known by make.

There are different makes. BSD make and GNU make are the main ones.

## Debugging (gdb)

do `gcc -g` for debug info

`bt` for backtrace
`call (cast to ret type)func(args)`
`p` to print values

`b` to breakpoint
ctrl-X o - to step through the program.

`f` to switch frame

Changing values at runtime:
`set var name = value`

Understanding pointers!

- `whatis` to understand the type of a variable
- `x` to examine memory
- `x/c` print as char

`info macro MACRO` shows the value of macros

## Collaboration (diff, patch, cvs, svn, git)


---

Checkpoint:

Identify two steps for each of the compiler's programming language and platform specific stages:

PL-specific:
- lexical analysis (lexing)
- syntax analysis (parsing)

Platform-specific
- assembling
- linking

What stand-alone tools are invoked as part of the compile chain?

cpp(1) - the C preprocessor
cc(1) - the C compiler
as(1) - the assembler
ld(1) - the linker

Identify two common flags passed to the compiler chain that are specific to the compilation stage and two flags specific to the linking stage:

Compilation stage
- `-On` for optimizations at level n
- `-g` emit debugger output.
  - honestly I'm not quite sure if this is the correct stage
- `-fPIC` for making sure your code is position independent (good for libraries).

Linking stage
- `-llibrary` to link liblibrary
- `-L` to add to the linker library search path

Why should we bother creating a Makefile?

To automate the building of the program for us and keep track of dependencies
so that small changes to a huge project doesn't trigger the compilation of
the entire project again. We only want to rebuild the code that we changed,
and nothing else, so we speed up compilation over successive incremental changes.

Consider the (terrible!) program at https://stevens.netmeister.org/631/gdb-examples/pointers.c.
Run it as './a.out foo', then explain the output.

```
argv[0]: ./a.out/
argv[1]:

argv[0]: .
argv[1]: moo

argv[0]: ./a.out/moo

```

Well, p is equivalent to the address of `argv[0]` and
q is equivalent to the address of `argv[1]`. These strings are stored
back to back in memory, separated only by the nullterm.

First we add a / to the end of `argv[0]`. This replaces the original nullterm and
replaces the f in foo of `argv[1]` with a nullterm.

Then we cat `argv[1]` onto `argv[0]`, but `argv[1]` now just sees a null term, so
nothing is concatenated.

`argv[0]` and `argv[1]` are printed.

Then the second char of `argv[0]` is set to nullterm and the first char of
`argv[1]` goes from nullterm to 'm'. The original o's from foo remain.

`argv[0]` and `argv[1]` are printed and we see only the first char of `argv[0]`
and foo for `argv[1]`.

Then we reset the second char of `argv[0]` to be `/` so we can read `./a.out/moo`.

Interestingly, when doing this in gdb, the full path of the executable was expanded.

Other questions:



---


