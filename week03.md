# Week 03

## stat(2)

stat - on all files return metadata for the file
lstat - returns data about the symlink itself
fstat - use fd not filename

fstatat - atomically access relative path name outside cwd.

st_dev & st_ino uniquely identifies files across all mount points.
st_rdev - used for certain devices
st_atime - last access
st_mtime - last modified
st_ctime - last file status change
st_birthtime - not required by POSIX - time of inode creation

st_flags - contains additional flags (see chflags and ls -o)

st_mode: 0000000 u:000 g:000 o:000
- regular file
- directory
- char special
- block special
- FIFO
- socket
- symbolic link

fs block size looks like 4K but ls tells us 20 blocks. This is because ls uses 512,
the BLOCKSIZE env var.

df also uses the environment variable BLOCKSIZE when displaying block count.

## uid and gid

Every process has 6 or more IDs associated with it.

who we really are
- real UID
- real GID

used for file access permission checks
- effective UID
- effective GID
- supplementary GIDs

saved by exec functions
- saved set-UID
- saved set-GID

setuid - set the eUID to st_uid
setgit - set the eGID to st_gid

when setuid is exec'd, the saved setuid is set to the eUID. A call to change
the eUID later will only be allowed to change the eUID to the rUID or the saved setuid.

important:
- exec sets the saved setuid and saved setgid
- changing the euid or egid, is allowed to either ruid, saved setuid, or group ids.

ping(1) is a setuid binary. Sending ICMP packets over raw-mode socket, requiring superuser.

copying the setuid file to your own local dir makes the set-uid bit cleared.

trying to chmod u+s still fails because we changed the owner of the file.

setuid bit is only meaningful when the st_uid field is different from the euid of the user executing the command.

wall(1) send messages to everyone else

terminals have group write permissions

wall is setgid and owned by root and under tty group.

Principle of Least Privilege:
- a process should only be given the privileges it needs. not more.

By setting wall to sgid but not suid and owned by root, we avoid root privilege escalation.

```c
setuid(); // always sets all 3: ruid, euid, saved set-uid
setgid();
seteuid(); // only succeeds under the condition above.
setegid();
```

We do not want to allow processes to retain elevated privileges.

How to check if we have the right permissions atomically?
`access(2)` tells us based on the ruid and rgid. It allows suid and sgid programs
to see if the real user could access without having to drop permissions.

access(path, mode):
- mode R_OK, W_OK, X_OK, F_OK (existence)

Summary:
- every process has euid and ruid, like egid and rgid.
- if suid or sgid bit is set, the euid or egid will become that of st_uid or st_gid at execution time
- seteuid() can be used to switch freely, until setuid() converts things irrevocably
- the euid and egid are used for file permission checks,
  but we can use access to check if the ruid has access.

## st_mode and permissions

S_IRUSR, S_IWUSR, S_IXUSR, ...

To open a file, need execute permissions on each directory component of the path.
Read permissions allow you to list the contents of the directory.

To open a file with O_RDWR or O_RDONLY, you need read permission.
To open a file with O_RDWR or O_WRONLY, you need write permission.
O_TRUNC requires write permissions.
To create new file, you need write and execute permission on the **directory**.
To delete a file, same holds true. But you don't care about the file itself.
To execute a file, you need execute permissions, but not read permissions.
  For compiled binaries, this makes sense. For interpreted files, you still need read perms.

Which permission set to use?
1. If euid == 0 (root), grant access
2. If euid == st_uid, check user permission bits.
    If they allow access, grant access.
    Deny otherwise.
3. If egid == st_gid, check group permission bits.
    If they allow access, grant access.
    Deny otherwise.
4. Else, check other permission bits.

Trying to remove a file without the write permission yields a prompt in rm(1),
but it is totally unnecessary.

## chmod(2) and chown(2)


```c
#include <fcntl.h>
chmod();
lchmod();
fchmod();
fchmodat();
```

Changing permission bits on the files requires euid == 0
or euid == st_uid.

S_ISUID - is setuid
S_ISGID - is setgid
S_ISVTX - sticky bit (saved text)

S_IRWXU, S_IRWXG, S_IRWXO

sgid bit is shown as S by ls if the group execute permission is missing.

```c
#include <fcntl.h>
chown();
lchown();
fchown();
fchownat();
```

Changes st_uid and st_gid for file. Generally requires euid 0.
Some SVR4's let users chown their files to anybody.
POSIX allows either, depending on _POSIX_CHOWN_RESTRICTED.

owner or group can be -1 to indicate that it should remain the same.
Non-superusers can change st_gid if
    euid == st_uid and owner == st_uid and group == egid (or one of the supplementary gids).

Recap:
- only root and owner can change permissions of a file
- only root can change the owner of a file,
  but the owner may change the group ownership of the file
- changing file perms and ownership has significant security implications

## umask(2)

When creating a new file, it will inherit:
- st_uid = eUID
- st_gid =
    - Linux - eGID of the process
    - BSD - GID of the directory in which it is created.

