---
title: Speak HTTP by hand
requires: [systemd, python3]
vars:
  PORT: { pick: ["9365", "9366", "9367"] }
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
      pkill -f 'stockroom-serve[r] ' 2>/dev/null || true
      pkill -f 'archive-syn[c] ' 2>/dev/null || true
      pkill -f 'stockroom-prob[e] ' 2>/dev/null || true
      sleep 0.5
      rm -f "$GYM_USER_HOME/raw-response.txt"
      systemd-run --collect --quiet --unit=stockroom --uid="$GYM_USER" --setenv=STOCKROOM_BUILD="$BUILD" \
        /opt/stockroom/stockroom-server "$PORT" 0.0.0.0
      wait_port --timeout 20 "$PORT" || {
        echo "stockroom did not start on port $PORT" >&2
        exit 1
      }
tasks:
  response_saved:
    timeout: 60
    check: |
      command -v nc >/dev/null 2>&1 || \
        hint_exit "nc is not installed on this host. It ships as netcat-openbsd on Debian and Ubuntu and as nmap-ncat on Fedora and Rocky - install it with your package manager and this rep is back in business."
      wait_file_contains --timeout 40 "$GYM_USER_HOME/raw-response.txt" "^HTTP/1\.[01] 200" || \
        hint_exit "No HTTP status line in ~/raw-response.txt yet. The server answers only once it has seen a completely empty line, so the request needs a trailing blank line of its own."
      wait_file_contains --timeout 10 "$GYM_USER_HOME/raw-response.txt" "$BUILD" || \
        hint_exit "That is an HTTP response, but not the one /status sends - check the path in your request line."
    hint: |
      echo "Pipe the request text into nc and redirect nc's output into the file. printf is the command that can produce the \\r\\n line endings HTTP asks for."
    solve: |
      printf 'GET /status HTTP/1.0\r\n\r\n' | nc 127.0.0.1 $PORT > ~/raw-response.txt
      cat ~/raw-response.txt
---

HTTP has no magic in it: it is lines of text sent over a TCP
connection. The smallest request a server will answer is a request line
and an empty line, with every line ending in carriage-return plus
newline (`\r\n`):

```
GET /status HTTP/1.0
<empty line>
```

`nc` can deliver exactly that. Anything you feed to its standard input
goes down the socket, and anything the server sends back comes out of
its standard output - so a pipe in and a redirect out capture the whole
exchange.

The one detail to get right is the line endings. `echo` gives you plain
newlines; `printf` interprets `\r` and `\n` in its format string, which
is what the protocol asks for.

Fetch `/status` from Stockroom Server on port `${PORT}` this way, and
save everything the server sends back - status line, headers and body -
into `~/raw-response.txt`:

::task{name="response_saved"}
#active
Waiting for a raw HTTP response in `~/raw-response.txt`...
#completed
That is the whole protocol, headers and all. This is the trick for
services that are not HTTP at all: point `nc` at an SMTP, Redis, or
plain-text port and type their protocol yourself.
::

::tip{title="Or type it live"}
Run `nc 127.0.0.1 ${PORT}` with no redirection, type
`GET /status HTTP/1.0`, then press `Enter` twice - the second `Enter` is
the empty line that ends the request. The answer prints straight into
your terminal.
::
