---
title: Read the same list with netstat
requires: [systemd, python3]
vars:
  PORT: { pick: ["9308", "9309", "9310"] }
  BUILD: { shell: "head -c4 /dev/urandom | od -An -tx1 | tr -d ' \\n'" }
init:
  - name: install_netstat
    run: |
      command -v netstat >/dev/null 2>&1 && exit 0
      if command -v apt-get >/dev/null 2>&1; then
        DEBIAN_FRONTEND=noninteractive apt-get install -y -q net-tools >/dev/null 2>&1 || {
          apt-get update -qq >/dev/null 2>&1 || true
          DEBIAN_FRONTEND=noninteractive apt-get install -y -q net-tools >/dev/null 2>&1 || true
        }
      elif command -v dnf >/dev/null 2>&1; then
        dnf install -y -q net-tools >/dev/null 2>&1 || true
      elif command -v apk >/dev/null 2>&1; then
        apk add --no-cache net-tools >/dev/null 2>&1 || true
      fi
      command -v netstat >/dev/null 2>&1 || echo "netstat is not installed and could not be installed here" >&2
      exit 0
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
  - name: start_service
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
      rm -f "$GYM_USER_HOME/legacy-port.txt"
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
tasks:
  netstat_used:
    check: |
      command -v netstat >/dev/null 2>&1 || \
        hint_exit "netstat is not installed on this host. It ships in the net-tools package - install that with your package manager, then come back."
      wait_exec '(^|/)netstat( |$)'
    hint: |
      echo "netstat takes the same three short options as ss for this job: TCP, listening, numeric."
    solve: |
      netstat -tln
  port_found:
    needs: [netstat_used]
    check: |
      wait_file_contains "$GYM_USER_HOME/legacy-port.txt" "^$PORT\s*$"
    hint: |
      echo "Same column as before: Local Address, the number after the colon. notes-api is the listener in the 93xx range."
    solve: |
      echo $PORT > ~/legacy-port.txt
---

Before `ss` there was `netstat`, and half the runbooks, blog posts, and
colleagues you will meet still speak it. It ships in the `net-tools`
package, which most distributions no longer install by default - but
when it is there, the flags will feel familiar:

| Question | `ss` | `netstat` |
|---|---|---|
| TCP listeners | `ss -tln` | `netstat -tln` |
| ...with owners | `ss -tlnp` | `netstat -tlnp` |
| TCP connections | `ss -tn` | `netstat -tn` |
| UDP sockets | `ss -uln` | `netstat -uln` |
| everything | `ss -a` | `netstat -a` |

`notes-api` is listening somewhere in the 93xx range again. This time,
find it with `netstat`:

::task{name="netstat_used"}
#active
Waiting for a `netstat` run...
#completed
Same picture, older tool. The differences are mostly cosmetic: `netstat`
walks `/proc/net/*` text files, while `ss` asks the kernel directly over
a netlink socket - which is why `ss` stays fast on a host with tens of
thousands of sockets.
::

Write the port you found into `~/legacy-port.txt`:

::task{name="port_found"}
#active
Waiting for the port number in `~/legacy-port.txt`...
#completed
Read both ways now. Reach for `ss` by default; recognize `netstat` when
you inherit a script that uses it.
::
