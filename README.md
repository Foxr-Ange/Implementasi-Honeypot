# Honeypot Bundle — Proxmox + Phishing Gateway
Mentor: Aditya

> Ringkas: VM `phishing-gateway` expose ke internet, forward ke VM `honeypot-core` di DMZ. ELK lokal untuk analitik.

---
## 0) Prasyarat
- Proxmox dengan bridges:
  - vmbr0 (WAN/Public) → untuk `phishing-gateway`
  - vmbr1 (DMZ) → untuk `phishing-gateway` (DMZ iface) & `honeypot-core`
  - vmbr2 (MGMT/Private) → untuk akses admin Kibana/SSH (opsional)
- DNS: `login.example.com` → IP publik `phishing-gateway`.
- VM OS: Ubuntu 22.04 LTS.

---
## 1) Deploy honeypot-core (DMZ)
Login ke VM **honeypot-core** (Ubuntu).

```bash
sudo apt update && sudo apt -y install docker.io docker-compose-plugin ufw
sudo ufw default deny outgoing
sudo ufw default deny incoming
# allow mgmt (ganti 10.10.10.0/24 dengan jaringan admin kamu)
sudo ufw allow from 10.10.10.0/24 to any port 5601 proto tcp
sudo ufw allow from 10.10.10.0/24 to any port 9200 proto tcp
sudo ufw allow from DMZ_GATEWAY_IP to any port 2222,2223,8080 proto tcp
# ICS jika tidak pakai host-mode: map manual. Untuk host-mode, pastikan gateway bisa reach 502/102/2404
sudo ufw enable
```

Salin folder `honeypot-core/` ke VM ini lalu jalankan:
```bash
cd honeypot-core
sudo docker compose up -d
```

Cek:
```bash
docker ps
curl -s localhost:9200 | head
```

---
## 2) Deploy phishing-gateway (WAN+DMZ)
Login ke VM **phishing-gateway**.

### 2.1 Nginx + HAProxy + Filebeat
```bash
sudo apt update && sudo apt -y install nginx haproxy filebeat certbot python3-certbot-nginx
sudo mkdir -p /var/www/phish
sudo cp -r phishing-gateway/nginx/html/* /var/www/phish/
sudo cp phishing-gateway/nginx/phish.conf /etc/nginx/sites-available/phish.conf
sudo ln -sf /etc/nginx/sites-available/phish.conf /etc/nginx/sites-enabled/phish.conf
sudo nginx -t && sudo systemctl reload nginx
```

### 2.2 TLS (Let’s Encrypt)
```bash
# Ganti domain
sudo certbot --nginx -d login.example.com --register-unsafely-without-email --agree-tos -n
```

### 2.3 HAProxy
Edit `phishing-gateway/haproxy.cfg` → ganti `DMZ_HONEYPOT_IP` dengan IP VM honeypot-core di DMZ.
```bash
sudo cp phishing-gateway/haproxy.cfg /etc/haproxy/haproxy.cfg
sudo systemctl restart haproxy
```

### 2.4 Filebeat (gateway → ELK)
Edit `phishing-gateway/filebeat-gateway.yml` → set `ELK_IP` dengan IP Elasticsearch (biasanya IP honeypot-core DMZ/MGMT).
```bash
sudo cp phishing-gateway/filebeat-gateway.yml /etc/filebeat/filebeat.yml
sudo systemctl enable --now filebeat
```

### 2.5 Firewall (gateway)
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 80,443/tcp
sudo ufw allow 22,23,502,102,2404/tcp
sudo ufw enable
```

---
## 3) Kibana & Indeks
Akses Kibana: `http://ELK_IP:5601` dari jaringan MGMT/admin.

Buat Index Pattern:
- `honeypot-*` (Cowrie/Conpot)
- `honeypot-gw-*` (Nginx/HAProxy)

Import/ikuti `kibana/alerts.md` untuk membuat alert.

---
## 4) Uji Fungsional
Dari host luar (internet):
```bash
nmap -sV -p 22,23,80,443,502,102,2404 <public_ip_gateway>
ssh test@<domain>  # harus nyambung ke Cowrie (fake)
curl -I https://login.example.com/
# Modbus
nmap -sV -p 502 <public_ip_gateway>
```

Periksa log real-time (Kibana Discover).

---
## 5) Catatan Keamanan Penting
- **JANGAN** memberi route dari DMZ ke LAN produksi.
- Default deny **egress** di honeypot-core; whitelist hanya ke ELK bila perlu.
- Decoy website **tidak** mengumpulkan kredensial; murni untuk pengujian UI/UX saja.
- Semua penggunaan harus sesuai etika dan peraturan yang berlaku.

---
## 6) Variabel yang harus diganti
- `login.example.com` → domain kamu.
- `DMZ_HONEYPOT_IP` → IP honeypot-core (DMZ).
- `ELK_IP` → IP Elasticsearch (DMZ/MGMT).
- `DMZ_GATEWAY_IP` → IP phishing-gateway pada segmen DMZ.

---
## 7) Struktur
```
honeypot-core/
  docker-compose.yml
  filebeat.yml
  cowrie/cowrie.cfg
  conpot/conpot.cfg
phishing-gateway/
  haproxy.cfg
  nginx/phish.conf
  nginx/html/index.html
  filebeat-gateway.yml
kibana/
  alerts.md
  dashboards.ndjson
```