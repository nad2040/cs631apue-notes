# IPC 2

## socketpair(2)

```c
int socketpair(int domain, int type, int protocol, int *sv);
```
the descriptors are written to `sv[0]` and `sv[1]`.

this call is currently only implemented for UNIX/local domain
AF_UNIX or PF_LOCAL

## socket(2) PF_LOCAL

```c
int socket(int domain, int type, int protocol);
```

in practice selecting the default protocol 0 works.

Domains:
- PF_LOCAL (prev AF_UNIX) AF meant address format. PF means protocol family
- PF_INET ARPA internet protocol
- PF_INET6 IPv6
- ...

Types:
- SOCK_STREAM (TCP) sequenced, reliable, two-way connection
- SOCK_DGRAM (UDP) connectionless, unreliable messages of a fixed maximum length
- SOCK_RAW access to internal network protocols and interfaces (ex ICMP packets).
  only available to superuser
- ...


with DGRAM:
`sendto()` and `recvfrom()`

`bind(2)` assigns a name to an unnamed socket.

binding a local domain socket creates an entry in the FS.

sendto doesn't check for connection so we don't need to use connect in the case of DGRAM.

after communication, sockets must be removed from the FS manually via unlink.

## PF_INET + DGRAM

the kernel assigns a port number for us, so we just use getsockname to get the port number.

we need to convert from network byte order to host byte order.

TCP/IP uses Big-Endian

use ntohs or ntohl

gethostbyname returns network address of the specified port.


netstat shows open ports on system.

`netstat -na | more`

with udp, we can send a packet when the receiver is down and we will never know if it arrives.

`tcpdump` captures traffic

`sudo tcpdump -X -n host <hostname>`


0x0303 represents unreachable

you  might receive an icmp to state that the host was unreachable.


Internet sockets do not get a FS entry, so we don't need to clean up after ourselves.

we can listen to any specific IP address or use wildcard INADDR_ANY

request an ephemeral port by calling bind with port 0

well-known ports 0-1023 can only be bound with euid 0.

use getsockname() to get port number

UDP is connectionless/unreliable. you can send packets without anyone listening.

Try:
- what happens if you specify a hostname with both IPv4 and v6 addresses or multiple.
  which address gets used?

practice using tcpdump.

## PF_INET6 + STREAM

TCP streams on IPv6.

gethostbyname2 allows you to specify the domain.

need to use connect.

for server side:
- bind
- listen
- accept gives you socketfds that actually correspond with each client.

connect will fail for clientside when the server is down.

you can use telnet to send data. netstat will show established connections.


accept will block if no clients are connected.

each connection requires a 3 way handshake (SYN, SYNACK, ACK)
each disconnection requires a FIN ACK FIN ACK.

Try:
- use send or sendto instead of write
- upgrade both programs to use a dual stack environment (v4 and v6).

- stream reader allows multiple clients to connect. how does the reader handle this?
- what happens if a client connects, sends, and disconnects before the server can react.
- what happens if you get more clients than you specified in listen. Use tcpdump to observe
  packets.

## I/O Multiplexing

You can only handle one socket at a time. This is why we need I/O multiplexing.

Modes of multiplexing:
- blocking - open one fd. block. then test the next fd
- fork - and use one process per client. communicate using signals or other IPC
- non-blocking mode - open one fd, immediately get results, open next fd,
  immediately get results, sleep
- async I/O - get notified by the kernel when either fd is ready for I/O

drawbacks:
- blocking is no good
- busy-polling each fd is also not desirable
- async I/O is somewhat limited

Let's use a special function that will tell us which ones are ready for I/O and let us
sleep otherwise.

```c
int select(int nfds, fd_set * restrict readfds, fd_set * restrict writefds,
           fd_set * restrict exceptfds, struct timeval * restrict timeout);
```

nfds: which fds we're interested in.
readfds, writefds, exceptfds: what conditions we're interested in.

timeout could be NULL - wait forever
or set to 0 - don't wait

FD sets are manipulated with the macros FDSET, FDCLR, FDISSET, FDZERO.

read, write, and except fd sets indicate those conditions. (except includes Out-of-Band data,
certain terminal events, etc)

