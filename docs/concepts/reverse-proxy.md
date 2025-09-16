# Reverse Proxy

A **reverse proxy** is a server that sits in front of one or more web servers and forwards client requests to the appropriate backend server. Unlike a **forward proxy**, which proxies on behalf of clients, a reverse proxy acts on behalf of servers.

![Placeholder: Diagram explaining reverse proxy](../images/reverse-proxy-diagram.png)

## Why Use a Reverse Proxy?

Reverse proxies are common in web hosting and homelab setups for several reasons:

### 1. Centralized Routing

One domain (e.g. `itsarsh.dev`) can serve multiple services:

* `app1.itsarsh.dev` → Docker container 1
* `app2.itsarsh.dev` → Docker container 2

### 2. SSL Termination

The reverse proxy handles HTTPS certificates (e.g., Let's Encrypt), so individual backend apps don't need to manage TLS.

### 3. Security

The proxy can restrict access:

* Only expose apps via Tailscale IP
* Require authentication before forwarding
* Filter or rate-limit incoming requests

### 4. Logging & Monitoring

Track requests in a centralized location for all services.

### 5. Port Management

All apps can run on internal ports (e.g., 3000, 5000, 8000), while the reverse proxy listens on public ports (80/443).

---

## My Setup: Nginx Proxy Manager (NPM)

In my homelab, I:

* Use **NPM** as a reverse proxy
* It's running on **port 80** in **host network mode**
* Use **Cloudflare Tunnels** to expose it to the public via `*.itsarsh.dev`

```txt
localhost:81         -> NPM admin
app.itsarsh.dev       -> App running on LAN IP
netdata.itsarsh.dev   -> Netdata dashboard
```

Your firewall allows ports selectively:

* `80/443` open to internet
* `81` only allowed on `192.168.1.0/24` and Tailscale interface

---

## How it Works (Step-by-Step)

1. User types `app.itsarsh.dev` in browser
2. DNS resolves via Cloudflare Tunnel
3. Tunnel forwards HTTPS traffic to NPM
4. NPM routes it to internal service (e.g., `192.168.1.50:3000`)

