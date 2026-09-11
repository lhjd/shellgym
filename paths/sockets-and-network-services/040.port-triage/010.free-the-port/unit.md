---
title: Free a port that is already in use
requires: [systemd, python3]
vars:
  PORT: { pick: ["9371", "9372", "9373"] }
init:
  - name: install_agents
    run: |
      install -d -m 0755 /opt/archive-sync
      cat > /opt/archive-sync/archive-sync <<'PY'
      #!/usr/bin/env python3
      """archive-sync - accepts connections and closes them without a word."""
      import socket
      import sys

      srv = socket.socket()
      srv.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
      srv.bind(("0.0.0.0", int(sys.argv[1])))
      srv.listen(8)
      while True:
          conn, _ = srv.accept()
          conn.close()
      PY
      chmod 0755 /opt/archive-sync/archive-sync
  - name: start_squatter
    run: |
      for u in notes-api notes-api-import notes-api-admin archive-sync archive-sync-2; do
        systemctl stop "$u.service" 2>/dev/null || true
      done
      for i in 1 2 3 4 5; do
        systemctl stop "api-probe-$i.service" 2>/dev/null || true
      done
      pkill -u "$GYM_USER" -f 'notes-ap[i] ' 2>/dev/null || true
      pkill -u "$GYM_USER" -f 'archive-syn[c] ' 2>/dev/null || true
      pkill -u "$GYM_USER" -f 'api-prob[e] ' 2>/dev/null || true
      sleep 0.5
      systemd-run --collect --quiet --unit=archive-sync --uid="$GYM_USER" \
        /opt/archive-sync/archive-sync "$PORT" || {
        echo "systemd-run refused to start archive-sync" >&2
        exit 1
      }
      wait_port --timeout 20 "$PORT" || {
        echo "archive-sync did not come up on port $PORT" >&2
        systemctl status archive-sync --no-pager 2>&1 | tail -10 >&2
        exit 1
      }
tasks:
  port_freed:
    timeout: 60
    check: |
      wait_port --timeout 15 "$PORT" || \
        hint_exit "Nothing is listening on port ${PORT} any more - this rep is already done, or its scene never came up."
      wait_port_free "$PORT"
    hint: |
      echo "The owner column of ss -tlnp holds the PID; kill takes it from there. ps -p <pid> -o args= first, if you want to see what you are about to stop."
    solve: |
      ss -tlnp sport = :$PORT
      kill $(ss -tlnpH sport = :$PORT | grep -o 'pid=[0-9]*' | cut -d= -f2)
---

A deploy of the `notes-api` service has just failed with the message
every operator has read at least once:

```
OSError: [Errno 98] Address already in use
```

The port it wants is `${PORT}`, and something else is already sitting on
it. The fix is never to guess - it is to ask who holds the port, decide
whether that process may go, and then stop it.

Find the process listening on `${PORT}` and terminate it, so the port
becomes free again:

::task{name="port_freed"}
#active
Waiting for port `${PORT}` to become free...
#completed
Released. What you stopped was an `archive-sync` agent that had been
misconfigured onto the API's port - which is exactly why the second step
matters: `ps -p <pid> -o args=` before `kill <pid>`, so you find out
what you are stopping while you still have the choice.
::

::hint{title="If kill does not seem to work"}
The default signal, `TERM`, asks a process to shut down and can be
ignored. Check with `ss` whether the port is really free; if the process
is still there, `kill -9 <pid>` cannot be refused - at the cost of
giving it no chance to clean up.
::
