# POSIX.1e ACLs

fine-grained control

implemented as extended attributes on the filesystem. It requires FS support.

users can specify individuals or groups with different access

## on Linux

```sh
setfacl -m g:group1:r file
setfacl -m u:user1:r file
```

NFS doesn't implement posix acls, but has its own in nfs v4.

if the extended attributes are present, a + is added after the strmode.

`cp` by default doesn't copy these ACLs

`getfacl file1 | setfacl --set-file=- file2` copies ACLs to file2

use `cp -p`

`setfacl -b` clears ACLs.

## on MacOS

```
chmod +a "group:wheel allow read" file
chmod +a "daemon deny read" file
```

and `ls -le` to show extended attributes.

see `man chmod` to get info on ACLs.

to delete ACL

```
chmod -a# num file
```

## on NetBSD

`tunefs -p enable` allows POSIX ACLs on the filesystem

the order of checking still remains user, then group, then other.

# eUIDs, file flags, mount options, securelevels

setuid programs
- require careful raising and lowering of privileges _only when needed_ (least privilege)
- rely on correct ownership and permissions (out of programmer control)

su(1)
- requires sharing of passwords.
- grants all or nothing access

sudo(8)
- often misconfigured granting too broad access (ALL:ALL)
- additional authen often dropped (NOPASSWD)
- restrictions often overlook privilege escalation.

if you give another user your privileges to use vi, they can edit your shell startup file.

shells check if the eUID matches the real UID so that is a safeguard against changing users.

but you can just use -p.

specifying the directory in the sudoers file helps restrict the access.

vi still lets you read the other files, but doesn't allow you to write.
But, you can call shell commands, change permissions, do the write.

## file flags

```c
int chflags(const char *path, u_long flags);
```

Linux systems use `chattr`

can be used to prevent files from being changed (immutable file).

only root can unset the flag if it is on.

`UF_APPEND, UF_IMMUTABLE`
`SF_APPEND, SF_IMMUTABLE`

if system immutable, it will also prevent removal even though usual removal semantics is
relevant to the directory and not the file itself. this is because removal still needs to
decrement the link count and thus needs to modify the file.

can be set on directories. in this case you can modify files, but not remove or rename

## mount -o

`sudo mount -u -o nosuid /mnt`

then running the suid executable doesn't set the uid to the owner.

`sudo mount -u -o noexec /mnt`

no one can run the suid executable now.

other options include rdonly. see mount man page.

## securelevels

superuser can raise the level, but only init can lower it.

lowering requires reboot

- Permanently insecure (-1)
- Insecure (0)
- Secure (1)
- Highly secure (2)

see secmodel_securelevel(9)

`sysctl security.models.securelevel.securelevel` to check current securelevel

# restricted shells, chroots, jails

restricted shells often
- prohibit cd
- prohibit changing environment variables
- prohibit running full path to executable (force PATH)
- prohibit output redirection

beware of trivial breakout

to do so, manually set the PATH for the restricted shell

however you need to keep track of commands like dd, which effectively redirect output.

if you get ed, you can just edit the file you want to.

## chroot

```c
int chroot(const char *dirname);
```

you need to setup everything inside this directory relative to the new root.

`su root -c "chroot /path/to/chroot /bin/sh"`
enters the chroot as root.

pwd shows /

if you forgot to give yourself ls, use echo *

ps can reveal you are in a chroot. it will show you the processes outside the chroot.

combine chroots with null mounts and use some mount options

open fds may be brought into the chroot.

## jails

- enforce a per-jail process view
- prohibit changing sysctl and securelevels
- prohibit mounting and unmounting filesystems
- prohibit modifying the network configuration
- disable raw sockets

jails effectively implement a process sandbox environment, forming the first OS-level virtualization.

chroots have been removed from POSIX but it still remains in most UNIX systems.

# Process priorities

scheduling priority

```c
#include <sys/resource.h>

int getpriority(int which, id_t who);
int setpriority(int which, id_t who, int prio);
```
default priority is 0
which is PRIO_PROCESS, PRIO_GROUP, PRIO_USER
who is PID, PGID, or UID.

prio is a value between -20 and 20.
value of 19 or 20 will schedule only if nothing <= 0 is running.

only superuser can lower the value.

Check errno! -1 is a valid return value!


`uptime` command shows uptime and load averages (last 1, 5, 15 minutes). This is the
number of processes in the run queue averaged over that interval

`nice` runs a process with a modified priority
`renice` modifies the priority of an already running process

top shows kern priority and they are adjusted values.

priority does not influence CPU placement

# Processor Affinity and CPU Sets

CPU pinning

`schedctl(8)` used to control scheduling of processes and threads

changing cpu affinity requires superuser privileges.

`schedctl -A <cpu> -p <pid>`

you can choose to allow users to change the cpu affinity
`sudo sysctl -w security.models.extensions.user_set_cpu_affinity=1`

you can change affinity while executing

child inherits parents processor affinity, but we can still change it.

CPU sets:
- really reserve cpus for certain processes.
- there is always a default set for all other processes.

`sudo psrset -c 1 2` (makes cpu 1 and 2 into one set)
- requires sudo

`sudo psrset -e <set number> <command>`

We can still explicitly move processes to default cpu set by setting the
affinity. Once we do so, we cannot move them back to the cpu set unless
you call psrset.

To make the sets available again, you need to delete them and remake them.

None of this is standardized!!!

# Capabilities, Control Groups, Containers

POSIX capabilities
- CAP_CHOWN - ability to chown files
- CAP_SETUID
- CAP_LINUX_IMMUTABLE - allow append only or immutable flags
- CAP_NET_BIND_SERVICE - allow network sockets < 1024
- CAP_NET_ADMIN - allow iface configuration, route table manipulation
- CAP_NET_RAW - allow raw packets
- CAP_SYS_ADMIN - broad sysadmin privileges (mounting fs'es, setting hostname, handling swap)

FreeBSD uses capsicum(4)
macOS and NetBSD use kauth
Linux uses capabilities(7)

Linux Namespaces
- mnt - mount points
- pid - process ID visibility
- net - virtualized network stack
- ipc - system V ipc visibility
- uts - unix time sharing (different host- or domain- names)
- user - user-IDs and privileges
- time - system time
- cgroup - control groups

Linux Control Groups (originally process containers)
- resource limiting (memory limit)
- prioritization (CPU util, disk I/O throughput)
- accounting
- control (freezing, checkpointing, restarting)
v2
- cpu - ability to schedule tasks
- cpuset - CPU and memory nodes
- freezer - activity of control groups
- hugetlb - large page support (HugeTLB) usage
- io - block device I/O
- memory - memory, kernel memory, swap memory
- perf_event - ability to monitor threads
- pids - number of processes
- rdma - remote directory memory access

implemented using virtual filesystem often under /sys/fs/cgroup

example
```
mkdir /sys/fs/cgroup/memory/group0
echo $$ > !$/tasks
echo 40M > /sys/fs/cgroup/memory/group0/memory.limit_in_bytes
```
see cgroups(7)

Containers
- use a null or union mount to provide the right environment
- restrict the processes in their utilization
- restrict filesystem views
- restrict processes from what they can see
- restrict processes from what they can do

cgroups and namespaces form the basis of many container technologies: CoreOS, LXC, Docker.


