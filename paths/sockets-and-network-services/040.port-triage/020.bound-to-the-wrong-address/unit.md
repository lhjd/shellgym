---
title: Rebind a service to the right address
requires: [systemd, python3]
vars:
  PORT: { pick: ["9381", "9382", "9383"] }
  BUILD: { shell: "head -c4 /dev/urandom | od -An -tx1 | tr -d ' \\n'" }
init:
  - name: install_service
    run: |
      install -d -m 0755 /opt/notes-api
      cat > /opt/notes-api/notes-api <<'PY'
      #!/usr/bin/env python3
      """notes-api - a small HTTP service that stores short text notes."""
      import json
      import os
      import sys
      from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer

      PORT = int(sys.argv[1])
      BIND = sys.argv[2] if len(sys.argv) > 2 else "0.0.0.0"
      BUILD = os.environ.get("NOTES_BUILD", "dev")
      INBOX = os.environ.get("NOTES_INBOX", "/var/lib/notes-api/received.log")

      NOTES = [
          {"id": "n-1041", "text": "rotate the backup keys", "author": "ana"},
          {"id": "n-2277", "text": "restart the ingest worker", "author": "bo"},
          {"id": "n-3390", "text": "archive last quarter of logs", "author": "chen"},
      ]


      class Handler(BaseHTTPRequestHandler):
          server_version = "NotesAPI/1.4"
          sys_version = ""

          def reply(self, code, payload, headers=()):
              body = (json.dumps(payload, indent=2) + "\n").encode()
              self.send_response(code)
              self.send_header("Content-Type", "application/json")
              self.send_header("Content-Length", str(len(body)))
              for name, value in headers:
                  self.send_header(name, value)
              self.end_headers()
              if self.command != "HEAD":
                  self.wfile.write(body)

          def do_GET(self):
              if self.path == "/status":
                  self.reply(200, {"status": "ok", "build": BUILD})
              elif self.path == "/notes":
                  self.reply(301, {"moved": "/api/notes"}, [("Location", "/api/notes")])
              elif self.path == "/api/notes":
                  self.reply(200, {"notes": NOTES})
              else:
                  self.reply(404, {"error": "no such endpoint"})

          do_HEAD = do_GET

          def do_POST(self):
              length = int(self.headers.get("Content-Length") or 0)
              raw = self.rfile.read(length).decode("utf-8", "replace")
              if self.path != "/api/notes":
                  self.reply(404, {"error": "no such endpoint"})
                  return
              try:
                  note_id = str(json.loads(raw)["id"])
              except Exception:
                  self.reply(400, {"error": "expected a JSON body with an id field"})
                  return
              os.makedirs(os.path.dirname(INBOX), exist_ok=True)
              with open(INBOX, "a") as fh:
                  fh.write(note_id + "\n")
              self.reply(201, {"accepted": note_id})

          def log_message(self, *args):
              pass


      ThreadingHTTPServer.allow_reuse_address = True
      ThreadingHTTPServer((BIND, PORT), Handler).serve_forever()
      PY
      chmod 0755 /opt/notes-api/notes-api
      mkdir -p /var/lib/notes-api
      chown "$GYM_USER" /var/lib/notes-api
  - name: start_service
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
      systemd-run --collect --quiet --unit=notes-api --uid="$GYM_USER" --setenv=NOTES_BUILD="$BUILD" \
        /opt/notes-api/notes-api "$PORT" 127.0.0.1 || {
        echo "systemd-run refused to start notes-api" >&2
        exit 1
      }
      wait_port --timeout 20 "$PORT" || {
        echo "notes-api did not come up on port $PORT" >&2
        systemctl status notes-api --no-pager 2>&1 | tail -10 >&2
        exit 1
      }
tasks:
  wildcard_bound:
    timeout: 60
    check: |
      for attempt in $(seq 1 40); do
        if ss -tlnH "sport = :$PORT" 2>/dev/null | awk '{print $4}' | grep -qE '^(0\.0\.0\.0|\*|\[::\]):'; then
          exit 0
        fi
        sleep 1
      done
      if [ -n "$(ss -tlnH "sport = :$PORT" 2>/dev/null)" ]; then
        hint_exit "Port ${PORT} is listening again, but still on 127.0.0.1 only. A port can only be bound once - the old process has to stop before the new one can take it."
      fi
      hint_exit "Nothing is listening on port ${PORT}. Start the server again, passing the address it should bind as the second argument."
    hint: |
      echo "Find the running server first (its command line ends in ${PORT} 127.0.0.1), stop it, then start it again with 0.0.0.0 as the second argument."
    solve: |
      ss -tlnp sport = :$PORT
      pkill -f "notes-api $PORT"
      #!wait 1
      /opt/notes-api/notes-api $PORT 0.0.0.0 &
      #!wait 3
---

`notes-api` is up on port `${PORT}` and answers perfectly from this
machine - and not at all from anywhere else. You met the reason in the
first module: it is bound to `127.0.0.1`, so the socket only ever
accepts connections that never left the host.

The server takes the address to bind as its second argument:

```
/opt/notes-api/notes-api <port> <address>
```

Get it listening on port `${PORT}` on **all** addresses. Two things have
to happen in the right order, because a TCP port can only be bound by
one socket at a time: the process holding `${PORT}` now has to go before
a new one can take it.

::task{name="wildcard_bound"}
#active
Waiting for a listener on `0.0.0.0:${PORT}`...
#completed
Rebound, and now reachable from the network. This is the single most
common version of "works on the server, not from my laptop" - and in
real services it is a line in a config file (`bind_address`, `listen`,
`--host`) rather than an argument, but the socket table tells the same
story either way.
::

::tip{title="Binding to everything is a decision"}
`0.0.0.0` exposes the service to every network the host is attached to.
For an admin endpoint or a database, `127.0.0.1` - or the address of one
internal interface - is usually the right answer, and the firewall is
the second line of defence, not the first.
::
