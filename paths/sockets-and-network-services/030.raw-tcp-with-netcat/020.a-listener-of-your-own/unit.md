---
title: Open a listening port
vars:
  MYPORT: { pick: ["9401", "9402", "9403"] }
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
      sleep 0.3
tasks:
  listening:
    check: |
      command -v nc >/dev/null 2>&1 || \
        hint_exit "nc is not installed on this host. It ships as netcat-openbsd on Debian and Ubuntu and as nmap-ncat on Fedora and Rocky - install it with your package manager and this rep is back in business."
      wait_port "$MYPORT"
    hint: |
      echo "nc listens with -l plus the port number (some builds want -l -p ${MYPORT}). Append & so it runs in the background and you keep your prompt."
    solve: |
      nc -l $MYPORT < /dev/null &
  self_inspected:
    needs: [listening]
    check: |
      wait_exec '(^|/)(ss|netstat|lsof)( |$)'
    hint: |
      echo "Same command as in the first module - list the TCP listeners and find your own nc among them."
    solve: |
      ss -tlnp
---

You have been reading other people's listeners all path long. Open one
of your own on port `${MYPORT}`.

`nc -l ${MYPORT}` binds the port and waits for a connection. It holds
the terminal while it waits, so send it to the background with a
trailing `&` and keep your prompt:

::task{name="listening"}
#active
Waiting for a listener on port `${MYPORT}`...
#completed
Bound. Your `nc` is now a network service, in every sense the kernel
cares about.
::

Now look at it from the outside - list the TCP listeners on this host
and find your own process among them:

::task{name="self_inspected"}
#active
Waiting for you to inspect the listening sockets...
#completed
There it is, next to the system's own services: same table, same
columns, and this time you know exactly which process is behind the
row.
::

::tip{title="Keep a background listener out of your terminal"}
A backgrounded `nc` still has your terminal as its standard input, and
the shell suspends any background process that tries to read from it.
`nc -l ${MYPORT} < /dev/null &` gives it an empty input instead, so it
just listens.
::
