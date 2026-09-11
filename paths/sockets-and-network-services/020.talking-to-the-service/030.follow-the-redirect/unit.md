---
title: Follow a redirect
requires: [systemd, python3]
vars:
  PORT: { pick: ["9357", "9358", "9359"] }
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
      rm -f "$GYM_USER_HOME/items.json"
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
tasks:
  items_saved:
    check: |
      wait_file_contains "$GYM_USER_HOME/items.json" "A-2277"
    hint: |
      if grep -q moved "$GYM_USER_HOME/items.json" 2>/dev/null; then
        echo "What you saved is the redirect itself, not the inventory. curl does not follow redirects unless you ask - the option is one letter, and it stands for 'location'."
      else
        echo "Request http://127.0.0.1:${PORT}/items and make curl follow the 301 it answers with, then send the result into ~/items.json."
      fi
    solve: |
      curl -s http://127.0.0.1:$PORT/items
      curl -sL http://127.0.0.1:$PORT/items > ~/items.json
      cat ~/items.json
---

The inventory used to live at `/items`. It moved, and the old address
now answers with a redirect:

```
HTTP/1.0 301 Moved Permanently
Location: /api/items
```

A browser would follow that `Location` header without telling you.
`curl` does not - it hands you the redirect response and stops, which is
exactly why a naive `curl` of a moved endpoint returns a body that is
not the data you wanted. `-L` tells it to follow.

Fetch `http://127.0.0.1:${PORT}/items`, following the redirect, and save
the inventory records into `~/items.json`:

::task{name="items_saved"}
#active
Waiting for the inventory records in `~/items.json`...
#completed
Followed. Try it once without `-L` and compare: an empty-looking result
from `curl` is very often an unfollowed redirect rather than a broken
service.
::
