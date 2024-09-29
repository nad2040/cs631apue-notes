# Week 4

## Disks and UNIX FS

`sudo disklabel disk`

- physical block size
- partition table

in each partition we can create a new fs.

A directory entry is really just a hardlink mapping filename to inode

No hardlinks across FSes
Inodes contains most of the information found in struct stat

Every inode has a link count (st_nlink) showing how many things
are pointing to it. If the nlink is 0, the block can be freed.

Moving a file is just creating a new entry and deleting the old one.

Root . and .. are linked back to root.

## Hardlinks and Symlinks

```c
link(const char *path1, const char *path2);
```
creates hardlink to an existing file and increments nlink.
POSIX allows cross FS hardlink, but it is not allowed across systems.

Only euid 0 can create links to directories (loops in FSes are bad).

```c
unlink(const char *path);
```
opposite of link

if nlink is 0 after decrement, free the blocks unless a process has it open.

Data is never "removed". Only the links are unlinked.

```c
rename(const char *from, const char *to);
```

if from refers to a file:
    if to exists and isn't dir, remove to and rename from to to
    if to is a directory, fail
    must have w+x perms for the directories containing from and to.

if from refers to a dir:
    if to is an empty dir, remove it and rename from to to
    if it isn't a dir, error
    w+x perms
    if from is a prefix to to, error

```c
symlink(const char *name1, const char *name2);
```

recall l family of syscalls

Symlink loops are detected.

if you create a link to a parent, you can link the repeated subdir traversal

How does ls -l show the target of the symlink when it should be dereferencing
the link?
```c
readlink(const char *path, char *buf, size_t bufsiz);
```

buf is not null terminated.


## Directories

```c
mkdir(const char *path, mode_t mode);
rmdir(const char *path); // only if the dir is empty.
```

```c
opendir()
readdir()
getdents()
```
order is opaque

file descriptor limits may or may not apply to directory streams.
- See COMPAT in opendir manpage

for filesystem traversal, use `fts(3)`.

to remove a directory we need r+x perms

```c
chdir(const char *path);
```

cd is a builtin! and it HAS to be a builtin!

POSIX requires a standalone utility called cd to exist. even though it isn't useful
because you fork a process when calling it.

Directory sizes

fifo and socket: file size of 0
regular files: count the bytes in the file
symlink: the length of path it points to. no carriage return
devices: device major and minor numbers

Creating the newfs shows the different owners and link counts but the same
inode number for . and ..
This is because they exist on different devices!

Directory size only increases when there needs to be more blocks to store
all the entries. Removing files doesn't deallocate blocks for the dir.

some filenames are too long. filename can be 255 characters long.

`hexdump -C .` doesn't work on all systems

4 bytes for inode in little endian
2 bytes of dirent/record length. (minimum seems to be 0x0c = 12 because of 4-byte alignment)
1 byte for type of file: e.g. 0x04 for dir
1 byte for length of filename: so this is where we get the limit of 255.
filename with some padding depending on the 2 byte dirent length field.

The last entry takes all the remaining space in the block.

Removing dirent doesn't modify the bytes. It just expands the dirent length
of the previous entry to consume the next entry. This is why the directory doesn't
shrink.
If we create a new file, it will find the extra space in the directory that just
expanded.

if we remove that file and create a new one, we see a potential reuse of inode.

Other FSes might be different!

## /etc/passwd getpwuid /etc/groups

passwd entry:
username (pw_name)
hashed password (pw_passwd put in another file for security)
numerical uid (pw_uid)
numerical gid (pw_gid)
comment (pw_gecos - general comprehensive OS)
initial wd (pw_dir)
initial shell (pw_shell)

It is stored in plaintext with one line per entry and colons separating
the fields. (none of these things can contain a colon)

In early days, gecos used to include fullname, phone numbers.
nowadays, only the fullname. It is useful for emails as it is pasted into the `from` field

root and toor are just two names with the same uid. it is useful for rescue.

if the account doesn't have a passwd, the asterisk for hashed passwd won't even exist.

login shell can be set to any executable. /sbin/nologin is used. it returns false.
you can set it to /bin/date
if you leave it blank, the system defaults to /bin/sh

initial wd is usually your home directory, but leaving it blank is okay and will
make your initial working directory root (/).

gecos field allows for expansion of & to the capitalized username.

having multiple entries for the same username is not normal. and is probably a bad idea.

Question: does it print the root username because that comes first in the passwd entries?
- yes

toor is mostly for BSD.

login command is a different way to login and it's what the system does.

finger command can be used to find out information about a user.

Why Charlie & -> Charlie Root? UNIX history.

```c
getpwuid(uid_t uid);
getpwnam(const char *nam);
```

On BSD, hashed password is in /etc/master.passwd

for systems using /etc/shadow
```c
#include <shadow.h>
struct spwd *getspnam(char *name);
```

