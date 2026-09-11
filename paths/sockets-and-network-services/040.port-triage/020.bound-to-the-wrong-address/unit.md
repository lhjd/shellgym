---
title: Rebind a service to the right address
requires: [systemd, python3]
vars:
  PORT: { pick: ["9381", "9382", "9383"] }
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
      pkill -f 'stockroom-serve[r] ' 2>/dev/null || true
      pkill -f 'archive-syn[c] ' 2>/dev/null || true
      pkill -f 'stockroom-prob[e] ' 2>/dev/null || true
      sleep 0.5
      systemd-run --collect --quiet --unit=stockroom --uid="$GYM_USER" --setenv=STOCKROOM_BUILD="$BUILD" \
        /opt/stockroom/stockroom-server "$PORT" 127.0.0.1
      wait_port --timeout 20 "$PORT" || {
        echo "stockroom did not start on port $PORT" >&2
        exit 1
      }
tasks:
  wildcard_bound:
    timeout: 60
    check: |
      for attempt in $(seq 1 40); do
        if ss -tlnH "sport = :$PORT" 2>/dev/null | awk '{print $4}' | grep -qE '^(0\.0\.0\.0|\*|\[::\]):'; then
          exit 0
        fi
        sleep 1
      done
      if [ -n "$(ss -tlnH "sport = :$PORT" 2>/dev/null)" ]; then
        hint_exit "Port ${PORT} is listening again, but still on 127.0.0.1 only. A port can only be bound once - the old process has to stop before the new one can take it."
      fi
      hint_exit "Nothing is listening on port ${PORT}. Start the server again, passing the address it should bind as the second argument."
    hint: |
      echo "Find the running server first (its command line ends in ${PORT} 127.0.0.1), stop it, then start it again with 0.0.0.0 as the second argument."
    solve: |
      ss -tlnp sport = :$PORT
      pkill -f "stockroom-server $PORT"
      #!wait 1
      /opt/stockroom/stockroom-server $PORT 0.0.0.0 &
      #!wait 3
---

Stockroom Server is up on port `${PORT}` and answers perfectly from this
machine - and not at all from anywhere else. You met the reason in the
first module: it is bound to `127.0.0.1`, so the socket only ever
accepts connections that never left the host.

The server takes the address to bind as its second argument:

```
/opt/stockroom/stockroom-server <port> <address>
```

Get it listening on port `${PORT}` on **all** addresses. Two things have
to happen in the right order, because a TCP port can only be bound by
one socket at a time: the process holding `${PORT}` now has to go before
a new one can take it.

::task{name="wildcard_bound"}
#active
Waiting for a listener on `0.0.0.0:${PORT}`...
#completed
Rebound, and now reachable from the network. This is the single most
common version of "works on the server, not from my laptop" - and in
real services it is a line in a config file (`bind_address`, `listen`,
`--host`) rather than an argument, but the socket table tells the same
story either way.
::

::tip{title="Binding to everything is a decision"}
`0.0.0.0` exposes the service to every network the host is attached to.
For an admin endpoint or a database, `127.0.0.1` - or the address of one
internal interface - is usually the right answer, and the firewall is
the second line of defence, not the first.
::
