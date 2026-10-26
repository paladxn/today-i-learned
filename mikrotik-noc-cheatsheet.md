# 🌐 MikroTik CLI Masterclass: Tes Skill Junior NOC (GMDP)

![MikroTik](https://img.shields.io/badge/MikroTik-RouterOS_v6_%2F_v7-blue?style=for-the-badge&logo=mikrotik)
![Role](https://img.shields.io/badge/Target-Junior_NOC-success?style=for-the-badge)
![Test](https://img.shields.io/badge/Date-27_Oct_2026-red?style=for-the-badge)
![Platform](https://img.shields.io/badge/Lab-PNETLab-orange?style=for-the-badge)

> **Tes Praktik:** Selasa, 27 Oktober 2026, 11.00 WIB via Zoom & PNETLab. 
> *Dokumen ini dirancang bukan hanya sebagai "contekan perintah", tetapi sebagai panduan berpikir (mental model) saat mengonfigurasi jaringan di bawah tekanan waktu.*

---

## 📌 Daftar Isi
- [0. Strategi Emas Saat Tes](#0-strategi-emas-saat-tes)
- [1. Konfigurasi Dasar & Keamanan Awal](#1-konfigurasi-dasar--keamanan-awal)
- [2. Interface, IP & Bridge Dasar](#2-interface-ip--bridge-dasar)
- [3. Routing, DNS & NAT (Jantung Internet)](#3-routing-dns--nat-jantung-internet)
- [4. DHCP Server & Client](#4-dhcp-server--client)
- [5. PPPoE Client & Server (Khas ISP!)](#5-pppoe-client--server-khas-isp)
- [6. VLAN & Bridge VLAN Filtering (Sering Diuji!)](#6-vlan--bridge-vlan-filtering-sering-diuji)
- [7. Routing Dinamis: OSPF (v6 vs v7)](#7-routing-dinamis-ospf-v6-vs-v7)
- [8. Firewall: Proteksi Router & Client](#8-firewall-proteksi-router--client)
- [9. Bandwidth Management (Queue)](#9-bandwidth-management-queue)
- [10. Failover Dua ISP](#10-failover-dua-isp)
- [11. Layanan Pendukung NOC (NTP, SNMP, Log)](#11-layanan-pendukung-noc-ntp-snmp-log)
- [12. Monitoring & Troubleshooting (Kunci NOC)](#12-monitoring--troubleshooting-kunci-noc)
- [13. Backup & Export (Wajib!)](#13-backup--export-wajib)
- [14. Contoh Lab Lengkap (4 Router)](#14-contoh-lab-lengkap-4-router)

---

## 0. Strategi Emas Saat Tes

> [!IMPORTANT]
> **Alur Berpikir OSI Layer (Bawah ke Atas):**
> Jangan pernah mengetik perintah acak. Ikuti alur ini:
> 1. **Fisik (L1):** Cek interface *running* atau *down*.
> 2. **Data Link (L2):** Bridge, VLAN, PPPoE.
> 3. **Network (L3):** IP Address, Routing (Static/OSPF).
> 4. **Layanan:** NAT, DHCP, DNS.
> 5. **Keamanan:** Firewall, Queue.

**Kebiasaan Wajib:**
- Selalu cek versi awal: `/system resource print` (Sintaks OSPF/BGP v6 dan v7 **BERBEDA**).
- Gunakan **Tab** untuk *autocomplete* dan **`?`** untuk melihat parameter.
- **Safe Mode (`Ctrl+X`)**: Sangat penting! Jika kamu mengubah routing/firewall dan terputus, router akan otomatis *rollback* ke konfigurasi sebelumnya jika kamu menekan `Ctrl+X` lagi.
- **Beri `comment=`** di setiap rule/interface. Ini menyelamatkanmu saat *troubleshooting*.
- **Berbicara saat *Share Screen***: "Saya cek IP dulu, lalu saya pastikan routing-nya..." Penilai NOC sangat menghargai *problem solving*, bukan hanya hasil akhir.

---

## 1. Konfigurasi Dasar & Keamanan Awal

```routeros
# Cek versi, uptime, CPU, RAM
/system resource print

# Identitas Router (Wajib jika soal meminta nama spesifik)
/system identity set name=NOC-Router-01

# Timezone (Penting untuk akurasi log)
/system clock set time-zone-name=Asia/Jakarta

# Ganti password (Praktik baik)
/user set admin password=PasswordKuat123

# Matikan service yang tidak perlu (Kurangi celah keamanan)
/ip service disable telnet,ftp,www,api,api-ssl
```

> [!WARNING]
> **Hati-hati dengan `system reset-configuration`!** Hanya gunakan jika kamu benar-benar yakin router dalam keadaan kotor dan soal mengizinkan reset. Di PNETLab, biasanya router sudah bersih (default).

---

## 2. Interface, IP & Bridge Dasar

> [!NOTE]
> **Konsep Bridge:** Anggap Bridge itu seperti *Switch Virtual* di dalam RouterOS. Jika kamu memasukkan interface fisik (ether2, ether3) ke dalam Bridge, maka IP Address **harus dipasang di interface Bridge-nya**, bukan di ether-nya.

```routeros
# Cek daftar interface (Cari flag 'R' = Running)
/interface print
/interface print stats

# Beri label fungsi (Jangan rename ether1 jika soal menyebut 'ether1')
/interface set ether1 comment="WAN / UPSTREAM ISP"
/interface set ether2 comment="LAN / PELANGGAN"

# Beri IP Address
/ip address add address=192.168.10.1/24 interface=ether2 comment=LAN
/ip address add address=10.10.10.2/30 interface=ether1 comment=WAN

# Loopback (Sangat berguna untuk Router-ID OSPF)
/interface bridge add name=lo
/ip address add address=1.1.1.1/32 interface=lo
```

---

## 3. Routing, DNS & NAT (Jantung Internet)

### Routing Statis
```routeros
# Default Route (Jalan menuju internet)
/ip route add dst-address=0.0.0.0/0 gateway=10.10.10.1

# Route ke jaringan spesifik (Misal: ada kantor cabang di 192.168.20.0/24)
/ip route add dst-address=192.168.20.0/24 gateway=10.0.0.2

# Cek (Flag: A=Active, S=Static, D=Dynamic)
/ip route print
```

### DNS
```routeros
# Izinkan client pakai router sebagai DNS
/ip dns set servers=8.8.8.8,1.1.1.1 allow-remote-requests=yes

# Tes resolve
/ping google.com count=3
```

### NAT (Network Address Translation)
> [!TIP]
> **Kenapa pakai Interface List?** Karena jika koneksi WAN kamu berubah dari DHCP ke PPPoE, kamu tidak perlu mengubah rule NAT. Cukup tambahkan interface baru ke list WAN.

```routeros
# Buat List WAN
/interface list add name=WAN
/interface list member add list=WAN interface=ether1
# (Jika pakai PPPoE, tambahkan pppoe-out1 ke list WAN ini)

# Rule NAT Masquerade
/ip firewall nat add chain=srcnat out-interface-list=WAN action=masquerade comment="NAT ke Internet"

# Port Forwarding (Akses Web Server LAN dari Luar)
/ip firewall nat add chain=dstnat in-interface-list=WAN protocol=tcp dst-port=8080 \
    action=dst-nat to-addresses=192.168.10.10 to-ports=80 comment="Forward 8080 ke Web Server"

# Cek counter (Harus naik jika ada traffic)
/ip firewall nat print stats
```

---

## 4. DHCP Server & Client

```routeros
# DHCP Client (Jika ISP memberi IP otomatis)
/ip dhcp-client add interface=ether1 disabled=no add-default-route=yes use-peer-dns=yes

# --- DHCP Server (Untuk LAN) ---
# 1. Buat IP Pool (Pastikan tidak mencakup IP Gateway .1)
/ip pool add name=pool-lan ranges=192.168.10.10-192.168.10.254

# 2. Buat DHCP Server di interface LAN
/ip dhcp-server add name=dhcp-lan interface=ether2 address-pool=pool-lan lease-time=1d disabled=no

# 3. Setting Network (Gateway & DNS yang dibagikan ke Client)
/ip dhcp-server network add address=192.168.10.0/24 gateway=192.168.10.1 dns-server=192.168.10.1

# IP Statis untuk perangkat tertentu (Reservasi MAC Address)
/ip dhcp-server lease add address=192.168.10.50 mac-address=AA:BB:CC:DD:EE:FF server=dhcp-lan
```

> [!TIP]
> **Cara Cepat (Wizard):** `/ip dhcp-server setup` (Ikuti saja prompt-nya, sangat berguna saat waktu mepet).

---

## 5. PPPoE Client & Server (Khas ISP!)

> [!IMPORTANT]
> Di perusahaan ISP (seperti GMDP), PPPoE adalah nyawa utama. Pahami betul bagian ini.

### A. PPPoE Client (Menerima Koneksi dari ISP)
```routeros
/interface pppoe-client add name=pppoe-out1 interface=ether1 user=user_isp password=pass_isp \
    add-default-route=yes use-peer-dns=yes disabled=no

# Cek status (Cari flag 'R' dan IP Address yang didapat)
/interface pppoe-client print
/interface pppoe-client monitor pppoe-out1 once
```
> [!WARNING]
> Jika kamu pakai PPPoE Client, ingat: **NAT harus keluar lewat `pppoe-out1`**, bukan `ether1`!

### B. PPPoE Server (Melayani Pelanggan)
```routeros
# 1. Pool IP Pelanggan (Gunakan subnet berbeda dari LAN/WAN)
/ip pool add name=pool-pppoe ranges=172.16.10.10-172.16.10.254

# 2. Profile (Atur DNS & Limit Bandwidth di sini)
# rate-limit = Upload/Download (dari sisi pelanggan)
/ppp profile add name=profile-isp local-address=172.16.10.1 remote-address=pool-pppoe \
    dns-server=8.8.8.8 rate-limit=5M/10M only-one=yes

# 3. Aktifkan Server di interface LAN
/interface pppoe-server server add service-name=isp interface=ether2 default-profile=profile-isp disabled=no

# 4. Buat Akun Pelanggan
/ppp secret add name=pelanggan1 password=rahasia service=pppoe profile=profile-isp

# 5. Cek Sesi Aktif
/ppp active print
```

---

## 6. VLAN & Bridge VLAN Filtering (Sering Diuji!)

> [!NOTE]
> **Konsep:** Bayangkan Router adalah gedung, dan VLAN adalah lantai yang berbeda. Bridge adalah tangga utama. Agar antar lantai tidak bisa saling akses sembarangan, kita pakai VLAN Filtering.

### A. VLAN Sederhana (Router-on-a-Stick)
```routeros
/interface vlan add name=vlan10 interface=ether2 vlan-id=10
/ip address add address=192.168.10.1/24 interface=vlan10
```

### B. Bridge VLAN Filtering (Cara Modern - Sering Keluar di Praktik!)
*Skenario: ether2 = Access VLAN10, ether3 = Access VLAN20, ether4 = Trunk (ke Switch).*
```routeros
# 1. Buat Bridge (JANGAN aktifkan filtering dulu)
/interface bridge add name=bridge-lan

# 2. Masukkan port ke bridge, setting PVID (Port VLAN ID) untuk Access
/interface bridge port add bridge=bridge-lan interface=ether2 pvid=10
/interface bridge port add bridge=bridge-lan interface=ether3 pvid=20
/interface bridge port add bridge=bridge-lan interface=ether4 comment="TRUNK"

# 3. Tabel VLAN (Tagged = Trunk + Bridge, Untagged = Access)
/interface bridge vlan add bridge=bridge-lan vlan-ids=10 tagged=bridge-lan,ether4 untagged=ether2
/interface bridge vlan add bridge=bridge-lan vlan-ids=20 tagged=bridge-lan,ether4 untagged=ether3

# 4. Interface VLAN untuk IP Gateway
/interface vlan add name=vlan10 interface=bridge-lan vlan-id=10
/ip address add address=192.168.10.1/24 interface=vlan10

# 5. AKTIFKAN FILTERING PALING AKHIR!
/interface bridge set bridge-lan vlan-filtering=yes
```
> [!CAUTION]
> Jika kamu mengaktifkan `vlan-filtering=yes` sebelum tabel VLAN lengkap, kamu akan **terputus** dari router (jika akses via Winbox/SSH). Gunakan Safe Mode!

---

## 7. Routing Dinamis: OSPF (v6 vs v7)

> [!NOTE]
> **Konsep OSPF:** Router saling "ngobrol" untuk bertukar informasi jalur (routing table) secara otomatis. Syarat "ngobrol": Area harus sama, Hello/Dead timer sama, Subnet link harus sama.

**Cek versi dulu (`/system resource print`)!**

```routeros
# --- ROUTEROS v6 ---
/routing ospf instance set [find default=yes] router-id=1.1.1.1
/routing ospf network add network=10.0.0.0/30 area=backbone
/routing ospf network add network=192.168.10.0/24 area=backbone

# --- ROUTEROS v7 (Sintaks Berbeda!) ---
/routing ospf instance add name=ospf-main version=2 router-id=1.1.1.1
/routing ospf area add name=backbone area-id=0.0.0.0 instance=ospf-main
/routing ospf interface-template add area=backbone networks=10.0.0.0/30
/routing ospf interface-template add area=backbone networks=192.168.10.0/24 passive

# VERIFIKASI (Wajib Full)
/routing ospf neighbor print
```

---

## 8. Firewall: Proteksi Router & Client

> [!WARNING]
> **Urutan Rule Sangat Mutlak!** Dibaca dari atas ke bawah. Jika rule `drop` ada di atas `accept established`, maka koneksi yang sah pun akan terputus.

```routeros
# 1. Proteksi Router Sendiri (Chain Input)
/ip firewall filter
add chain=input action=accept connection-state=established,related comment="Izinkan koneksi sah"
add chain=input action=drop connection-state=invalid comment="Buang paket aneh"
add chain=input action=accept protocol=icmp comment="Izinkan Ping ke Router"
add chain=input action=accept in-interface=ether2 comment="Izinkan akses manajemen dari LAN"
add chain=input action=drop in-interface=ether1 comment="Blokir akses dari WAN"

# 2. Proteksi Traffic Client (Chain Forward)
add chain=forward action=accept connection-state=established,related
add chain=forward action=drop connection-state=invalid
# (Tambahkan rule blokir khusus di sini jika diminta soal)
```

---

## 9. Bandwidth Management (Queue)

```routeros
# Simple Queue (Target = IP Client, max-limit = Upload/Download)
/queue simple add name="Limit-Pelanggan" target=192.168.10.10/32 max-limit=5M/5M

# Queue dengan Burst (Kecepatan sesaat lebih tinggi)
/queue simple add name="Burst-Client" target=192.168.10.20/32 max-limit=5M/5M \
    burst-limit=10M/10M burst-threshold=3M/3M burst-time=10s/10s
```
> [!NOTE]
> Jika pelanggan menggunakan PPPoE, **JANGAN pakai Simple Queue**. Gunakan `rate-limit` di `/ppp profile` (lihat bagian 5).

---

## 10. Failover Dua ISP

*Skenario: ISP1 mati, otomatis pindah ke ISP2.*

```routeros
# 1. Daftarkan kedua interface ke list WAN (untuk NAT)
/interface list member add list=WAN interface=ether1
/interface list member add list=WAN interface=ether2
/ip firewall nat add chain=srcnat out-interface-list=WAN action=masquerade

# 2. Routing Failover (Distance kecil = Prioritas)
# check-gateway=ping berfungsi agar route otomatis mati jika gateway tidak bisa di-ping
/ip route add dst-address=0.0.0.0/0 gateway=10.10.1.1 distance=1 check-gateway=ping comment="ISP1"
/ip route add dst-address=0.0.0.0/0 gateway=10.10.2.1 distance=2 check-gateway=ping comment="ISP2"
```

---

## 11. Layanan Pendukung NOC (NTP, SNMP, Log)

```routeros
# NTP (Agar jam router akurat, penting untuk sinkronisasi Log)
/system ntp client set enabled=yes
/system ntp client servers add address=pool.ntp.org

# SNMP (Agar bisa dipantau oleh sistem monitoring seperti Cacti/The Dude)
/snmp set enabled=yes contact=noc@gmdp.net.id location="POP-Tangerang"
/snmp community set [find default=yes] name=rahasia addresses=192.168.10.0/24

# Log (Melihat kejadian real-time)
/log print
/log print follow  # Tekan Ctrl+C untuk stop
```

---

## 12. Monitoring & Troubleshooting (Kunci NOC)

> [!TIP]
> **Alur Troubleshooting "Client Tidak Bisa Internet":**
> 1. **Link:** `/interface print` -> Running?
> 2. **IP:** Client dapat IP? (`/ip dhcp-server lease print` atau `/ppp active print`)
> 3. **Gateway:** Bisa ping gateway dari client?
> 4. **Routing:** Ada default route aktif di router? (`/ip route print`)
> 5. **NAT:** Counter NAT naik saat client akses? (`/ip firewall nat print stats`)
> 6. **Firewall:** Ada rule drop yang menghalangi?
> 7. **DNS:** Bisa ping 8.8.8.8 tapi tidak bisa ping google.com? Berarti masalah DNS.

```routeros
# Alat Monitoring
/tool torch interface=ether1        # Lihat siapa yang pakai bandwidth besar
/tool sniffer quick interface=ether1 ip-protocol=icmp  # "Wireshark" mini
/interface ethernet monitor ether1 once  # Cek speed/duplex kabel
/ip neighbor print                  # Cek tetangga MikroTik
/system resource monitor            # Cek CPU/RAM realtime

# Troubleshooting PPPoE Khusus
/log print follow where message~"pppoe"  # Cari error login
/ip pool used print                      # Apakah IP Pool habis?
```

---

## 13. Backup & Export (Wajib!)

```routeros
# Backup Binary (Untuk Restore di versi RouterOS yang sama)
/system backup save name=hasil-tes-noc

# Export Text (Bisa dibaca di Notepad, sangat berguna untuk dokumentasi)
/export file=hasil-tes-noc
/export hide-sensitive file=hasil-tes-noc-aman  # Tanpa password
```

---

## 14. Contoh Lab Lengkap (4 Router)

*Latih ini berulang kali sampai lancar.*

**Topologi:**
```text
[R-CUST] --(PPPoE)-- [R-ACCESS] --(OSPF)-- [R-CORE] --(Static)-- [R-INET]
Subnet PPPoE: 172.16.10.0/24
Link Core-Access: 10.0.0.0/30
Link Core-INET: 203.0.113.0/30
```

**Langkah Pengerjaan yang Disarankan:**
1. **R-INET:** Beri IP `203.0.113.1/30` dan loopback `8.8.8.8/32`.
2. **R-CORE:** IP `203.0.113.2/30` (ke INET) dan `10.0.0.1/30` (ke Access). Buat default route ke INET. Set OSPF.
3. **R-ACCESS:** IP `10.0.0.2/30` (ke Core). Buat default route ke Core. Set OSPF. Buat PPPoE Server (Pool: `172.16.10.10-254`).
4. **R-CUST:** Buat PPPoE Client ke R-ACCESS. Beri IP LAN `192.168.50.1/24`. Buat NAT Masquerade ke `pppoe-out1`. Buat DHCP Server untuk LAN.
5. **R-CORE (Tambahan):** Harus ada route statis balik ke subnet PPPoE (`/ip route add dst-address=172.16.10.0/24 gateway=10.0.0.2`) dan NAT Masquerade ke arah INET.

**Uji Akhir:**
- Di R-CUST, ping `8.8.8.8` (harus tembus).
- Di R-CORE, cek `/routing ospf neighbor print` (harus state `Full`).
- Cek `/ppp active print` di R-ACCESS (harus ada `pelanggan1`).

---

## 🧭 Jebakan Umum (Baca Sebelum Tes!)

1. **Subnet Bentrok:** Pool PPPoE/DHCP tidak boleh sama dengan subnet WAN/LAN.
2. **Lupa Route Balik:** Jika router A bisa ping router B, tapi router B tidak bisa ping A, berarti tidak ada route balik.
3. **Salah Interface NAT:** Jika pakai PPPoE, NAT *out-interface* harus `pppoe-out1`, bukan `ether1`.
4. **Lupa `vlan-filtering=yes`:** VLAN tidak akan jalan jika filtering belum diaktifkan (dan pastikan urutannya di akhir).
5. **Terputus saat setting Firewall:** Selalu gunakan Safe Mode (`Ctrl+X`).

> [!TIP]
> **Tips PNETLab:**
> - Gunakan **CHR** (Cloud Hosted Router) untuk MikroTik.
> - Jika *copy-paste* konfigurasi panjang terputus, paste per blok kecil.
> - Pastikan login default (`admin` kosong) sudah diganti jika diminta.
> - Jangan lupa simpan lab sebelum ditutup.
