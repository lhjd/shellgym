# Talking to the service

Knowing that something listens on a port is half the story. The other
half is asking it a question and reading the answer. This module points
`curl` - the universal command-line HTTP client - at Stockroom Server.

**Stockroom Server** serves inventory records over plain HTTP. These are
all the endpoints you will need, and every rep repeats the ones it uses:

| Endpoint | What it does |
|---|---|
| `GET /status` | reports health and the running build id |
| `GET /items` | a permanent redirect to `/api/items` |
| `GET /api/items` | the inventory records, as JSON |
| `POST /api/items` | accepts one new record, as JSON |

A URL for a service on this machine looks like
`http://127.0.0.1:<port>/<endpoint>` - the same loopback address you
have been reading out of `ss`.
