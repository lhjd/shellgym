---
title: Find the process behind a port
requires: [systemd, python3]
vars:
  PORT: { pick: ["9305", "9306", "9307"] }
  AGENT_PORT: { value: "9315" }
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
  - name: start_stockroom
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
      rm -f "$GYM_USER_HOME/owner.pid"
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
      systemd-run --collect --quiet --unit=stockroom-import --uid="$GYM_USER" --setenv=STOCKROOM_BUILD="$BUILD" \
        /opt/stockroom/stockroom-server "$AGENT_PORT" 0.0.0.0 || {
        echo "systemd-run refused to start stockroom-import" >&2
        exit 1
      }
      wait_port --timeout 20 "$AGENT_PORT" || {
        echo "stockroom-import did not come up on port $AGENT_PORT" >&2
        systemctl status stockroom-import --no-pager 2>&1 | tail -10 >&2
        exit 1
      }
tasks:
  owner_found:
    check: |
      PID=$(pgrep -f "stockroom-serve[r] $PORT" | head -1)
      if [ -z "$PID" ]; then
        echo "Stockroom Server is not running on port $PORT" >&2
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

Two services are listening on this machine right now: the Stockroom
inventory API and a Stockroom import agent. One of them owns port
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
