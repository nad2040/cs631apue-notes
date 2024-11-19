Formats:
assember output (a.out)
common object file format (COFF)
Mach-O - macos
ELF - Linux switched to ELF as default in around 1998.

ELF is used for binary files such as:
- executables
- relocatable object files (.o)
- core files (.core)
- shared libraries (.so) - position independent code.

User list of paths is in LD_LIBRARY_PATH. ld.so.conf shouldn't be used.
Then check DT_RPATH and DT_RUNPATH
Then the default /usr/lib

readelf -d shows shared libraries needed.

ktrace -i ./a.out - traces syscalls
kdump - prints it out

if you remove dynamic linker, then you use ld64.so
if no-dynamic-linker, no interp header, you just get segfault.

at link-time, resolve undefined symbols and get static libraries.
at execution-time, dynamic libraries.

archives usually end in .a
effectively a single file containing other object files.
linking statically pulls in all the code from the archives into the executable
```sh
ar
```

```sh
cc -c -fPIC impl.c -o impl.o
cc -shared impl.o -o libimpl.so
```

environment has precedence over RPATH

but without LD_LIBRARY_PATH, we can use -rpath

`cc main.o -L./lib -lldtest -Wl,-rpath,./lib`

if an suid root executable could use your library code, you
could easily get root access. So, if it is suid, the system
ignores LD_LIBRARY_PATH. If not, it is honored.

Dynamic loading at run time without linker.

```c
#include <dlfcn.h>
dlopen();
dlsym();
```

---

What is LD_PRELOAD?

What is LD_DEBUG?

---

Compare the ELF headers for a simple executable and a shared library -- what's different?

- I ran readelf -h on /usr/lib/libc.so and found a different type (DYN (Shared object file))
  and a different entrypoint address. There are a lot more section headers in libc.so as well.
  With readelf -l I see TLS and GNU_RELRO are headers for the shared library.

How many different machine types (e_machine) are defined on your NetBSD system?

- Elf32_Half is 16 bit and in /usr/include/elf.h, 0 to 243 are all defined in the elf.h header.
  Some are reserved though. My vm only supports aarch64.

What is the purpose of a run-time link-editor?

- It is used to find and load the shared objects of a given executable.
  It uses the different ELF sections and the different sources (LD_LIBRARY_PATH, /usr/lib, etc.)
  to find the location of the libraries. It then resolves symbols and loads libraries into the process
  image.

What are advantages and disadvantages of each static linking versus dynamic linking?

- Static linking
    Advantage: faster startup, self-contained
    Disadvantage: bigger executable size, can't change implementations without
     rebuilding the whole project
- Dynamic linking
    Advantage: smaller executable size, allows for hot reloading code.
    Disadvantage: slightly slower startup time as the run-time link editor
           needs to find the libraries and load them into the process image.

Complete the libgreet exercise. What command did you use to create the shared library?

- I used the following commands to 1) build the shared lib and 2) use the shared lib.

first compile with position-independent code flag
and then make it a shared object.

cc -c -fPIC libgreet/greet.c -o libgreet/greet.o
cc -shared libgreet/greet.o -o libgreet/libgreet.so

cc hello.c -o hello -lgreet -I./libgreet -L./libgreet

---

Questions

Are there any security concerns for having /lib64/ld-linux-...so executable
despite the executable file not having execute permissions?
- it's okay because you still need read permissions.
  and if you have read permissions, you can just give yourself a
  copy and add them back.

The ld.elf_so is not executable.

sudo uses libcrypt so changing the actual implementation can fuck your
implementation up.

How does ELF store debug symbols?
- just use another section to store all the debug info.

tar used to be tape archives. but then it started supporting many different archive formats.

Reading the larger executable for static link. The practical difference between static
and dynamic link libraries is negligible. Also remember the sticky bit! That was useful
back then when hardware was slower.


Modern executables use rip-relative addresses for jumps. This is basically how PIC can
work.
