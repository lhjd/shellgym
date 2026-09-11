# Who is listening?

To Linux, a network service is just a process holding a **socket** that
is bound to an address and a port and waiting for connections. Nearly
every everyday network question - "is the API up?", "why is the port
already in use?", "why does it work on the box but not from my laptop?"
- is answered by reading that one picture correctly.

The tool for reading it is `ss` (socket statistics), part of the
`iproute2` package that ships with every modern distribution. Its
retired predecessor `netstat` makes a guest appearance too, because you
will keep meeting it in older runbooks.

Your practice target throughout this path is `notes-api`: a deliberately
small HTTP service that stores short text notes. Every rep installs and
starts it for you and tells you everything about it that you need -
there is nothing to carry over from one rep to the next.
