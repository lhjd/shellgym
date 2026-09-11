---
title: Send a request body
requires: [systemd, python3]
vars:
  PORT: { pick: ["9361", "9362", "9363"] }
  ASSET: { pick: ["A-5510", "A-5511", "A-5512", "A-5513"] }
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
      rm -f /var/lib/stockroom/received.log
      systemd-run --collect --quiet --unit=stockroom --uid="$GYM_USER" --setenv=STOCKROOM_BUILD="$BUILD" \
        /opt/stockroom/stockroom-server "$PORT" 0.0.0.0
      wait_port --timeout 20 "$PORT" || {
        echo "stockroom did not start on port $PORT" >&2
        exit 1
      }
tasks:
  record_accepted:
    check: |
      wait_file_contains /var/lib/stockroom/received.log "^$ASSET$"
    hint: |
      if [ -s /var/lib/stockroom/received.log ]; then
        echo "The server accepted a record, but the last one it logged is $(tail -1 /var/lib/stockroom/received.log) - not ${ASSET}. Check the asset id in your JSON."
      else
        echo "Nothing has reached the server yet. -d <data> both sends a body and switches curl to POST; run it without -s once and read the reply - a 400 means the server could not parse your JSON."
      fi
    solve: |
      curl -s -X POST -H "Content-Type: application/json" -d "{\"asset\":\"$ASSET\"}" http://127.0.0.1:$PORT/api/items
---

So far you have only asked for things. Registering a new item means
sending data, and that means a `POST` with a body:

- `-d '<data>'` supplies the request body (and, on its own, already
  switches `curl` from `GET` to `POST`),
- `-H '<name>: <value>'` adds a request header - servers that consume
  JSON expect `Content-Type: application/json`,
- `-X POST` states the method explicitly, which is worth typing while
  you are learning even when `-d` implies it.

Stockroom Server accepts one record at a time on `POST /api/items`, and
the body it wants is a JSON object with a single `asset` field:

```
{"asset": "A-0000"}
```

Mind the quoting: JSON needs its double quotes, and they have to survive
the shell - so wrap the whole body in single quotes, or put a backslash
in front of every inner quote.

Register asset `${ASSET}` with the server on port `${PORT}`:

::task{name="record_accepted"}
#active
Waiting for the server to accept `${ASSET}`...
#completed
Accepted - the server answered `201 Created` and wrote the record to its
log. Requests, headers, redirects, and now a request body: that is most
of what an operator ever needs from an HTTP client.
::

::tip{title="Read the answer while you debug"}
Drop the `-s` and add `-i` and you see the status line of the reply.
A `400` means the server did not understand the body you sent - almost
always a quoting accident rather than a server problem.
::
