# Installation

This is how i setted-up my homelab server from scratch.  

---

## 1. Install Ubuntu Server

1. Download the latest **Ubuntu Server LTS ISO** (I used `24.04.3 LTS`).  
   👉 https://ubuntu.com/download/server

2. Flash the ISO onto a USB stick (I used [Rufus](https://rufus.ie/) on Windows).

3. Boot the OptiPlex from the USB.  
   In BIOS, set **boot mode = UEFI** and enable **USB boot**.

4. During installation:
   - Hostname: `homelab`
   - User: `arsh`
   - Choose **OpenSSH server** to be installed
   - Skip snaps unless needed


5. After logging in via SSH or locally:
   ```bash
   sudo apt update && sudo apt upgrade -y
   sudo reboot
   ```

---

## 2. Install Tools

These are the main tools I installed on my homela.  


### 1. Cockpit

**Purpose**:  
Cockpit is a web-based system manager for Linux. It gives me a dashboard to monitor resources, manage users, update packages, and troubleshoot without needing to SSH for every small task.

**Install**:

```bash
sudo apt install cockpit -y
sudo systemctl enable --now cockpit
```

**Access**:

* LAN: `https://192.168.1.50:9090`
* Subdomain: `cockpit.itsarsh.dev` (via NPM + Cloudflare)

---

### 2. Portainer CE

**Purpose**:
Portainer is a UI to manage Docker. I use it to deploy, stop, and manage containers without memorizing long `docker run` commands.

**Install**:

```bash
docker volume create portainer_data
docker run -d \
  -p 9000:9000 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

**Access**:

* LAN: `https://192.168.1.50:9000`
* Subdomain: `portainer.itsarsh.dev`

---

### 3. Nginx Proxy Manager (NPM)

**Purpose**:
NPM acts as a reverse proxy. Instead of remembering ports (`9090`, `9000`, etc.), I can map services to subdomains like:

* `cockpit.itsarsh.dev`
* `portainer.itsarsh.dev`
* `netdata.itsarsh.dev`

It also has inbuilt SSL provider (Let’s Encrypt) to provide SSL certificates automatically.

**Install**:

```bash
docker volume create npm_data
docker volume create npm_letsencrypt
docker run -d \
  -p 80:80 -p 81:81 -p 443:443 \
  --name npm \
  --restart=always \
  -v npm_data:/data \
  -v npm_letsencrypt:/etc/letsencrypt \
  jc21/nginx-proxy-manager:latest
```

**Access (admin panel)**:

* LAN only: `http://192.168.1.50:81`
* Tailscale: `http://<tailscale-ip>:81`

---

### 4. Netdata

**Purpose**:
Netdata is real-time monitoring for CPU, RAM, disk I/O, and network traffic.
It shows second-by-second activity, unlike Cockpit which is more of an overview.

**Install**:

```bash
bash <(curl -Ss https://my-netdata.io/kickstart.sh)
```

**Access**:

* LAN: `http://192.168.1.50:19999`
* Subdomain: `netdata.itsarsh.dev`

---

### 5. Cloudflare Tunnel

**Purpose**:
Cloudflare Tunnel is used to securely expose my services to the internet without opening ports on my home router.
All traffic goes through Cloudflare’s network.

**Install**:

```bash
sudo apt update
sudo apt upgrade -y

sudo apt install curl lsb-release

curl -L https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee /usr/share/keyrings/cloudflare-archive-keyring.gpg >/dev/null

echo "deb [signed-by=/usr/share/keyrings/cloudflare-archive-keyring.gpg] https://pkg.cloudflare.com/cloudflared $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/cloudflared.list

sudo apt update

sudo apt install cloudflared
```

**Setup**:

```bash
cloudflared tunnel login
cloudflared tunnel create homelab
```

**Config example** (`~/.cloudflared/config.yml`):

```yaml
tunnel: <tunnel-id>
credentials-file: /home/arsh/.cloudflared/<tunnel-id>.json

ingress:
  - hostname: *.itsarsh.dev
    service: http://localhost:80
    originRequest:
      noTLSVerify: true
  - service: http_status:404
```

---

### 6. Firewall Rules

UFW is configured to restrict access based on **network type**:

Here are the exact UFW rules used:

```bash
sudo ufw allow in on tailscale0 to any port 22 proto tcp    # SSH
sudo ufw allow in on tailscale0 to any port 81 proto tcp    # NPM
sudo ufw allow in on tailscale0 to any port 19999 proto tcp # Netdata
sudo ufw allow in on tailscale0 to any port 9000 proto tcp  # Portainer
```

This is what final firewall rules looks like:

```bash
[ 1] 22 on tailscale0           ALLOW IN    Anywhere       # SSH over Tailscale
[ 2] 80,443/tcp                 ALLOW IN    Anywhere       # Public web traffic (via NPM/Cloudflare)
[ 3] 81/tcp                     ALLOW IN    192.168.1.0/24 # NPM admin only on LAN
[ 4] 81/tcp on tailscale0       ALLOW IN    Anywhere       # NPM admin via Tailscale
[ 5] Anywhere on docker0        ALLOW IN    Anywhere       # Containers network
[ 6] 22 (v6) on tailscale0      ALLOW IN    Anywhere (v6)
[ 7] 80,443/tcp (v6)            ALLOW IN    Anywhere (v6)
[ 8] 81/tcp (v6) on tailscale0  ALLOW IN    Anywhere (v6)
[ 9] Anywhere (v6) on docker0   ALLOW IN    Anywhere (v6)
```

#### Access Policy:

* **Public Internet**: only ports `80` and `443` (websites via NPM/Cloudflare).
* **LAN**: admin panel on port `81`, Cockpit (`9090`), Netdata (`19999`), Portainer (`9443`).
* **Tailscale**: full access for remote management, including SSH.

---

### 7. SMART Monitoring Tools

**Purpose**:
I use it to check the health of drives and receive alerts if a disk is failing.

**Install**:

```bash
sudo apt install smartmontools -y
sudo systemctl enable --now smartmontools
```

**Check a disk**:

```bash
sudo smartctl -a /dev/sda
```

---

### 8. Tailscale

**Purpose**:
Its a Zero-config VPN. I use it to access my homelab securely from anywhere without relying on Cloudflare for internal tools.

**Install**:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

**Usage**:
Now I can access my homelab using its **Tailscale IP** from my laptop or phone.
Example: `https://100.x.y.z:9090` → Cockpit, even if Cloudflare Tunnel is down.