Hitting eof requires a read, so a fd will be ready to read even if you then read EOF.

pselect provides ns granularity on the timeout and allows you to provide a signal mask.



connection means fd is ready.

Let's combine select and fork.

Other options for Multiplexing.
- poll(2)
- epoll(2) (Linux)
- kqueue(2) (*BSD)
- libevent or libev

See c10k for more detail.


---

Weekly Checkpoint

Sockets in the UNIX domain can be used for communications between unrelated processes.
But can you also use a socket for communication between related processes?
Can you create a socket, then fork, then communicate via that file descriptor?

- I don't see why you couldn't use sockets to communicate between a parent and child.
  However, I don't think it's possible to create the socket and then fork. That would
  give the child process the same file descriptor as the parent. You need to fork, and
  then create end of the socket separately. I guess this is what socketpair basically
  does for you. It gives you two ends and you use one end for one process and the other
  end for the other process.

Can you have multiple processes using the same socket to send data to a single reader?

- I think so, but then you wouldn't be able to differentiate between the data sent by each
  process unless you send that information and make the reader parse the output.
  There could also be some interleaving of output. Also, if you close the socket on one process,
  it would close on all the others.

When should you call htons(3)/ntohs(3), when htonl(3)/ntohl(3)?

- s means short (2 bytes) and l means long (actually 4 bytes). Ports go from 0-65535 (2 bytes).
  v4 addresses are 4 octets (4 bytes). You would use the short function on ports and long function
  on v4 addresses.

How does streamread.c handle multiple simultaneous connections?

- It seems that stream read only handles (reads from) one connection at a time. When the connection stops,
  the next one in the backlog gets handled.

---

Only for local domain we create file in filesystem.

on socketpair, you cannot just write to the file.

Create signal handlers to catch signals and clear the socket pair.

"Address already in use" Is likely to cause confusion.

postfix daemon might use local domain sockets.

bind allows you to name an unnamed socket.

sockaddr_in - Internet domain
sockaddr_un - UNIX domain

bind with INADDR_ANY allows any connections
name.sin_addr.s_addr

port = 0 -> ephemeral port >= 1024
port < 1024, reserved well-known ports

check /etc/services for known ports

you can only use a port number if it's free.

netstat shows state for tcp and nothing for udp. udp is stateless.

`/etc/nsswitch.conf`
name-service switch configuration file

first look at files, etc, the finally look go to DNS server.

Things get funky if your system doesn't define localhost as 127.0.0.1 or ::1.

`host` and `dig` are DNS lookups.

`ping` calls gethostbyname. this then looks at nsswitch.conf.

don't mess will your /etc/hosts!!!

traceroute iteratively uses shorter and shorter time-to-live to get the full path from
you to an ip.

STREAM socket require a separate socket to listen for incoming connections.

I/O multiplexing (keep track of multiple fds at the same time)

Source Addr/Port, Dest Addr/Port uniquely determines a connection.

Is SOCK_STREAM = TCP and SOCK_DGRAM = UDP?

SOCK_STREAM and SOCK_DGRAM are pretty much equivalent to TCP and UDP.
There are exceptions but those are rare.

QUIC is built on UDP and datagrams.

SCSP is a stream protocol.

When do we use raw sockets?
- ICMP, ARP, etc. you need superuser.
- tools that use raw sockets use set-uid.

tcpdump opens an interface in promiscuous mode. This is different from
a socket.

Broadcast address is the highest IP in the subnet. INADDR_ANY means bind to
any address.

Broadcast will send to any machine on the network. if they respond, then you
get the response.

How to do a UDP port scan?
- you don't get a response. sometimes you won't get an ICMP host-unreachable.

With TCP, you either get SYN-ACK, RESET or nothing.

How does IPv6 work?
- not too different from IPv4.

don't need conversion from host to network order for IPv6, because it will
always be in the correct order.

FD_SETSIZE is the limit of the fd sets.

---

sws

CGIs (Common Gateway Interface) - interactive websites done by fork.

Directory Indexing - display contents when index.html doesn't exist.

`/usr/libexec/httpd -b -I 8080 -X -u -p sws -c cgi-bin docroot`



