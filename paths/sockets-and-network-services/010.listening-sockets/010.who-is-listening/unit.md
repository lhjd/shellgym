---
title: Find the port a service listens on
requires: [systemd, python3]
vars:
  PORT: { pick: ["9301", "9302", "9303", "9304"] }
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
      rm -f "$GYM_USER_HOME/stockroom-port.txt"
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
  inspected:
    check: |
      wait_exec '(^|/)(ss|netstat|lsof)( |$)'
    hint: |
      echo "Ask ss for TCP sockets (-t) that are listening (-l), with numeric ports (-n)."
    solve: |
      ss -tln
  port_found:
    needs: [inspected]
    check: |
      wait_file_contains "$GYM_USER_HOME/stockroom-port.txt" "^$PORT\s*$"
    hint: |
      if [ -s "$GYM_USER_HOME/stockroom-port.txt" ]; then
        echo "The file should hold the port number on its own, nothing else. Right now it starts with: $(head -c 60 "$GYM_USER_HOME/stockroom-port.txt")"
      else
        echo "The port is the number after the colon in the Local Address:Port column, somewhere in the 93xx range. Put just that number in the file."
      fi
    solve: |
      echo $PORT > ~/stockroom-port.txt
---

Stockroom Server is running on this machine, listening on some TCP port
between 9300 and 9310. Nobody wrote down which one.

`ss` prints sockets. Three options turn its firehose into the list you
want:

```
ss -tln
```

- `-t` - TCP sockets only (`-u` asks for UDP instead),
- `-l` - only sockets in the **listening** state,
- `-n` - numeric: print `:9999`, do not translate port numbers into
  service names.

::task{name="inspected"}
#active
Waiting for you to look at the listening sockets...
#completed
There it is - a listener in the 93xx range, in the
**Local Address:Port** column.
::

Now write that port number - just the number, nothing else - into
`~/stockroom-port.txt`:

::task{name="port_found"}
#active
Waiting for the port number in `~/stockroom-port.txt`...
#completed
Recorded. "Which port is it on?" is the opening question of almost every
service investigation, and one `ss` line answers it.
::

::tip{title="Filter at the source"}
`ss` accepts a filter expression after its options: `ss -tln 'sport = :8080'`
shows only sockets whose source port is 8080. Useful once you know what
you are looking for.
::
