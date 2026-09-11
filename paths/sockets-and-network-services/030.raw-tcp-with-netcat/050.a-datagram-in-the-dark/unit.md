---
title: Send a UDP datagram
vars:
  MYPORT: { pick: ["9421", "9422", "9423"] }
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
      rm -f "$GYM_USER_HOME/udp-inbox.txt"
      sleep 0.3
tasks:
  udp_bound:
    timeout: 60
    check: |
      command -v nc >/dev/null 2>&1 || \
        hint_exit "nc is not installed on this host. It ships as netcat-openbsd on Debian and Ubuntu and as nmap-ncat on Fedora and Rocky - install it with your package manager and this rep is back in business."
      for attempt in $(seq 1 40); do
        if ss -uanH 2>/dev/null | awk '{print $4}' | grep -q ":$MYPORT$"; then
          exit 0
        fi
        sleep 1
      done
      hint_exit "No UDP socket is bound to port ${MYPORT} yet. A plain nc -l opens a TCP socket - it needs one more option to speak UDP."
    hint: |
      echo "Add -u to the listening form you used before, and keep the redirect into ~/udp-inbox.txt."
    solve: |
      nc -u -l $MYPORT > ~/udp-inbox.txt < /dev/null &
      #!wait 2
  delivered:
    needs: [udp_bound]
    timeout: 60
    check: |
      wait_file_contains "$GYM_USER_HOME/udp-inbox.txt" "^$TOKEN$"
    hint: |
      echo "The sending side needs -u as well - a TCP client cannot talk to a UDP listener, and it will not be told so."
    solve: |
      echo $TOKEN | nc -u -w 2 127.0.0.1 $MYPORT &
      #!wait 3
---

Everything so far has been TCP: a connection is established, bytes
arrive in order, and both ends find out when something breaks. UDP drops
all of that. There is no connection - just single datagrams, fired off
in the hope that somebody is listening.

`nc` speaks it with one extra option, `-u`, on both sides.

Start a UDP listener on port `${MYPORT}`, collecting what arrives into
`~/udp-inbox.txt`:

::task{name="udp_bound"}
#active
Waiting for a UDP socket bound to port `${MYPORT}`...
#completed
Bound. Notice it does not appear in `ss -tln` - that is the TCP table.
`ss -uln` is where UDP sockets live, and they show a state of `UNCONN`,
because there is no connection to be in.
::

Now send the token to it, again over UDP:

```
${TOKEN}
```

::task{name="delivered"}
#active
Waiting for `${TOKEN}` to arrive in `~/udp-inbox.txt`...
#completed
Delivered - this time with nobody guaranteeing that it would be. Send a
UDP datagram to a port where nothing is listening and your command
succeeds just the same: the silence is the whole point of the protocol,
and the reason DNS, NTP and syslog build their own checks on top of it.
::
