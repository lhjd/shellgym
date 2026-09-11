---
title: Identify and query an unknown service
requires: [systemd, python3]
vars:
  PORT: { pick: ["9391", "9392", "9393"] }
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
  - name: start_scene
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
      rm -rf "$GYM_USER_HOME/triage"
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
      for pair in archive-sync:9394 archive-sync-2:9395; do
        unit=${pair%%:*}
        port=${pair##*:}
        systemd-run --collect --quiet --unit="$unit" --uid="$GYM_USER" \
          /opt/archive-sync/archive-sync "$port" || {
          echo "systemd-run refused to start $unit" >&2
          exit 1
        }
        wait_port --timeout 20 "$port" || {
          echo "$unit did not come up on port $port" >&2
          systemctl status "$unit" --no-pager 2>&1 | tail -10 >&2
          exit 1
        }
      done
tasks:
  port_recorded:
    timeout: 60
    check: |
      wait_file_contains "$GYM_USER_HOME/triage/port.txt" "^$PORT\s*$"
    hint: |
      echo "Three ports in the 939x range are listening. Only one of them answers an HTTP request - try each with curl, and give it -m 3 so a silent one does not hold your terminal."
    solve: |
      mkdir -p ~/triage
      ss -tlnp
      curl -s -m 3 http://127.0.0.1:$PORT/status
      echo $PORT > ~/triage/port.txt
  build_recorded:
    needs: [port_recorded]
    check: |
      wait_file_contains "$GYM_USER_HOME/triage/build.txt" "$BUILD"
    hint: |
      echo "Same request as the one that identified the service - this time send its output into ~/triage/build.txt instead of the screen."
    solve: |
      curl -s http://127.0.0.1:$PORT/status > ~/triage/build.txt
  owner_recorded:
    needs: [port_recorded]
    check: |
      PID=$(pgrep -f "notes-ap[i] $PORT" | head -1)
      if [ -z "$PID" ]; then
        echo "notes-api is not running on port $PORT" >&2
        exit 2
      fi
      wait_file_contains "$GYM_USER_HOME/triage/owner.pid" "^$PID\s*$"
    hint: |
      echo "The owner column of ss -tlnp holds pid=<number> for the row whose Local Address ends in :${PORT}."
    solve: |
      ss -tlnpH sport = :$PORT | grep -o 'pid=[0-9]*' | cut -d= -f2 > ~/triage/owner.pid
      cat ~/triage/owner.pid
---

Last rep, and nothing new in it - only everything at once.

Three services are listening in the `939x` range on this host. One of
them is `notes-api`, whose `/status` endpoint answers with JSON;
the other two accept connections and say nothing at all. Nobody has
documented which is which.

Produce a small triage report in `~/triage/`:

1. `port.txt` - the port `notes-api` is listening on,
2. `build.txt` - the answer its `/status` endpoint gives,
3. `owner.pid` - the PID of the process behind that port.

::task{name="port_recorded"}
#active
Waiting for the `notes-api` port in `~/triage/port.txt`...
#completed
Identified. A listening port only proves that *something* is there; the
service is whichever one answers the request you expected.
::

::task{name="build_recorded"}
#active
Waiting for the `/status` response in `~/triage/build.txt`...
#completed
Recorded - including the build id, which is what turns "the API is up"
into "the API is up and running the version we shipped".
::

::task{name="owner_recorded"}
#active
Waiting for the owning PID in `~/triage/owner.pid`...
#completed
Port, service, process. That is the whole loop: `ss` to find out what is
listening and who owns it, `curl` or `nc` to find out whether it still
answers. Everything else in networking is a bigger version of these
three questions.
::
