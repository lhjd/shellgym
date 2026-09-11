---
title: Send data between two sockets
vars:
  MYPORT: { pick: ["9411", "9412", "9413"] }
  TOKEN: { shell: "head -c5 /dev/urandom | od -An -tx1 | tr -d ' \\n'" }
init:
  - name: install_netcat
    run: |
      command -v nc >/dev/null 2>&1 && exit 0
      if command -v apt-get >/dev/null 2>&1; then
        DEBIAN_FRONTEND=noninteractive apt-get install -y -q netcat-openbsd >/dev/null 2>&1 || {
          apt-get update -qq >/dev/null 2>&1 || true
          DEBIAN_FRONTEND=noninteractive apt-get install -y -q netcat-openbsd >/dev/null 2>&1 || true
        }
      elif command -v dnf >/dev/null 2>&1; then
        dnf install -y -q nmap-ncat >/dev/null 2>&1 || true
      elif command -v apk >/dev/null 2>&1; then
        apk add --no-cache netcat-openbsd >/dev/null 2>&1 || true
      fi
      command -v nc >/dev/null 2>&1 || echo "nc is not installed and could not be installed here" >&2
      exit 0
  - name: clear_port
    run: |
      pkill -u "$GYM_USER" -f "n[c] .*$MYPORT" 2>/dev/null || true
      rm -f "$GYM_USER_HOME/inbox.txt"
      sleep 0.3
tasks:
  listening:
    check: |
      command -v nc >/dev/null 2>&1 || \
        hint_exit "nc is not installed on this host. It ships as netcat-openbsd on Debian and Ubuntu and as nmap-ncat on Fedora and Rocky - install it with your package manager and this rep is back in business."
      wait_port "$MYPORT"
    hint: |
      echo "Start the listener on ${MYPORT} first, and send its standard output into ~/inbox.txt with a redirect."
    solve: |
      nc -l $MYPORT > ~/inbox.txt < /dev/null &
  delivered:
    needs: [listening]
    check: |
      wait_file_contains "$GYM_USER_HOME/inbox.txt" "^$TOKEN$"
    hint: |
      if [ -s "$GYM_USER_HOME/inbox.txt" ]; then
        echo "Something arrived, but it is not ${TOKEN} - compare it with the token on the page."
      else
        echo "The listener is up, so now connect a second nc to 127.0.0.1 on port ${MYPORT} and feed the token into its standard input."
      fi
    solve: |
      echo $TOKEN | nc 127.0.0.1 $MYPORT &
      #!wait 2
---

Two `nc` processes make a complete, if very minimal, network
application: one listening, one connecting, and a stream of bytes in
between.

First, the receiving end. Start a listener on port `${MYPORT}` and send
whatever arrives into `~/inbox.txt`:

::task{name="listening"}
#active
Waiting for a listener on port `${MYPORT}`...
#completed
Listening, with its output pointed at the file.
::

Now the sending end. Open a second terminal - or use this one, since the
listener is in the background - and connect another `nc` to
`127.0.0.1` on port `${MYPORT}`, feeding it this token as input:

```
${TOKEN}
```

::task{name="delivered"}
#active
Waiting for `${TOKEN}` to arrive in `~/inbox.txt`...
#completed
Delivered. Nothing here knows or cares that it is one machine: replace
`127.0.0.1` with another host's address and the same two commands move
data across the network.
::

::tip{title="When nc will not hang up"}
Having sent your data, `nc` keeps the connection open in case the other
side answers. `-N` (or `-q 0` on some builds) tells it to close the
connection as soon as its input ends.
::
