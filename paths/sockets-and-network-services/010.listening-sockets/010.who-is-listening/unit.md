---
title: Find the port a service listens on
requires: [systemd, python3]
vars:
  PORT: { pick: ["9301", "9302", "9303", "9304"] }
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
      rm -f "$GYM_USER_HOME/service-port.txt"
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
tasks:
  inspected:
    check: |
      wait_exec '(^|/)(ss|netstat|lsof)( |$)'
    hint: |
      echo "Ask ss for TCP sockets (-t) that are listening (-l), with numeric ports (-n)."
    solve: |
      ss -tln
  port_found:
    needs: [inspected]
    check: |
      wait_file_contains "$GYM_USER_HOME/service-port.txt" "^$PORT\s*$"
    hint: |
      if [ -s "$GYM_USER_HOME/service-port.txt" ]; then
        echo "The file should hold the port number on its own, nothing else. Right now it starts with: $(head -c 60 "$GYM_USER_HOME/service-port.txt")"
      else
        echo "The port is the number after the colon in the Local Address:Port column, somewhere in the 93xx range. Put just that number in the file."
      fi
    solve: |
      echo $PORT > ~/service-port.txt
---

A small HTTP service called `notes-api` is running on this machine,
listening on some TCP port between 9300 and 9310. Nobody wrote down
which one.

`ss` prints sockets. Three options turn its firehose into the list you
want:

```
ss -tln
```

- `-t` - TCP sockets only (`-u` asks for UDP instead),
- `-l` - only sockets in the **listening** state,
- `-n` - numeric: print `:9999`, do not translate port numbers into
  service names.

::task{name="inspected"}
#active
Waiting for you to look at the listening sockets...
#completed
There it is - a listener in the 93xx range, in the
**Local Address:Port** column.
::

Now write that port number - just the number, nothing else - into
`~/service-port.txt`:

::task{name="port_found"}
#active
Waiting for the port number in `~/service-port.txt`...
#completed
Recorded. "Which port is it on?" is the opening question of almost every
service investigation, and one `ss` line answers it.
::

::tip{title="Filter at the source"}
`ss` accepts a filter expression after its options: `ss -tln 'sport = :8080'`
shows only sockets whose source port is 8080. Useful once you know what
you are looking for.
::
