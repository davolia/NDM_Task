# X-Forwarded-For Test Stand

## Architecture

```
User ──► nginx1:8081 ──► nginx2:8082 ──► nginx3:8083 ──► app (traefik/whoami)
         172.20.0.11      172.20.0.12      172.20.0.13     172.20.0.20
```

Each nginx is also exposed directly so any single hop or multi-hop path is testable:

| Port | Container | Upstream |
|------|-----------|----------|
| 8081 | nginx1    | → nginx2 → nginx3 → app |
| 8082 | nginx2    | → nginx3 → app |
| 8083 | nginx3    | → app directly |

## How it works

### X-Forwarded-For chain

Each nginx includes `shared/geo_trusted.conf` which defines two nginx variables:

- `$is_trusted_proxy` — set to `1` for `172.20.0.11–13` (the three nginx containers), `0` for everything else.
- `$xff_value` — the value written into the `X-Forwarded-For` header sent upstream:
  - **Request from a trusted proxy** → `"$http_x_forwarded_for, $remote_addr"` (append)
  - **Request from an untrusted source** → `"$remote_addr"` (reset — discards any XFF the client sent)

Each nginx sets exactly `proxy_set_header X-Forwarded-For $xff_value;`. This means:

1. The first nginx in the path always **replaces** whatever `X-Forwarded-For` the client sent with just the client's IP — spoofing is neutralised at the edge.
2. Every subsequent trusted nginx **appends** its predecessor's IP, building the full chain.

### Result seen by the app

| Path | X-Forwarded-For received by app |
|------|----------------------------------|
| user → nginx3 → app | `<ENTRY_IP>` |
| user → nginx2 → nginx3 → app | `<ENTRY_IP>, 172.20.0.12` |
| user → nginx1 → nginx2 → nginx3 → app | `<ENTRY_IP>, 172.20.0.11, 172.20.0.12` |

`<ENTRY_IP>` is whatever IP Docker reports as the source of the curl connection (the Docker bridge gateway, `172.20.0.1`, when running curl on the host machine).

## Start

```bash
docker compose up -d
```

## Test Protocol

Run these commands after `docker compose up -d`. The output is from `traefik/whoami` which prints all received headers.

---

### Test 1 — Single hop (user → nginx3 → app)

```bash
curl -s http://localhost:8083
```

**What to look for in the output:**
```
X-Forwarded-For: 172.20.0.1
```
nginx3 is the entry point. It receives from an untrusted source and resets XFF to `$remote_addr` only.

---

### Test 2 — Two hops (user → nginx2 → nginx3 → app)

```bash
curl -s http://localhost:8082
```

**Expected:**
```
X-Forwarded-For: 172.20.0.1, 172.20.0.12
```
nginx2 resets XFF to the entry IP; nginx3 (trusting nginx2) appends nginx2's IP.

---

### Test 3 — Full chain (user → nginx1 → nginx2 → nginx3 → app)

```bash
curl -s http://localhost:8081
```

**Expected:**
```
X-Forwarded-For: 172.20.0.1, 172.20.0.11, 172.20.0.12
```
Full three-proxy chain recorded.

---

### Test 4 — Spoofed XFF, single hop (security check)

```bash
curl -s -H "X-Forwarded-For: 1.2.3.4" http://localhost:8083
```

**Expected — spoofed value is absent:**
```
X-Forwarded-For: 172.20.0.1
```
nginx3 sees an untrusted source → resets XFF to `$remote_addr`, discarding `1.2.3.4`.

---

### Test 5 — Spoofed XFF, full chain (security check)

```bash
curl -s -H "X-Forwarded-For: 1.2.3.4" http://localhost:8081
```

**Expected — spoofed value is absent:**
```
X-Forwarded-For: 172.20.0.1, 172.20.0.11, 172.20.0.12
```
nginx1 strips the spoofed header at the edge; downstream proxies only append their own IPs.

---

### Test 6 — App is not directly reachable

The `app` container has no exposed port — it is only reachable through the nginx chain:

```bash
curl http://localhost:8084   # no listener → connection refused
```

---

## Stop

```bash
docker compose down
```