reading all entries sequentially in some order: `getpwent(void)`
`setpwent()` rewinds stream to beginning
`endpwent()` closes stream.

The asterisk is given to you by getpwnam based on euid.

Groups:

group name - gr_name
hashed password - gr_passwd (not in POSIX)
numerical GID - gr_gid
group members - gr_mem (char **)

Group passwords are hardly ever used. supplementary group membership sufficiently solves the problem.

```c
#include <grp.h>
struct group *getgrgid(gid_t gid);
struct group *getgrnam(const char *name);
getgrent(void);
setgrent(void);
endgrent(void);

int getgroups(int gidsetlen, gid_t *gidset);
int getgrouplist(const char *name, gid_t basegid, gid_t *groups, int *ngroups);
```
POSIX doesn't define whether getgroups(2) includes the egid

calling getgroups with gidsetlen == 0 doesn't modify gidset
getgrouplist(3) is non-standard

id(1) and groups(1) are actually the same executable but checks argv[0] to decide how to work.

newgrp command makes a new shell with you in the group.

Other system databases:
- /etc/hosts: hostent
- /etc/networks: netent
- /etc/protocols: protoent
- /etc/services: servent

## time (atime, mtime, ctime)

last (data) access time
last modified time
last file status change time

default ls uses mtime
-t is mtime
-u is atime
-c is ctime

creating a new file sets all times equal. reading updates atime. appending updates mtime and ctime.

creating a second hardlink increments st_nlink, so only ctime gets updated
changing permissions only modifies ctime because st_mode gets updated.

touch command updates a file's atime and mtime. need read and write perms.

touch -t 197001010000 file should set time to epoch. we have write perms but aren't the owner, so fail.
if we are the owner, even birthtime gets updated because the mtime is updated to a value before the current birthtime.
ctime gets updated when we update the other times.

touch -a for atime
touch -m for mtime

updates to atime as a side effect of read or other actions doesn't cause an update to ctime.
intentional modification of atime (e.g. through touch) will update ctime.

sed -i doesn't edit inplace. it creates a new file and updates the filename. all the times get updated,
including birthtime

FSes have an option -o noatime to disable atime updates altogether. noatime, nodiratime, relatime, lazytime

some progs use the mtime and atime, like mail, so we shouldn't get rid of atime everywhere.

On Linux, it seems that reading the file doesn't update the atime.

relatime (relative atime) atime is only updated if the mtime or ctime is newer than the atime
or if the atime is older than 24hrs.

```c
#include <sys/time.h>

utimes(const char *path, const struct timeval times[2]);
lutimes(const char *path, const struct timeval times[2]);
futimes(int fd, const struct timeval times[2]);
utimensat(int fd, const char *path, const struct timespec times[2], int flag);
```
array includes atime and mtime
if `times == NULL`, you set atime and mtime to current time. w perms sufficient
if `times[i] != NULL`, set accordingly, but need to be owner.

ctime is set to current time.

timespec allows for nanosecond precision.

Time is an illusion.

```c
#include <time.h>

time_t time(time_t *tloc);
```

value of time since unix epoch

`time(3)` is a library function, not syscall, so where does it come from?

`<sys/time.h> gettimeofday(2)` is what's called on NetBSD

POSIX says use clock_gettime
```c
#include <sys/time.h>

int clock_gettime(clockid_t clock_id, struct timespec *tp);
```
gets seconds and nsec since epoch

Breaking time using `gmtime(3)`

```c
struct tm *gmtime(const time_t *clock);
```
converts t to UTC.

struct tm stores:
- years since 1900??
- days from 0-365 (to support leap day)
- seconds after the minute 0-61 (to support leap seconds)

International Atomic Time (TAI) leap seconds help deal with the inaccuracies of atomic time.

To date (2020) 27 leap seconds since 1972.

POSIX says time monotonically increases and shouldn't bother with leap seconds.

Other systems only allow seconds to up to 60 (so 1 leap second). Up to 61 allows a double leap second.

`char *asctime(const struct tm *tm);` gives you the string representation of the time.

now we get into timezones, daylight savings time, offset from UTC, etc.

`struct tm *localtime(const time_t *clock);` gets you the local time struct tm. Checks TZ env var. and system timezone.

/usr/share/zoneinfo has timezone information. It is managed by IANA (Internet Assigned Numbers Authority)

Making your own time:

```c
time_t mktime(struct tm *tm); // reverse direction of gmtime and localtime.
char *ctime(const time_t *clock); // string of time but with time_t not struct tm.
```

Only sane time representation: ISO-8601 +%Y-%m-%dT%H:%M:%SZ
YYYY-MM-DDTHH:MM:SSZ

```c
#define ISO8601_LENGTH 20 + 1 // +1 for \0
```

no tzinfo. ZULU time stands for UTC...

`ssize_t strftime(char * restrict buf, size_t maxsize, const char * restrict format, const struct tm * restrict timeptr);`





