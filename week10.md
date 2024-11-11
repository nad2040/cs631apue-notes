# Daemons

Do one thing and one thing only.

Don't leak resources!!!

Consider current working directory

No or limited user interaction

consider creating debug output (logging)

== Writing a Daemon

clear the env
fork
terminate parent
set desirable umask
setSID to become session leader
change CWD to a safe place, like root
close or redirect to /dev/null 0,1,2
open logs for writing
enter the actual code of the server/service

prevent against multiple instances via a lockfile

allow for easy determination of pid via a pidfile

include a system initialization script (rc.d, init.d, systemd)

configuration file convention: /etc/name.conf

reread config file on SIGHUP
- Under normal circumstances, SIGHUP is only generated when a
  process loses its controlling terminal

relay information via event logging (syslog)

```c
daemon(int nochdir, int noclose);
```

daemon() changes the PID because we forked.


