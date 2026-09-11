---
title: Tell a loopback listener from a public one
requires: [systemd, python3]
vars:
  PUBLIC_PORT: { pick: ["9321", "9322", "9323"] }
  LOCAL_PORT: { pick: ["9331", "9332", "9333"] }
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
      rm -f "$GYM_USER_HOME/public-port.txt"
      systemd-run --collect --quiet --unit=notes-api --uid="$GYM_USER" --setenv=NOTES_BUILD="$BUILD" \
        /opt/notes-api/notes-api "$PUBLIC_PORT" 0.0.0.0 || {
        echo "systemd-run refused to start notes-api" >&2
        exit 1
      }
      wait_port --timeout 20 "$PUBLIC_PORT" || {
        echo "notes-api did not come up on port $PUBLIC_PORT" >&2
        systemctl status notes-api --no-pager 2>&1 | tail -10 >&2
        exit 1
      }
      systemd-run --collect --quiet --unit=notes-api-admin --uid="$GYM_USER" --setenv=NOTES_BUILD="$BUILD" \
        /opt/notes-api/notes-api "$LOCAL_PORT" 127.0.0.1 || {
        echo "systemd-run refused to start notes-api-admin" >&2
        exit 1
      }
      wait_port --timeout 20 "$LOCAL_PORT" || {
        echo "notes-api-admin did not come up on port $LOCAL_PORT" >&2
        systemctl status notes-api-admin --no-pager 2>&1 | tail -10 >&2
        exit 1
      }
tasks:
  public_found:
    check: |
      wait_file_contains "$GYM_USER_HOME/public-port.txt" "^$PUBLIC_PORT\s*$"
    hint: |
      if grep -q "$LOCAL_PORT" "$GYM_USER_HOME/public-port.txt" 2>/dev/null; then
        echo "Port ${LOCAL_PORT} is bound to 127.0.0.1, the loopback address - only this machine can reach it. The other listener is the one you want."
      else
        echo "Run ss -tln and compare the Local Address column of the two rows: one of them is bound to a single address, the other to all of them."
      fi
    solve: |
      ss -tln
      echo $PUBLIC_PORT > ~/public-port.txt
---

Two listeners are up on this host:

- the `notes-api` service on port `${PUBLIC_PORT}`,
- an internal admin endpoint on port `${LOCAL_PORT}`.

A colleague reports that one of them cannot be reached from their
laptop. The **Local Address** column explains why, and it is worth
reading slowly:

- `127.0.0.1:<port>` - bound to the loopback address. Connections from
  this machine work; connections from anywhere else never arrive.
- `0.0.0.0:<port>` - bound to every IPv4 address the machine has, so
  anyone who can route to the host may connect. (`netstat` and `ss`
  sometimes print this as `*:<port>`; the IPv6 equivalent is `[::]`.)

Work out which of the two ports is reachable from another machine, and
write it into `~/public-port.txt`:

::task{name="public_found"}
#active
Waiting for the publicly reachable port in `~/public-port.txt`...
#completed
That one column - a bind address - is behind a large share of "but it
works on the server!" tickets. Reading it first saves hours of blaming
the firewall.
::
