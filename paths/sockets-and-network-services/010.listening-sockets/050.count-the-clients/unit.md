---
title: List established connections
requires: [systemd, python3]
vars:
  PORT: { pick: ["9341", "9342", "9343"] }
  CLIENTS: { pick: ["2", "3", "4"] }
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
  - name: install_probe
    run: |
      install -d -m 0755 /opt/notes-api
      cat > /opt/notes-api/api-probe <<'PY'
      #!/usr/bin/env python3
      """api-probe - holds one idle TCP connection open, so ss has rows to show."""
      import socket
      import sys
      import time

      sock = socket.create_connection(("127.0.0.1", int(sys.argv[1])), timeout=10)
      time.sleep(int(sys.argv[2]))
      sock.close()
      PY
      chmod 0755 /opt/notes-api/api-probe
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
      rm -f "$GYM_USER_HOME/clients.txt"
      systemd-run --collect --quiet --unit=notes-api --uid="$GYM_USER" --setenv=NOTES_BUILD="$BUILD" \
        /opt/notes-api/notes-api "$PORT" 0.0.0.0 || {
        echo "systemd-run refused to start notes-api" >&2
        exit 1
      }
      wait_port --timeout 20 "$PORT" || {
        echo "notes-api did not come up on port $PORT" >&2
        systemctl status notes-api --no-pager 2>&1 | tail -10 >&2
        exit 1
      }
  - name: start_clients
    run: |
      i=1
      while [ "$i" -le "$CLIENTS" ]; do
        systemd-run --collect --quiet --unit="api-probe-$i" --uid="$GYM_USER" \
          /opt/notes-api/api-probe "$PORT" 3600
        i=$((i + 1))
      done
      for attempt in $(seq 1 40); do
        n=$(ss -tnH state established "dst :$PORT" 2>/dev/null | wc -l)
        [ "$n" -ge "$CLIENTS" ] && exit 0
        sleep 0.5
      done
      echo "only $n of $CLIENTS import connections came up" >&2
      exit 1
tasks:
  counted:
    check: |
      wait_file_contains "$GYM_USER_HOME/clients.txt" "^$CLIENTS\s*$"
    hint: |
      echo "With -l you asked for listening sockets; drop it and ss shows connections too. Narrow them down with: state established, and a dst filter for port ${PORT}."
    solve: |
      ss -tn state established dst :$PORT
      ss -tnH state established dst :$PORT | wc -l > ~/clients.txt
---

Listening sockets are only half of the picture. Every client that is
actually *connected* to `notes-api` has a socket too, in the `ESTAB`
(established) state.

Drop the `-l` and `ss` stops restricting itself to listeners. Two extra
pieces of its filter language narrow the result down:

```
ss -tn state established dst :${PORT}
```

- `state established` - only fully established connections (other
  useful values: `listening`, `time-wait`, `syn-sent`),
- `dst :${PORT}` - only sockets whose *remote* end is port `${PORT}` -
  in other words, the client side of connections **to** `notes-api`.

A batch of import workers is connected right now. Count those
connections and write the number into `~/clients.txt`:

::task{name="counted"}
#active
Waiting for the connection count in `~/clients.txt`...
#completed
Counted. Watching this number is how you tell a service that is idle
from one that is drowning - and `ss -tn state time-wait` is where you
look when a host runs out of ephemeral ports.
::

::tip{title="Headers get in the way of counting"}
`-H` tells `ss` to omit the header line, which is exactly what you want
when the output is going into `wc -l`.
::
