# Raw TCP with netcat

`curl` speaks HTTP on your behalf. `nc` (netcat) speaks nothing at all:
it wires a TCP or UDP socket straight to your terminal's standard input
and output and then gets out of the way. That makes it the tool for the
layer underneath - is the port open at all, what does the service say if
I talk to it by hand, can these two endpoints exchange a single byte?

Three shapes cover most of what you will ever do with it:

| Command | What it does |
|---|---|
| `nc HOST PORT` | connect, send stdin, print whatever comes back |
| `nc -l PORT` | listen on PORT and print what arrives |
| `nc -z HOST PORT` | only test whether the port accepts a connection |

Add `-u` to any of them to use UDP instead of TCP.

::tip{title="Which netcat?"}
Several netcat implementations are in circulation. Debian and Ubuntu
usually ship `netcat-openbsd`, whose listen form is `nc -l PORT`; the
traditional netcat wants `nc -l -p PORT`, and Fedora and Rocky ship
`nmap-ncat`, where `nc` is a link to `ncat`. If one form complains about
its arguments, try the other - the ideas are identical.
::
