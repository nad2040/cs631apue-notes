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

---

SIGHUP is used to signal when the controlling terminal disappears.

some daemons have debug mode (in the foreground instead of daemon mode).

We don't need a lockfile for our http server daemon.
`pidfile(3)` helps with lockfiles

---

# HTTP

HTTP 0.9
original version by Tim Berners Lee.

RFC1945 HTTP/1.0 (May 1996)
RFC2616 HTTP/1.1 (June 1999) most popular
RFC7540 HTTP/2 (May 2015)
RFC9114 HTTP/3 (June 2022)

RFC's use specific words in all caps.
- SHOULD
- MUST
- SHALL
- MAY

This is explained by RFC2119

HTTP is a request/response protocol.

different people choose to support different protocols.

1. Client sends a request to the server

If-Modified-Since: ...

only get the details if modified since


There are lots more headers.

User Agent header story:
Mozilla 5.0 was the last version before Firefox was born.
You can just write whatever in the user-agent.

Now we have Sec-Ch-Ua for actual user agent information.

These headers are useful for user fingerprinting.


2. Server responds

- status line (success or error code)

- Content-Type: text/html; charset=...

- MIME type

- 301 Moved Permanently
    frequently websites redirect you to https

1xx - informational; request received, continuing process
2xx - success;
3xx - redirection;
4xx - client error;
5xx - server error;


At some point you need to time out connections if you run out of ports.
(408)

Load balancing - protect servers from being overloaded.

Direct server return load balancing: so don't go through the gate server.

It let's you cache using multiple layers of servers.

-connect www.stevens.edu:443


HTTP is a transfer protocol. It serves data, not just text.

Accept or Accept-Encoding client header can specify different formats (e.g., application/json)
or encodings such as gzip, Brotli, etc.

Data can be transferred in either direction

POST, PUT, DELETE
GET,


CGI (Common Gateway Interface) - resouce is executed, needs to generate appropriate
    response headers

server-side scripting (ASP, PHP, Perl, ...)

client-side scripting (JavaScript/ECMAScript/JScript)

applications based on HTTP, using:
- AJAX
- RESTful services
- JSON, XML, YAML to represent state and abstract information

Browser restricts what you have access to.

---

What our project does:

HTTP/1.0
request methods: GET, HEAD
request headers: If-Modified-Since
response headers: Date, Server, Last-Modified, Content-Type, ... see manpage


program startup and daemonization
rough network logic
input / request parsing and validation
cgi handling
user directory handling (~username and append sws)
directory index generation
regular file serving