```c
#include <sys/stat.h>
umask();
```

It unsets/masks the bits in the umask.

Midterm is to program `ls(1)`

## Union Mount and Whiteout Files

whiteout file - is used to hide a lower layer file when the corresponding upper layer
    file in a union mount has been removed.

All edits happen in the upper layer. Lower layer directories end up in upper layer, but
files inside do not show up. This is a shadow directory.

`ls -W` shows whiteout files
`S_ISWHT(st_mode)`

---

Checkin:

Identify a struct stat member not listed in the slides/videos:
- NetBSD extensions include st_gen, the file/inode generation number.

- I'm not sure what it could tell you about your filesystem. Maybe whether
  files have been deleted or not?

Identify three setuid executables on your system.
What do they do, and why do they need to be setuid?

After using the following command:
`find /bin /usr/bin /sbin /usr/sbin -perm -04000`

I picked the following 3:
/bin/rcmd - the only one in /bin for me.
From the NetBSD manpage:
    rcmd executes command on host.

    rcmd copies its standard input to the remote command, the standard output
    of the remote command to its standard output, and the standard error of
    the remote command to its standard error.

I found in the OpenBSD manpage that it needs to be suid because it needs to use
reserved ports, which regular users lack permission to use.

/usr/sbin/traceroute
From the manpage:
    The Internet is a large and complex aggregation of network hardware,
    connected together by gateways.  Tracking the route one's packets
    follow (or finding the miscreant gateway that's discarding your
    packets) can be difficult.  Traceroute uses the IP protocol `time to
    live' field and attempts to elicit an ICMP TIME_EXCEEDED response from
    each gateway along the path to some host.

    The only mandatory parameter is the destination host name or IP number.
    The default probe datagram length is 40 bytes, but this may be
    increased by specifying a packet length (in bytes) after the
    destination host name.

For a similar reason to ping, traceroute needs the suid bit. It uses the ICMP
packets.

/usr/bin/crontab
From the manpage:
    crontab is the program used to install, deinstall, or list the tables
    used to drive the cron(8) daemon in ISC Cron.  Each user can have their
    own crontab, and though these are files in /var/cron, they are not
    intended to be edited directly.

Because the cron job files are owned by root, crontab needs the suid to allow
regular users to add jobs.

I noticed a lot of the binaries listed by the find command I used are networking
related.

Consider the following file and directory information:

    $ ls -ld . file
    drwx-w-r-x  2 alice  apue  48 Sep 11 02:57 .
    --wx--s--x  1 alice  apue   4 Sep 11 02:57 file
    $

Which users can copy the file? Which can remove it?
- alice cannot copy because user/owner read perms are off. alice can create a
  new file in the directory though.
- no one in the group apue can open it either because group read perms are off.
- others can't copy because read perms are off.
- root can, because perms don't matter when you're root.

- actually alice can read it after changing the permissions.

- alice can remove the file because x and w on the directory
- root can remove the file because root
- group cannot but it can add execute permissions and then rm
- other cannot.

Most Unix systems don't allow a non-root user to chown files.
Why not?
- If you allow non-root users to chown files, you allow them to accidentally
  make files inaccessible to themselves.

---

Raw disks

/dev/rwd1a = trying to be westerndigital (wd)

/dev/c0l1s2p3

/dev/ld4a and /dev/rld4a
- rld4a is a char device while the first one is block device.

- with the r, you're telling the kernel, you know you're going to the disk directly.

- else you go through the kernel interface.

- to copy a disk directly, copy from raw disk

---

the stat output and ls output uses the names of the ids, from /etc/passwd. Thus that file has to be readable.

Why does /etc/passwd have multiple poeple with uid 0?

Convention is "root" has uid 0.

/rescue/sh is statically linked so you can still do stuff if you accidentally rm'd
your libc.

uids don't have any semantic meaning by default, but some systems have conventions
for system accounts.

It used to be the case that the password hashes were stored in /etc/passwd. BAD IDEA.

on bsd: `sudo cat /etc/master.passwd`

---

when we operate on raw network sockets, we need root privileges.

On Linux ping isn't setuid because of restricting other privileges.

---

To try to access a file, by default you use effective uid gid. You don't want to
drop privileges and raise privileges or stat the file.

Access with real uid and gid solves the problem atomically.

---

fsck() allows you to check if you rm'd data by accident.
Generally data is still there, but just unlinked.

---

permissions nowadays ACLs (Access Control Lists)

---

Please start using const for functions to introduce the invariants to the compiler.

---

tmp dir has sticky bit. sticky bit changes the semantics on who can remove.

only owner or root can actually delete files?

you can be abused through a symlink attack.

---

how are suid and sgid useful without execute permissions? It is not generally useful. Some systems decided to overload
this functionality. On some Linux distros, it does "mandatory locking".

---

st_rmode gets us the permission string.
for ls, just pretend ls -1


