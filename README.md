𝔸 ℂ𝕠𝕠𝕝 𝕎𝕒𝕪 𝕋𝕠 𝕊𝕖𝕥𝕦𝕡 𝕒 | 𝕆𝕡𝕖𝕟 𝕍𝕡𝕟 𝕊𝕖𝕣𝕧𝕖𝕣 𝔽𝕠𝕣 𝕚𝕣𝕒𝕟☘️
***

# 🛡️ OpenVPN over Cloudflare Tunnel

> **A censorship-resistant, secure OpenVPN setup using Google Cloud and Cloudflare.**

This project provides a step-by-step guide to deploying a private OpenVPN server on **Google Cloud** and proxying it through **Cloudflare Tunnel**. By routing traffic over **Port 443 (HTTPS)**, this setup mimics standard web traffic, making it highly effective at bypassing firewalls and deep packet inspection (DPI).

![Ubuntu](https://img.shields.io/badge/OS-Ubuntu%2020.04%2F22.04-blue)
![OpenVPN](https://img.shields.io/badge/VPN-OpenVPN-green)
![Cloudflare](https://img.shields.io/badge/Proxy-Cloudflare%20Tunnel-orange)
![License](https://img.shields.io/badge/License-MIT-purple)

## ✨ Features

*   **🕵️ Stealth:** Runs on Port 443 (HTTPS), blending in with normal web traffic.
*   **🔒 Secure:** Uses Cloudflare Tunnel (`cloudflared`) to hide your server's IP address entirely.
*   **🚀 Fast:** Leverages Google Cloud’s global infrastructure and Cloudflare’s edge network.
*   **🛡️ Anti-Censorship:** Bypasses standard port blocking and DPI filters.
*   **📱 Multi-Platform:** Works on Linux, Windows, macOS, Android, and iOS.

## 📋 Prerequisites

Before starting, ensure you have:

1.  **A Google Cloud Account** with a project created.
2.  **A Domain Name** (Any registrar works: Namecheap, GoDaddy, NIC.ir, etc.).
3.  **Root Access** to your Google Cloud VM (SSH).
4.  **Basic Linux Knowledge** (Terminal usage).

## 🚀 Quick Start

### Step 1: Provision the Google Cloud VM

1.  Go to the [Google Cloud Console](https://console.cloud.google.com/).
2.  Create a new **Compute Engine Instance**.
    *   **OS:** Ubuntu 22.04 LTS (Recommended).
    *   **Machine Type:** `e2-micro` (Free Tier eligible) or `e2-small` for better performance.
3.  **Firewall Rules:** Ensure your VPC Network allows inbound traffic on **TCP 80** and **TCP 443**.

### Step 2: Install OpenVPN

SSH into your VM and run the following commands:

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install OpenVPN and EasyRSA
sudo apt install openvpn easy-rsa -y
```

### Step 3: Configure OpenVPN

Create the server configuration file:

```bash
sudo nano /etc/openvpn/server.conf
```

Paste the following configuration. **Key Change:** We use `port 443` and `proto tcp` to mimic HTTPS.

```conf
port 443
proto tcp
dev tun
ca ca.crt
cert server.crt
key server.key
dh dh.pem
auth SHA256
tls-auth ta.key 0
topology subnet
server 10.8.0.0 255.255.255.0
ifconfig-pool-persist ipp.txt

# Push DNS settings (Cloudflare DNS)
push "redirect-gateway def1 bypass-dhcp"
push "dhcp-option DNS 208.67.222.222"
push "dhcp-option DNS 208.67.220.220"

keepalive 10 120
tls-version-min 1.2
tls-cipher TLS-DHE-RSA-WITH-AES-256-GCM-SHA384
compression gzip

# Security
user nobody
group nogroup
persist-key
persist-tun
status openvpn-status.log
verb 3
explicit-exit-notify 1
```

### Step 4: Enable IP Forwarding & NAT

To allow internet access through the VPN, configure NAT:

```bash
# Enable IP Forwarding
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# Configure iptables
sudo iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE
sudo iptables -A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A INPUT -i tun+ -j ACCEPT
sudo iptables -A FORWARD -i tun+ -j ACCEPT

# Persist rules
sudo apt install iptables-persistent -y
sudo netfilter-persistent save
```

### Step 5: Start OpenVPN

```bash
sudo systemctl enable openvpn@server
sudo systemctl start openvpn@server
sudo systemctl status openvpn@server
```

---

## ☁️ Step 6: Set Up Cloudflare Tunnel

This step hides your Google Cloud IP and allows connection via your domain.

### 1. Install `cloudflared`

```bash
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
sudo dpkg -i cloudflared-linux-amd64.deb
```

### 2. Create the Tunnel

```bash
cloudflared tunnel create my-openvpn-tunnel
```
*Copy the **Tunnel ID** and the path to the **credentials file** generated.*

### 3. Configure Routing

Create/edit the config file:
```bash
nano ~/.cloudflared/config.yml
```

Add the following (replace `<YOUR-TUNNEL-ID>` and your username):

```yaml
tunnel: <YOUR-TUNNEL-ID>
credentials-file: /home/your_username/.cloudflared/<YOUR-TUNNEL-ID>.json

ingress:
  - hostname: vpn.yourdomain.com
    service: tcp://localhost:443
  - service: http_status:404
```

### 4. Run the Tunnel

```bash
cloudflared tunnel run my-openvpn-tunnel
```

### 5. Configure DNS in Cloudflare Dashboard

1.  Log in to [Cloudflare Dashboard](https://dash.cloudflare.com/).
2.  Go to **DNS** > **Records**.
3.  Add a **CNAME** record:
    *   **Name:** `vpn` (creates `vpn.yourdomain.com`)
    *   **Target:** `<YOUR-TUNNEL-ID>.cfargotunnel.com`
    *   **Proxy Status:** Proxied (Orange Cloud icon should be ON)

---

## 💻 Step 7: Client Configuration

Download the `ca.crt`, `client.crt`, and `client.key` files generated during the OpenVPN setup. Create a file named `client.ovpn` on your device with the following content:

```conf
client
dev tun
proto tcp
remote vpn.yourdomain.com 443
resolv-retry infinite
nobind
persist-key
persist-tun
ca ca.crt
cert client.crt
key client.key
remote-cert-tls server
tls-version-min 1.2
tls-cipher TLS-DHE-RSA-WITH-AES-256-GCM-SHA384
verify-x509-name vpn.yourdomain.com name
verb 3
```

**Connect using any OpenVPN client** (OpenVPN Connect, Tunnelblick, OpenVPN for Android, etc.) and import this file.

---

## ⚠️ Troubleshooting & Notes

*   **Port 443 Blocking:** In rare cases, ISPs may block port 443. If this happens, consider using **OpenVPN over SSH** or tools like **Streisand**.
*   **Performance:** Cloudflare adds minimal latency. For best speeds, choose a Google Cloud region close to your location (e.g., `us-central1`, `europe-west1`).
*   **Security:** Never share your `.key` files publicly. Keep `cloudflared` and OpenVPN updated.
*   **SSL Certificates:** You do **not** need a public SSL certificate for the server. Cloudflare handles SSL termination at the edge.
## 📜 License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.

---

<div align="center">

Made ❤️ by **@𝔸𝕪𝕙𝕒𝕟𝕄𝕒𝕟𝕤𝕦𝕣 𝟚𝟘𝟚𝟞☘️** | Powered by **Google Cloud** & **Cloudflare**

</div>
