# 🌐 MikroTik CLI Cheatsheet - Persiapan Tes Junior NOC

![MikroTik](https://img.shields.io/badge/MikroTik-RouterOS-blue?style=for-the-badge&logo=mikrotik)
![Role](https://img.shields.io/badge/Role-Junior_NOC-success?style=for-the-badge)
![Company](https://img.shields.io/badge/Company-GMDP-orange?style=for-the-badge)
![Platform](https://img.shields.io/badge/Platform-PnetLab-purple?style=for-the-badge)

> **PT Global Media Data Prima (GMDP) - PnetLab Test**  
> *Dokumen ini berisi rangkuman perintah MikroTik CLI (RouterOS) yang paling sering muncul dan diuji dalam praktik Junior Network Operation Center (NOC).*

---

## 📌 Daftar Isi
- [1. Konfigurasi Dasar & Identitas](#1-konfigurasi-dasar--identitas)
- [2. Routing & Akses Internet (NAT)](#2-routing--akses-internet-nat)
- [3. DHCP Client & Server](#3-dhcp-client--server)
- [4. PPPoE Client (Koneksi ke ISP)](#4-pppoe-client-koneksi-ke-isp)
- [5. PPPoE Server (Melayani Pelanggan)](#5-pppoe-server-melayani-pelanggan)
- [6. VLAN & Bridge](#6-vlan--bridge)
- [7. Firewall Dasar (Keamanan)](#7-firewall-dasar-keamanan)
- [8. Bandwidth Management (Queue)](#8-bandwidth-management-queue)
- [9. Failover / Netwatch (Dua ISP)](#9-failover--netwatch-dua-isp)
- [10. Tools Real-time Monitoring & Troubleshooting](#10-tools-real-time-monitoring--troubleshooting)
- [11. Backup & Restore](#11-backup--restore)

---

## 1. Konfigurasi Dasar & Identitas
*⚙️ Perintah untuk memulai dan memastikan perangkat dikenali.*

```bash
# ==========================================
# 1. Konfigurasi Dasar & Identitas
# ==========================================
# Cek versi RouterOS
/system resource print

# Ubah nama router
/system identity set name=NOC-Router-01

# Cek daftar interface
/interface print

# Beri nama interface agar rapi (misal ether1 jadi WAN, ether2 jadi LAN)
/interface set ether1 name=WAN
/interface set ether2 name=LAN

# Beri IP Address
/ip address add address=192.168.10.1/24 interface=LAN
/ip address add address=10.10.10.2/30 interface=WAN
```

---

## 2. Routing & Akses Internet (NAT)
*🌐 Agar router bisa terhubung ke internet dan membagikan koneksi ke client.*

```bash
# ==========================================
# 2. Routing & Akses Internet (NAT)
# ==========================================
# Cek tabel routing
/ip route print

# Tambah Default Route (Gateway ke Internet)
/ip route add dst-address=0.0.0.0/0 gateway=10.10.10.1

# Konfigurasi NAT (Masquerade) agar client LAN bisa internet
/ip firewall nat add chain=srcnat out-interface=WAN action=masquerade

# Konfigurasi DNS (Izinkan client pakai router sebagai DNS)
/ip dns set servers=8.8.8.8,1.1.1.1 allow-remote-requests=yes
```

---

## 3. DHCP Client & Server
*📡 Membagikan IP otomatis ke client.*

```bash
# ==========================================
# 3. DHCP Client & Server
# ==========================================
# DHCP Client (untuk WAN jika dari ISP-nya DHCP)
/ip dhcp-client add interface=WAN disabled=no

# DHCP Server (untuk LAN)
# 1. Buat IP Pool
/ip pool add name=pool-lan ranges=192.168.10.10-192.168.10.254

# 2. Buat DHCP Server
/ip dhcp-server add interface=LAN address-pool=pool-lan disabled=no name=dhcp-lan

# 3. Buat DHCP Network (set gateway & DNS)
/ip dhcp-server network add address=192.168.10.0/24 gateway=192.168.10.1 dns-server=192.168.10.1
```

---

## 4. PPPoE Client (Koneksi ke ISP)
*🔌 Koneksi utama dari ISP biasanya menggunakan PPPoE, bukan DHCP.*

```bash
# ==========================================
# 4. PPPoE Client (Koneksi ke ISP)
# ==========================================
# Buat PPPoE Client di interface WAN
/interface pppoe-client add name=pppoe-out1 interface=WAN user=user_isp password=pass_isp disabled=no add-default-route=yes use-peer-dns=yes
```

---

## 5. PPPoE Server (Melayani Pelanggan)
*🧑‍💻 Konfigurasi untuk membagikan koneksi ke pelanggan via PPPoE.*

```bash
# ==========================================
# 5. PPPoE Server (Melayani Pelanggan)
# ==========================================
# 1. Buat IP Pool untuk PPPoE
/ip pool add name=pool-pppoe ranges=10.10.10.10-10.10.10.100

# 2. Buat Profile PPPoE (set DNS dan IP lokal router)
/ppp profile add name=profile-isp local-address=10.10.10.1 remote-address=pool-pppoe dns-server=8.8.8.8

# 3. Aktifkan PPPoE Server di interface LAN
/interface pppoe-server server add service-name=service-isp interface=LAN default-profile=profile-isp disabled=no

# 4. Buat Secret (User & Password Pelanggan)
/ppp secret add name=pelanggan1 password=rahasia service=pppoe profile=profile-isp
```

---

## 6. VLAN & Bridge
*🏷️ Memisahkan traffic jaringan (misal: VLAN 10 untuk Internet, VLAN 20 untuk IPTV).*

```bash
# ==========================================
# 6. VLAN & Bridge
# ==========================================
# Buat interface VLAN di ether yang mengarah ke Switch/Client
/interface vlan add name=vlan10 interface=LAN vlan-id=10
/interface vlan add name=vlan20 interface=LAN vlan-id=20

# Beri IP di VLAN tersebut
/ip address add address=192.168.10.1/24 interface=vlan10

# Setup Bridge (Jika menggunakan Bridge VLAN Filtering - Level lanjut)
/interface bridge add name=bridge-lan
/interface bridge port add bridge=bridge-lan interface=LAN
```

---

## 7. Firewall Dasar (Keamanan)
*🛡️ Mencegah router diakses sembarangan dari jaringan luar (WAN).*

```bash
# ==========================================
# 7. Firewall Dasar (Keamanan)
# ==========================================
# Izinkan koneksi yang sudah established/related (Wajib di paling atas)
/ip firewall filter add chain=input action=accept connection-state=established,related

# Blokir akses dari WAN (misal drop semua dari ether1/WAN)
/ip firewall filter add chain=input action=drop in-interface=WAN

# Cek list firewall
/ip firewall filter print
```

---

## 8. Bandwidth Management (Queue)
*⏱️ Membatasi kecepatan internet client (sering diuji di praktik NOC).*

```bash
# ==========================================
# 8. Bandwidth Management (Simple Queue)
# ==========================================
# Batasi bandwidth IP client (misal client dengan IP 192.168.10.10 dibatasi 5Mbps)
/queue simple add name="Limit-Pelanggan" target=192.168.10.10/32 max-limit=5M/5M
```

---

## 9. Failover / Netwatch (Dua ISP)
*🔄 Skenario: Kalau ISP 1 mati, otomatis pindah ke ISP 2.*

```bash
# ==========================================
# 9. Failover / Netwatch (Dua ISP)
# ==========================================
# Konfigurasi Dasar: Tambah 2 default route dengan distance berbeda
# (Distance lebih kecil = prioritas utama)
/ip route add dst-address=0.0.0.0/0 gateway=ISP1-IP distance=1
/ip route add dst-address=0.0.0.0/0 gateway=ISP2-IP distance=2

# Gunakan Netwatch untuk memantau IP Gateway ISP 1
/tool netwatch add host=ISP1-IP interval=10s \
down-script="/ip route disable [find gateway=ISP1-IP]" \
up-script="/ip route enable [find gateway=ISP1-IP]"
```

---

## 10. Tools Real-time Monitoring & Troubleshooting
*🔍 Saat diberi skenario "Client tidak bisa internetan" atau diminta pantau jaringan.*

```bash
# ==========================================
# 10. Tools Real-time Monitoring & Troubleshooting
# ==========================================
# Cek koneksi ke luar dari router
/ping 8.8.8.8
/ping google.com

# Cek jalur routing (apakah mentok di mana)
/tool traceroute 8.8.8.8

# Cek apakah NAT berjalan (lihat counter packets/bytes-nya nambah atau tidak)
/ip firewall nat print stats

# Cek log sistem (mencari pesan error atau link down)
/log print

# Cek status interface (apakah running atau link down)
/interface print stats

# Melihat traffic secara real-time di interface tertentu (seperti melihat siapa yang download besar)
/tool torch interface=WAN

# Cek koneksi aktif (siapa saja yang sedang terhubung ke router)
/ip firewall connection print

# Melihat profil penggunaan CPU dan memori router
/system resource monitor
```

---

## 11. Backup & Restore
*💾 Jangan pernah lupa backup konfigurasi jika kamu sudah selesai mengerjakan soal.*

```bash
# ==========================================
# 11. Backup & Restore (Wajib sebelum selesai tes!)
# ==========================================
# Backup file binary (untuk restore di versi RouterOS yang sama)
/system backup save name=hasil-tes-noc

# Backup file text/RSC (bisa dibaca di notepad dan di-import)
/export file=hasil-tes-noc
```

---

> [!TIP]
> **💡 Tips Tambahan Saat Tes PnetLab:**
> 1. **Gunakan Tab Completion**: Ketik awal perintah lalu tekan `Tab`, MikroTik akan otomatis melengkapi. (Misal: ketik `/ip fi` lalu `Tab`, akan jadi `/ip firewall`).
> 2. **Tanda Tanya (`?`)**: Jika lupa parameter, ketik `?` untuk melihat bantuan. Contoh: `/ip address add ?`
> 3. **Hati-hati dengan Konfigurasi**: Karena di PnetLab, jika kamu salah setting routing atau firewall dan terputus, kamu mungkin harus reset lab atau masuk lewat console. Selalu cek `ping` setelah melakukan perubahan.

> [!WARNING]
> **📸 Screenshot/Simpan Konfigurasi:**  
> Biasanya instruktur akan meminta bukti screenshot atau file hasil export (`/export`). Pastikan kamu tahu cara menyalin teks dari terminal PnetLab ke notepad.
