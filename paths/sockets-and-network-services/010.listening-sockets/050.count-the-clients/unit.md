---
title: List established connections
requires: [systemd, python3]
vars:
  PORT: { pick: ["9341", "9342", "9343"] }
  CLIENTS: { pick: ["2", "3", "4"] }
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
  - name: install_probe
    run: |
      install -d -m 0755 /opt/stockroom
      cat > /opt/stockroom/stockroom-probe <<'PY'
      #!/usr/bin/env python3
      """Stockroom import probe - holds one idle TCP connection open."""
      import socket
      import sys
      import time

      sock = socket.create_connection(("127.0.0.1", int(sys.argv[1])), timeout=10)
      time.sleep(int(sys.argv[2]))
      sock.close()
      PY
      chmod 0755 /opt/stockroom/stockroom-probe
  - name: start_stockroom
    run: |
      for u in stockroom stockroom-import stockroom-admin archive-sync archive-sync-2; do
        systemctl stop "$u.service" 2>/dev/null || true
      done
      for i in 1 2 3 4 5; do
        systemctl stop "stockroom-probe-$i.service" 2>/dev/null || true
      done
      pkill -f 'stockroom-serve[r] ' 2>/dev/null || true
      pkill -f 'archive-syn[c] ' 2>/dev/null || true
      pkill -f 'stockroom-prob[e] ' 2>/dev/null || true
      sleep 0.5
      rm -f "$GYM_USER_HOME/clients.txt"
      systemd-run --collect --quiet --unit=stockroom --uid="$GYM_USER" --setenv=STOCKROOM_BUILD="$BUILD" \
        /opt/stockroom/stockroom-server "$PORT" 0.0.0.0
      wait_port --timeout 20 "$PORT" || {
        echo "stockroom did not start on port $PORT" >&2
        exit 1
      }
  - name: start_clients
    run: |
      i=1
      while [ "$i" -le "$CLIENTS" ]; do
        systemd-run --collect --quiet --unit="stockroom-probe-$i" --uid="$GYM_USER" \
          /opt/stockroom/stockroom-probe "$PORT" 3600
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
actually *connected* to Stockroom Server has a socket too, in the
`ESTAB` (established) state.

Drop the `-l` and `ss` stops restricting itself to listeners. Two extra
pieces of its filter language narrow the result down:

```
ss -tn state established dst :${PORT}
```

- `state established` - only fully established connections (other
  useful values: `listening`, `time-wait`, `syn-sent`),
- `dst :${PORT}` - only sockets whose *remote* end is port `${PORT}` -
  in other words, the client side of connections **to** Stockroom
  Server.

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
