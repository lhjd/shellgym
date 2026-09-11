---
title: Send a request body
requires: [systemd, python3]
vars:
  PORT: { pick: ["9361", "9362", "9363"] }
  NOTE: { pick: ["n-5510", "n-5511", "n-5512", "n-5513"] }
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
      rm -f /var/lib/notes-api/received.log
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
  record_accepted:
    check: |
      wait_file_contains /var/lib/notes-api/received.log "^$NOTE$"
    hint: |
      if [ -s /var/lib/notes-api/received.log ]; then
        echo "The server accepted a record, but the last one it logged is $(tail -1 /var/lib/notes-api/received.log) - not ${NOTE}. Check the id in your JSON."
      else
        echo "Nothing has reached the server yet. -d <data> both sends a body and switches curl to POST; run it without -s once and read the reply - a 400 means the server could not parse your JSON."
      fi
    solve: |
      curl -s -X POST -H "Content-Type: application/json" -d "{\"id\":\"$NOTE\"}" http://127.0.0.1:$PORT/api/notes
---

So far you have only asked for things. Storing a new note means sending
data, and that means a `POST` with a body:

- `-d '<data>'` supplies the request body (and, on its own, already
  switches `curl` from `GET` to `POST`),
- `-H '<name>: <value>'` adds a request header - servers that consume
  JSON expect `Content-Type: application/json`,
- `-X POST` states the method explicitly, which is worth typing while
  you are learning even when `-d` implies it.

`notes-api` accepts one note at a time on `POST /api/notes`, and the
body it wants is a JSON object with a single `id` field:

```
{"id": "n-0000"}
```

Mind the quoting: JSON needs its double quotes, and they have to survive
the shell - so wrap the whole body in single quotes, or put a backslash
in front of every inner quote.

Store a note with the id `${NOTE}` on the server on port `${PORT}`:

::task{name="record_accepted"}
#active
Waiting for the server to accept `${NOTE}`...
#completed
Accepted - the server answered `201 Created` and wrote the note to its
log. Requests, headers, redirects, and now a request body: that is most
of what an operator ever needs from an HTTP client.
::

::tip{title="Read the answer while you debug"}
Drop the `-s` and add `-i` and you see the status line of the reply.
A `400` means the server did not understand the body you sent - almost
always a quoting accident rather than a server problem.
::
