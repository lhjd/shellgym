# Talking to the service

Knowing that something listens on a port is half the story. The other
half is asking it a question and reading the answer. This module points
`curl` - the universal command-line HTTP client - at `notes-api`.

`notes-api` is a small HTTP service that stores short text notes. These
are all the endpoints you will need, and every rep repeats the ones it
uses:

| Endpoint | What it does |
|---|---|
| `GET /status` | reports health and the running build id |
| `GET /notes` | a permanent redirect to `/api/notes` |
| `GET /api/notes` | the stored notes, as JSON |
| `POST /api/notes` | accepts one new note, as JSON |

A URL for a service on this machine looks like
`http://127.0.0.1:<port>/<endpoint>` - the same loopback address you
have been reading out of `ss`.
