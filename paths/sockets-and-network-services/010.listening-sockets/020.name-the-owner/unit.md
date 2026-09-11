---
title: Find the process behind a port
requires: [systemd, python3]
vars:
  PORT: { pick: ["9305", "9306", "9307"] }
  AGENT_PORT: { value: "9315" }
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
      rm -f "$GYM_USER_HOME/owner.pid"
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
      systemd-run --collect --quiet --unit=notes-api-import --uid="$GYM_USER" --setenv=NOTES_BUILD="$BUILD" \
        /opt/notes-api/notes-api "$AGENT_PORT" 0.0.0.0 || {
        echo "systemd-run refused to start notes-api-import" >&2
        exit 1
      }
      wait_port --timeout 20 "$AGENT_PORT" || {
        echo "notes-api-import did not come up on port $AGENT_PORT" >&2
        systemctl status notes-api-import --no-pager 2>&1 | tail -10 >&2
        exit 1
      }
tasks:
  owner_found:
    check: |
      PID=$(pgrep -f "notes-ap[i] $PORT" | head -1)
      if [ -z "$PID" ]; then
        echo "notes-api is not running on port $PORT" >&2
        exit 2
      fi
      wait_file_contains "$GYM_USER_HOME/owner.pid" "^$PID\s*$"
    hint: |
      if [ -s "$GYM_USER_HOME/owner.pid" ]; then
        echo "owner.pid holds a number, but not the PID of the process on port ${PORT}. Two services are listening here - read the row whose Local Address ends in :${PORT}."
      else
        echo "Adding -p to your ss options makes it print an owner column like users:((\"python3\",pid=1234,fd=3)). The number after pid= is what the file needs."
      fi
    solve: |
      ss -tlnp
      ss -tlnpH sport = :$PORT | grep -o 'pid=[0-9]*' | cut -d= -f2 > ~/owner.pid
      cat ~/owner.pid
---

Two services are listening on this machine right now: `notes-api` and
its import agent. One of them owns port
`${PORT}`, and you need to know which process that is - to read its
logs, to signal it, or just to find out what it actually is.

Adding `-p` asks `ss` to name the owner of each socket:

```
ss -tlnp
```

The extra column reads something like
`users:(("python3",pid=1234,fd=3))` - the program name, its process ID,
and the file descriptor number the socket sits on.

Write the PID of the process listening on `${PORT}` into `~/owner.pid`:

::task{name="owner_found"}
#active
Waiting for the PID of the process on port `${PORT}` in `~/owner.pid`...
#completed
Found. Notice what `ss` actually told you: the program is `python3` -
the interpreter, not the service. `ps -p <pid> -o args=` prints the full
command line and reveals which script it is really running.
::

::tip{title="Empty owner column?"}
`ss -p` can only name processes whose `/proc` entry you are allowed to
read. Sockets belonging to other users - most system services - stay
anonymous until you ask again as root: `sudo ss -tlnp`.
::
