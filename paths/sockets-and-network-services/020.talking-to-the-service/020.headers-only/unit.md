---
title: Read the response headers
requires: [systemd, python3]
vars:
  PORT: { pick: ["9354", "9355", "9356"] }
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
      rm -f "$GYM_USER_HOME/headers.txt"
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
  headers_saved:
    check: |
      wait_file_contains "$GYM_USER_HOME/headers.txt" "NotesAPI"
    hint: |
      if [ -f "$GYM_USER_HOME/headers.txt" ]; then
        echo "headers.txt has no Server: line in it - that looks like a response body, not the headers. One capital-letter option asks curl for the headers alone."
      else
        echo "The option is a single capital letter, and it is not -h (that one prints curl's own help)."
      fi
    solve: |
      curl -sI http://127.0.0.1:$PORT/status > ~/headers.txt
      cat ~/headers.txt
---

Every HTTP response starts with a status line and a block of headers,
and `curl` hides them by default. `-I` asks for them on their own - it
sends a `HEAD` request, so the server describes the resource without
sending its body:

```
curl -I http://127.0.0.1:${PORT}/status
```

You get back something like:

```
HTTP/1.0 200 OK
Server: NotesAPI/1.4
Content-Type: application/json
Content-Length: 42
```

The first line is the one that settles most arguments: `200` means the
request worked, `301`/`302` a redirect, `401`/`403` a permission
problem, `404` a wrong URL, and anything starting with `5` a failure
inside the server.

Save the response headers of `/status` into `~/headers.txt`:

::task{name="headers_saved"}
#active
Waiting for the response headers in `~/headers.txt`...
#completed
Captured. When you want headers *and* body, `-i` prints both, and
`-D <file>` writes the headers to a file while the body goes wherever
you sent it.
::
