# 𝔸 𝕊𝕥𝕖𝕒𝕝𝕙 𝕎𝕒𝕪 𝕋𝕠 𝕊𝕖𝕥𝕦𝕡 𝕒 | 𝕆𝕡𝕖𝕟 𝕍𝕡𝕟 𝕊𝕖𝕣𝕧𝕖𝕣 𝕗𝕠𝕣 𝕚𝕣𝕒𝕟☘️
***

# 🛡️ اوپن‌وی‌ان (OpenVPN) روی تانل کلودفلر
> **یک راهکار امن و مقاوم در برابر سانسور با استفاده از گوگل کلاود و کلودفلر.**

این پروژه یک راهنمای گام‌به‌گام برای استقرار یک سرور اوپن‌وی‌ان (OpenVPN) خصوصی بر روی **Google Cloud** و پروکسی کردن آن از طریق **Cloudflare Tunnel** ارائه می‌دهد. با مسیریابی ترافیک از طریق **پورت ۴۴۳ (HTTPS)**، این پیکربندی ترافیک را کاملاً شبیه به وب‌گردی معمولی نشان می‌دهد که آن را بسیار موثر در دور زدن فایروال‌ها و بازرسی عمیق بسته‌ها (DPI) می‌کند.

![Ubuntu](https://img.shields.io/badge/OS-Ubuntu%2020.04%2F22.04-blue)
![OpenVPN](https://img.shields.io/badge/VPN-OpenVPN-green)
![Cloudflare](https://img.shields.io/badge/Proxy-Cloudflare%20Tunnel-orange)
![License](https://img.shields.io/badge/License-MIT-purple)

## ✨ ویژگی‌ها
*   **🕵️ استلث (مخفی):** اجرا روی پورت ۴۴۳ (HTTPS)، ترکیب شدن با ترافیک عادی وب.
*   **🔒 امن:** استفاده از تانل کلودفلر (`cloudflared`) برای پنهان‌سازی کامل آدرس IP سرور.
*   **🚀 سریع:** بهره‌گیری از زیرساخت جهانی گوگل کلاود و شبکه لبه‌ای (Edge) کلودفلر.
*   **🛡️ ضد سانسور:** دور زدن مسدودسازی پورت‌های استاندارد و فیلترهای DPI.
*   **📱 چند پلتفرمی:** سازگار با لینوکس، ویندوز، مک‌او‌اس، اندروید و iOS.

## 📋 پیش‌نیازها
قبل از شروع، مطمئن شوید که موارد زیر را دارید:
1.  **حساب گوگل کلاود** با یک پروژه ایجاد شده.
2.  **یک نام دامنه** (هر رجیستراری کار می‌کند: Namecheap، GoDaddy، NIC.ir و غیره).
3.  **دسترسی روت (Root)** به ماشین مجازی گوگل کلاود شما (SSH).
4.  **دانش پایه لینوکس** (کار با ترمینال).

## 🚀 شروع سریع

### مرحله ۱: راه‌اندازی ماشین مجازی گوگل کلاود
1.  به [کنسول گوگل کلاود](https://console.cloud.google.com/) بروید.
2.  یک **Compute Engine Instance** جدید ایجاد کنید.
    *   **سیستم عامل:** Ubuntu 22.04 LTS (پیشنهاد می‌شود).
    *   **نوع ماشین:** `e2-micro` (شامل طرح رایگان) یا `e2-small` برای عملکرد بهتر.
3.  **قوانین فایروال:** مطمئن شوید که شبکه VPC شما اجازه ترافیک ورودی روی **TCP 80** و **TCP 443** را می‌دهد.

### مرحله ۲: نصب اوپن‌وی‌ان
وارد SSH سرور خود شوید و دستورات زیر را اجرا کنید:

```bash
# به‌روزرسانی سیستم
sudo apt update && sudo apt upgrade -y

# نصب OpenVPN و EasyRSA
sudo apt install openvpn easy-rsa -y
```

### مرحله ۳: پیکربندی اوپن‌وی‌ان
فایل پیکربندی سرور را ایجاد کنید:

```bash
sudo nano /etc/openvpn/server.conf
```

پیکربندی زیر را در آن جایگذاری کنید. **تغییر کلیدی:** ما از `port 443` و `proto tcp` استفاده می‌کنیم تا ترافیک شبیه HTTPS باشد.

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

# تنظیمات DNS (Cloudflare DNS)
push "redirect-gateway def1 bypass-dhcp"
push "dhcp-option DNS 208.67.222.222"
push "dhcp-option DNS 208.67.220.220"

keepalive 10 120
tls-version-min 1.2
tls-cipher TLS-DHE-RSA-WITH-AES-256-GCM-SHA384
compression gzip

# امنیت
user nobody
group nogroup
persist-key
persist-tun
status openvpn-status.log
verb 3
explicit-exit-notify 1
```

### مرحله ۴: فعال‌سازی IP Forwarding و NAT
برای اجازه دسترسی به اینترنت از طریق VPN، باید NAT را پیکربندی کنید:

```bash
# فعال‌سازی IP Forwarding
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# پیکربندی iptables
sudo iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE
sudo iptables -A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A INPUT -i tun+ -j ACCEPT
sudo iptables -A FORWARD -i tun+ -j ACCEPT

# ذخیره قوانین برای پایدار ماندن پس از ریستارت
sudo apt install iptables-persistent -y
sudo netfilter-persistent save
```

### مرحله ۵: شروع اوپن‌وی‌ان

```bash
sudo systemctl enable openvpn@server
sudo systemctl start openvpn@server
sudo systemctl status openvpn@server
```

---

## ☁️ مرحله ۶: راه‌اندازی تانل کلودفلر
این مرحله آدرس IP گوگل کلاود شما را مخفی کرده و امکان اتصال از طریق دامنه شما را فراهم می‌کند.

### ۱. نصب `cloudflared`

```bash
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
sudo dpkg -i cloudflared-linux-amd64.deb
```

### ۲. ایجاد تانل

```bash
cloudflared tunnel create my-openvpn-tunnel
```
*آدرس **Tunnel ID** و مسیر فایل **credentials** ایجاد شده را کپی کنید.*

### ۳. پیکربندی Routing
فایل پیکربندی را ایجاد یا ویرایش کنید:

```bash
nano ~/.cloudflared/config.yml
```

مقادیر زیر را اضافه کنید (نام کاربری و `<YOUR-TUNNEL-ID>` را جایگزین کنید):

```yaml
tunnel: <YOUR-TUNNEL-ID>
credentials-file: /home/your_username/.cloudflared/<YOUR-TUNNEL-ID>.json
ingress:
  - hostname: vpn.yourdomain.com
    service: tcp://localhost:443
  - service: http_status:404
```

### ۴. اجرای تانل

```bash
cloudflared tunnel run my-openvpn-tunnel
```

### ۵. پیکربندی DNS در داشبورد کلودفلر
1.  وارد [داشبورد کلودفلر](https://dash.cloudflare.com/) شوید.
2.  به بخش **DNS** > **Records** بروید.
3.  یک رکورد **CNAME** اضافه کنید:
    *   **Name:** `vpn` (این دامنه `vpn.yourdomain.com` را ایجاد می‌کند)
    *   **Target:** `<YOUR-TUNNEL-ID>.cfargotunnel.com`
    *   **Proxy Status:** Proxied (آیکون ابر باید نارنجی/روشن باشد)

---

## 💻 مرحله ۷: پیکربندی کلاینت (کاربر)
فایل‌های `ca.crt`، `client.crt` و `client.key` که در مرحله نصب اوپن‌وی‌ان ایجاد شده‌اند را دانلود کنید. یک فایل با نام `client.ovpn` روی دستگاه خود ایجاد کنید و محتوای زیر را در آن قرار دهید:

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

**با استفاده از هر کلاینت اوپن‌وی‌ان** (OpenVPN Connect، Tunnelblick، OpenVPN for Android و غیره) به این فایل متصل شوید.

---

## ⚠️ عیب‌یابی و نکات مهم
*   **مسدودسازی پورت ۴۴۳:** در موارد نادر، ISPها ممکن است پورت ۴۴۳ را مسدود کنند. در این صورت، از **OpenVPN over SSH** یا ابزارهایی مانند **Streisand** استفاده کنید.
*   **عملکرد:** کلودفلر تأخیر کمی اضافه می‌کند. برای بهترین سرعت، منطقه‌ای از گوگل کلاود را انتخاب کنید که به موقعیت مکانی شما نزدیک‌تر است (مثلاً `us-central1` برای آمریکا، `europe-west1` برای اروپا).
*   **امنیت:** فایل‌های `.key` خود را هرگز به صورت عمومی به اشتراک نگذارید. `cloudflared` و OpenVPN را به‌روز نگه دارید.
*   **گواهی SSL:** شما به گواهی SSL عمومی برای سرور نیاز **ندارید**. کلودفلر پایان‌بخشی SSL را در لبه شبکه انجام می‌دهد.

## 📜 مجوز
این پروژه تحت مجوز MIT منتشر شده است. برای جزئیات بیشتر به فایل [LICENSE](LICENSE) مراجعه کنید.

---
<div align="center">
ساخته شده با ❤️ توسط **@𝔸𝕪𝕙𝕒𝕟𝕄𝕒𝕟𝕤𝕦𝕣 𝟚𝟘𝟚𝟞☘️** | قدرتمند شده توسط **Google Cloud** و **Cloudflare**
</div>
