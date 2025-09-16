# DNS & Cloudflare

## Overview

This section explains how DNS works, why Cloudflare is used in this homelab, and how you can configure your own domain (like `itsarsh.dev`) to point to your local server securely using Cloudflare Tunnel.

---

## What is DNS?

DNS (Domain Name System) is like the internet's phone book. It translates human-readable domains (like `itsarsh.dev`) into IP addresses (like `192.168.1.50` or a public IP).

### Key DNS Terms:

* **A Record**: Maps domain to an IPv4 address.
* **AAAA Record**: Maps domain to an IPv6 address.
* **CNAME**: Alias for another domain.
* **MX**: Mail exchange records.

---

## Why Use Cloudflare in a Homelab?

Cloudflare provides:

* Free DNS hosting
* Built-in CDN and DDoS protection
* Secure reverse proxy with **Cloudflare Tunnel** (bypasses NAT and firewall)
* HTTPS with automatic SSL certificates

You don't need to expose your home IP or open ports. Instead, your homelab connects **outbound** to Cloudflare's servers.

---

## How Cloudflare Tunnel Works

Instead of exposing homelab directly to the internet, you can create a tunnel from your server to Cloudflare using the `cloudflared` daemon. DNS for your subdomains points to the tunnel, and requests get routed securely.

### Example:

* `portainer.itsarsh.dev` → Cloudflare Tunnel → `localhost:9000`
* `netdata.itsarsh.dev` → Tunnel → `localhost:19999`

---

## Setting Up DNS in Cloudflare

1. Register a domain (`itsarsh.dev`)
2. Add domain to Cloudflare and change nameservers as instructed.
3. Go to the DNS tab and create these entries:

| Type  | Name          | Target                                   | Proxy Status |
| ----- | ------------- | ---------------------------------------- | ------------ |
| CNAME | `portainer`   | `tunnel-id.cfargotunnel.com`             | Proxied      |
| CNAME | `netdata`     | `tunnel-id.cfargotunnel.com`             | Proxied      |

---

## Setting Up Cloudflare Tunnel

  1. On your server, install `cloudflared`:

    ```bash
    wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
    sudo dpkg -i cloudflared-linux-amd64.deb
    ```

  2. Authenticate:

    ```bash
    cloudflared tunnel login
    ```

  3. Create a tunnel:

    ```bash
    cloudflared tunnel create homelab-tunnel
    ```

  4. Add route:

    ```bash
    cloudflared tunnel route dns homelab-tunnel portainer.itsarsh.dev
    ```

  5. Create a config:
  
```yml
  #/etc/cloudflared/config.yml
  tunnel: homelab-tunnel
  credentials-file: /root/.cloudflared/homelab-tunnel.json
  ingress:
    - hostname: portainer.itsarsh.dev
      service: http://localhost:9000
    - hostname: netdata.itsarsh.dev
      service: http://localhost:19999
    - service: http_status:404
```

  6. Enable and run:

    ```bash
    sudo systemctl enable cloudflared
    sudo systemctl start cloudflared
    ```


---

## Important Notes

* Make sure Cloudflare proxy is **enabled** (orange cloud icon).
* You can add as many subdomains as you want via the tunnel.
* Avoid pointing the apex domain to internal services unless needed.
* Use wildcard `*.itsarsh.dev` with care (for catch-all routing).

---