---
title: Test whether a port is open
requires: [systemd, python3]
vars:
  OPEN: { pick: ["9501", "9502", "9503"] }
  BUILD: { shell: "head -c4 /dev/urandom | od -An -tx1 | tr -d ' \\n'" }
init:
  - name: install_netcat
    run: |
      command -v nc >/dev/null 2>&1 && exit 0
      if command -v apt-get >/dev/null 2>&1; then
        DEBIAN_FRONTEND=noninteractive apt-get install -y -q netcat-openbsd >/dev/null 2>&1 || {
          apt-get update -qq >/dev/null 2>&1 || true
          DEBIAN_FRONTEND=noninteractive apt-get install -y -q netcat-openbsd >/dev/null 2>&1 || true
        }
      elif command -v dnf >/dev/null 2>&1; then
        dnf install -y -q nmap-ncat >/dev/null 2>&1 || true
      elif command -v apk >/dev/null 2>&1; then
        apk add --no-cache netcat-openbsd >/dev/null 2>&1 || true
      fi
      command -v nc >/dev/null 2>&1 || echo "nc is not installed and could not be installed here" >&2
      exit 0
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
      rm -f "$GYM_USER_HOME/open-port.txt"
      systemd-run --collect --quiet --unit=stockroom-import --uid="$GYM_USER" --setenv=STOCKROOM_BUILD="$BUILD" \
        /opt/stockroom/stockroom-server "$OPEN" 0.0.0.0 || {
        echo "systemd-run refused to start stockroom-import" >&2
        exit 1
      }
      wait_port --timeout 20 "$OPEN" || {
        echo "stockroom-import did not come up on port $OPEN" >&2
        systemctl status stockroom-import --no-pager 2>&1 | tail -10 >&2
        exit 1
      }
tasks:
  open_found:
    check: |
      command -v nc >/dev/null 2>&1 || \
        hint_exit "nc is not installed on this host. It ships as netcat-openbsd on Debian and Ubuntu and as nmap-ncat on Fedora and Rocky - install it with your package manager and this rep is back in business."
      wait_file_contains "$GYM_USER_HOME/open-port.txt" "^$OPEN\s*$"
    hint: |
      echo "nc -z connects without sending anything; add -v so it tells you which attempt succeeded and which was refused. Try 9501, 9502 and 9503 in turn."
    solve: |
      nc -zv -w 2 127.0.0.1 9501
      nc -zv -w 2 127.0.0.1 9502
      nc -zv -w 2 127.0.0.1 9503
      echo $OPEN > ~/open-port.txt
---

A colleague says the Stockroom import listener is "on 9501, or maybe
9502, or 9503". You could read `ss` output - but the question they are
really asking is whether a connection succeeds, and that is what `nc`
answers directly:

```
nc -zv -w 2 127.0.0.1 9501
```

- `-z` - zero I/O: connect, then hang up without sending anything,
- `-v` - verbose: say `succeeded!` or `Connection refused` instead of
  staying silent,
- `-w 2` - give up after two seconds, so a port that silently drops
  packets (a firewall, usually) does not hang your terminal.

Find which of `9501`, `9502` and `9503` accepts a connection, and write
that port into `~/open-port.txt`:

::task{name="open_found"}
#active
Waiting for the open port in `~/open-port.txt`...
#completed
Knocked. Note the three answers you can get, and what each means:
*succeeded* - something is listening; *connection refused* - the host
answered, but nothing is bound there; *timeout* - nothing answered at
all, which usually means a firewall rather than a stopped service.
::

::tip{title="ss or nc?"}
`ss` shows what *this* host has open; `nc -z` shows what is reachable
*from where you are standing*. When they disagree, the gap between them
is the firewall, the routing table, or a bind address - which is exactly
what makes running both worthwhile.
::
