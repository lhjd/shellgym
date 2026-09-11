---
title: Identify and query an unknown service
requires: [systemd, python3]
vars:
  PORT: { pick: ["9391", "9392", "9393"] }
  BUILD: { shell: "head -c4 /dev/urandom | od -An -tx1 | tr -d ' \\n'" }
init:
  - name: install_stockroom
    run: |
      install -d -m 0755 /opt/stockroom
      cat > /opt/stockroom/stockroom-server <<'PY'
      #!/usr/bin/env python3
      """Stockroom Server - a small inventory HTTP API used as a practice target."""
      import json
      import os
      import sys
      from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer

      PORT = int(sys.argv[1])
      BIND = sys.argv[2] if len(sys.argv) > 2 else "0.0.0.0"
      BUILD = os.environ.get("STOCKROOM_BUILD", "dev")
      INBOX = os.environ.get("STOCKROOM_INBOX", "/var/lib/stockroom/received.log")

      ITEMS = [
          {"asset": "A-1041", "item": "shelf bracket", "location": "aisle-3", "qty": 12},
          {"asset": "A-2277", "item": "pallet jack", "location": "dock-1", "qty": 2},
          {"asset": "A-3390", "item": "label roll", "location": "aisle-7", "qty": 48},
      ]


      class Handler(BaseHTTPRequestHandler):
          server_version = "StockroomServer/1.4"
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
              elif self.path == "/items":
                  self.reply(301, {"moved": "/api/items"}, [("Location", "/api/items")])
              elif self.path == "/api/items":
                  self.reply(200, {"items": ITEMS})
              else:
                  self.reply(404, {"error": "no such endpoint"})

          do_HEAD = do_GET

          def do_POST(self):
              length = int(self.headers.get("Content-Length") or 0)
              raw = self.rfile.read(length).decode("utf-8", "replace")
              if self.path != "/api/items":
                  self.reply(404, {"error": "no such endpoint"})
                  return
              try:
                  asset = str(json.loads(raw)["asset"])
              except Exception:
                  self.reply(400, {"error": "expected a JSON body with an asset field"})
                  return
              os.makedirs(os.path.dirname(INBOX), exist_ok=True)
              with open(INBOX, "a") as fh:
                  fh.write(asset + "\n")
              self.reply(201, {"accepted": asset})

          def log_message(self, *args):
              pass


      ThreadingHTTPServer.allow_reuse_address = True
      ThreadingHTTPServer((BIND, PORT), Handler).serve_forever()
      PY
      chmod 0755 /opt/stockroom/stockroom-server
      mkdir -p /var/lib/stockroom
      chown "$GYM_USER" /var/lib/stockroom
  - name: install_agents
    run: |
      install -d -m 0755 /opt/stockroom
      cat > /opt/stockroom/archive-sync <<'PY'
      #!/usr/bin/env python3
      """Archive sync agent - accepts connections and closes them immediately."""
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
      chmod 0755 /opt/stockroom/archive-sync
  - name: start_scene
    run: |
      for u in stockroom stockroom-import stockroom-admin archive-sync archive-sync-2; do
        systemctl stop "$u.service" 2>/dev/null || true
      done
      for i in 1 2 3 4 5; do
        systemctl stop "stockroom-probe-$i.service" 2>/dev/null || true
      done
      pkill -u "$GYM_USER" -f 'stockroom-serve[r] ' 2>/dev/null || true
      pkill -u "$GYM_USER" -f 'archive-syn[c] ' 2>/dev/null || true
      pkill -u "$GYM_USER" -f 'stockroom-prob[e] ' 2>/dev/null || true
      sleep 0.5
      rm -rf "$GYM_USER_HOME/triage"
      systemd-run --collect --quiet --unit=stockroom --uid="$GYM_USER" --setenv=STOCKROOM_BUILD="$BUILD" \
        /opt/stockroom/stockroom-server "$PORT" 0.0.0.0 || {
        echo "systemd-run refused to start stockroom" >&2
        exit 1
      }
      wait_port --timeout 20 "$PORT" || {
        echo "stockroom did not come up on port $PORT" >&2
        systemctl status stockroom --no-pager 2>&1 | tail -10 >&2
        exit 1
      }
      for pair in archive-sync:9394 archive-sync-2:9395; do
        unit=${pair%%:*}
        port=${pair##*:}
        systemd-run --collect --quiet --unit="$unit" --uid="$GYM_USER" \
          /opt/stockroom/archive-sync "$port" || {
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
      PID=$(pgrep -f "stockroom-serve[r] $PORT" | head -1)
      if [ -z "$PID" ]; then
        echo "Stockroom Server is not running on port $PORT" >&2
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
them is Stockroom Server, whose `/status` endpoint answers with JSON;
the other two accept connections and say nothing at all. Nobody has
documented which is which.

Produce a small triage report in `~/triage/`:

1. `port.txt` - the port Stockroom Server is listening on,
2. `build.txt` - the answer its `/status` endpoint gives,
3. `owner.pid` - the PID of the process behind that port.

::task{name="port_recorded"}
#active
Waiting for the Stockroom Server port in `~/triage/port.txt`...
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
