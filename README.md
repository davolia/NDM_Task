# nginx XFF test stand

Three nginx reverse proxies in front of a simple app. Each proxy is also exposed
directly on the host so you can test any combination of hops.

```
port 8081  ->  nginx1 (172.20.0.11)  ->  nginx2  ->  nginx3  ->  app
port 8082  ->  nginx2 (172.20.0.12)  ->  nginx3  ->  app
port 8083  ->  nginx3 (172.20.0.13)  ->  app
```

The app (`traefik/whoami`) has no exposed port, only reachable via the proxies.

## How XFF works here

All three nginx instances load `shared/geo_trusted.conf`. It uses the `geo` module
to check whether the request came from one of the internal proxy IPs (172.20.0.11-13).

If the source is a trusted proxy, it appends `$remote_addr` to the existing
`X-Forwarded-For`. If it's an outside client, it resets XFF to just `$remote_addr`,
discarding whatever the client sent. This is how spoofed headers get dropped.

## Start

```bash
docker compose up -d
```

## Test protocol

> Note: `172.20.0.1` in the output is the Docker bridge gateway — that's the IP
> nginx sees when you hit it via the exposed port from the host.

**1. Single hop — curl directly to nginx3**

```bash
curl -s http://localhost:8083 | grep X-Forwarded
```

Expected:
```
X-Forwarded-For: 172.20.0.1
```

---

**2. Two hops — nginx2 -> nginx3 -> app**

```bash
curl -s http://localhost:8082 | grep X-Forwarded
```

Expected:
```
X-Forwarded-For: 172.20.0.1, 172.20.0.12
```

---

**3. Full chain — nginx1 -> nginx2 -> nginx3 -> app**

```bash
curl -s http://localhost:8081 | grep X-Forwarded
```

Expected:
```
X-Forwarded-For: 172.20.0.1, 172.20.0.11, 172.20.0.12
```

---

**4. Spoofed XFF, single hop**

```bash
curl -s -H "X-Forwarded-For: 1.2.3.4" http://localhost:8083 | grep X-Forwarded
```

Expected — `1.2.3.4` must not appear:
```
X-Forwarded-For: 172.20.0.1
```

---

**5. Spoofed XFF, full chain**

```bash
curl -s -H "X-Forwarded-For: 1.2.3.4" http://localhost:8081 | grep X-Forwarded
```

Expected — `1.2.3.4` must not appear:
```
X-Forwarded-For: 172.20.0.1, 172.20.0.11, 172.20.0.12
```

---

**6. App not directly reachable**

```bash
curl http://localhost:8084
# should get: connection refused
```

## Stop

```bash
docker compose down
```
