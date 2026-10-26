# 🌐 MikroTik CLI Cheatsheet: Tes Skill Junior NOC (GMDP, PNETLab)

> **Tes:** Selasa, 27 Oktober 2026, 11.00 WIB, Zoom, konfigurasi MikroTik di PNETLab.
> Dokumen ini dikembangkan dari cheatsheet sumber, dengan perbaikan konflik IP/interface, penambahan topik yang sering diuji (routing antar router, OSPF, bridge VLAN filtering, troubleshooting PPPoE/DHCP), dan satu contoh lab lengkap yang bisa dilatih ulang.

---

## 📌 Daftar Isi

0. [Strategi Tes & Aturan Main](#0-strategi-tes--aturan-main)
1. [Konfigurasi Dasar & Identitas](#1-konfigurasi-dasar--identitas)
2. [Interface & IP Address](#2-interface--ip-address)
3. [Routing Statis, DNS & NAT](#3-routing-statis-dns--nat)
4. [DHCP Client & Server](#4-dhcp-client--server)
5. [PPPoE Client](#5-pppoe-client)
6. [PPPoE Server](#6-pppoe-server)
7. [VLAN & Bridge](#7-vlan--bridge)
8. [Routing Dinamis: OSPF (+BGP singkat)](#8-routing-dinamis-ospf--bgp-singkat)
9. [Firewall](#9-firewall)
10. [Bandwidth Management (Queue)](#10-bandwidth-management-queue)
11. [Failover Dua ISP](#11-failover-dua-isp)
12. [Layanan Pendukung NOC (NTP, SNMP, Log)](#12-layanan-pendukung-noc-ntp-snmp-log)
13. [Monitoring & Troubleshooting](#13-monitoring--troubleshooting)
14. [Backup & Export](#14-backup--export)
15. [Contoh Lab Lengkap (4 Router)](#15-contoh-lab-lengkap-4-router)
16. [Tabel Gejala → Penyebab → Cek](#16-tabel-gejala--penyebab--cek)
17. [Jebakan Umum](#17-jebakan-umum)
18. [Tips Khusus PNETLab](#18-tips-khusus-pnetlab)
19. [Checklist Sebelum Selesai](#19-checklist-sebelum-selesai)

---

## 0. Strategi Tes & Aturan Main

**Urutan kerja yang aman (dari bawah ke atas, ala OSI):**

1. Baca seluruh soal, gambar topologi kecil di catatan (nama router, interface, subnet).
2. **L1/L2:** cek interface `running`, VLAN/bridge.
3. **L3:** IP address, lalu routing (static/OSPF), lalu NAT.
4. **Layanan:** DHCP, PPPoE, DNS.
5. **Kebijakan:** firewall, queue.
6. **Verifikasi tiap langkah** dengan `ping`, jangan menunggu di akhir.
7. **Export/backup** setelah selesai.

**Kebiasaan yang wajib:**

- Cek versi dulu: `/system resource print`. Sintaks **RouterOS v6 dan v7 berbeda** (terutama routing/OSPF/BGP).
- Gunakan **Tab** untuk autocomplete dan **`?`** untuk bantuan parameter.
- Gunakan **Safe Mode** (`Ctrl+X`) sebelum mengubah routing/firewall di router yang kamu akses lewat jaringan. Kalau terputus, perubahan otomatis di-rollback.
- Pakai `comment=` untuk menandai fungsi interface/rule. Jangan merename `etherX` kalau soal mengacu ke nama `etherX`.
- **Ucapkan alur berpikirmu** saat share screen ("saya cek IP dulu, lalu route..."). Penilai juga menilai cara debug.

> 💡 Kalau ragu soal subnet, ingat: `/30` = 2 host, `/29` = 6, `/28` = 14, `/27` = 30, `/26` = 62, `/25` = 126, `/24` = 254.

---

## 1. Konfigurasi Dasar & Identitas

```routeros
# Cek versi, uptime, CPU, RAM
/system resource print

# Ubah nama router
/system identity set name=NOC-Router-01

# Timezone & jam
/system clock set time-zone-name=Asia/Jakarta

# Ganti password admin (jika diminta/praktik baik)
/user set admin password=PasswordKuat123

# Matikan service yang tidak perlu (sisakan ssh/winbox)
/ip service disable telnet,ftp,www,api,api-ssl

# Reset konfigurasi kosong (hanya jika soal mengizinkan!)
/system reset-configuration no-defaults=yes skip-backup=yes
```

> ⚠️ Di PNETLab/CHR, jangan matikan `telnet` kalau kamu masuk lewat console telnet dan belum yakin. Console PNETLab biasanya lewat jalur terpisah, tapi lebih aman matikan service **di akhir** atau jangan disentuh kalau tidak diminta.

---

## 2. Interface & IP Address

```routeros
# Daftar interface (cek flag R = running)
/interface print
/interface print stats

# Beri label fungsi tanpa mengganti nama
/interface set ether1 comment="WAN / UPSTREAM"
/interface set ether2 comment="LAN / PELANGGAN"

# Beri IP address
/ip address add address=192.168.10.1/24 interface=ether2 comment=LAN
/ip address add address=10.10.10.2/30 interface=ether1 comment=WAN

# Cek / hapus
/ip address print
/ip address remove [find where address="10.10.10.2/30"]

# Loopback (dipakai untuk router-id OSPF, dll.)
/interface bridge add name=lo
/ip address add address=1.1.1.1/32 interface=lo
```

> **Aturan emas:** satu subnet tidak boleh dipasang di dua interface berbeda pada router yang sama. Kalau `ether2` dimasukkan ke bridge, **pindahkan IP ke interface bridge**.

---

## 3. Routing Statis, DNS & NAT

### Routing statis

```routeros
# Default route (gateway ke internet/upstream)
/ip route add dst-address=0.0.0.0/0 gateway=10.10.10.1

# Route ke jaringan di router lain
/ip route add dst-address=192.168.20.0/24 gateway=10.0.0.2

# Cek (cari flag A = active, S = static, D = dynamic)
/ip route print
/ip route print where dst-address=0.0.0.0/0
```

### DNS

```routeros
/ip dns set servers=8.8.8.8,1.1.1.1 allow-remote-requests=yes

# DNS statis (jika diminta)
/ip dns static add name=router.lab address=192.168.10.1

# Tes resolve
/ping google.com count=3
```

### NAT (Masquerade)

```routeros
# Dengan interface fisik
/ip firewall nat add chain=srcnat out-interface=ether1 action=masquerade comment="NAT LAN ke WAN"

# Praktik lebih rapi: pakai interface list (aman untuk dua ISP / PPPoE)
/interface list add name=WAN
/interface list member add list=WAN interface=ether1
/ip firewall nat add chain=srcnat out-interface-list=WAN action=masquerade

# Port forward (akses server di LAN dari luar)
/ip firewall nat add chain=dstnat in-interface-list=WAN protocol=tcp dst-port=8080 \
    action=dst-nat to-addresses=192.168.10.10 to-ports=80 comment="Forward 8080 ke web server"

# Cek apakah NAT jalan (counter bytes/packets harus naik)
/ip firewall nat print stats
```

> ⚠️ Kalau WAN memakai **PPPoE client**, interface keluar adalah `pppoe-out1`, bukan `ether1`. Gunakan `out-interface=pppoe-out1` atau interface list.

---

## 4. DHCP Client & Server

### Cara cepat: wizard

```routeros
/ip dhcp-server setup
```

Wizard menanyakan interface, subnet, gateway, pool, DNS, dan lease time. Bagus saat waktu mepet. Tapi pahami juga versi manual di bawah.

### Manual

```routeros
# DHCP Client (WAN jika upstream memberi IP lewat DHCP)
/ip dhcp-client add interface=ether1 disabled=no add-default-route=yes use-peer-dns=yes

# DHCP Server (LAN)
/ip pool add name=pool-lan ranges=192.168.10.10-192.168.10.254
/ip dhcp-server add name=dhcp-lan interface=ether2 address-pool=pool-lan lease-time=1d disabled=no
/ip dhcp-server network add address=192.168.10.0/24 gateway=192.168.10.1 dns-server=192.168.10.1

# IP statis untuk perangkat tertentu (reservation)
/ip dhcp-server lease add address=192.168.10.50 mac-address=AA:BB:CC:DD:EE:FF server=dhcp-lan

# Cek
/ip dhcp-server print
/ip dhcp-server lease print
/ip dhcp-client print
```

> Pastikan **range pool tidak mencakup IP gateway** (`192.168.10.1`). Pool di atas mulai dari `.10`, jadi aman.

---

## 5. PPPoE Client

```routeros
/interface pppoe-client add name=pppoe-out1 interface=ether1 user=user_isp password=pass_isp \
    add-default-route=yes use-peer-dns=yes disabled=no

# Cek status (R = running, IP didapat dari server)
/interface pppoe-client print
/ip address print
/interface pppoe-client monitor pppoe-out1 once
```

Bila PPPoE client dipakai, **NAT harus keluar lewat `pppoe-out1`** (lihat bagian 3).

---

## 6. PPPoE Server

```routeros
# 1. Pool IP pelanggan (SUBNET BERBEDA dari WAN/LAN lain)
/ip pool add name=pool-pppoe ranges=172.16.10.10-172.16.10.254

# 2. Profile: IP lokal router, pool, DNS, limit kecepatan
/ppp profile add name=profile-isp local-address=172.16.10.1 remote-address=pool-pppoe \
    dns-server=8.8.8.8 rate-limit=5M/5M only-one=yes

# 3. Aktifkan PPPoE server di interface sisi pelanggan
/interface pppoe-server server add service-name=isp interface=ether2 default-profile=profile-isp disabled=no

# 4. User/secret pelanggan
/ppp secret add name=pelanggan1 password=rahasia service=pppoe profile=profile-isp

# 5. Monitoring
/ppp active print
/ppp secret print
/ip pool used print
/interface pppoe-server print
```

**Catatan penting:**

- `rate-limit=rx/tx` dibaca dari sisi **pelanggan**: angka pertama = **upload** pelanggan, angka kedua = **download**.
- Pelanggan harus bisa keluar internet: pastikan ada **default route** dan **NAT** untuk subnet `172.16.10.0/24`.
- Kalau server dipasang di atas **bridge**, isi `interface=bridge-lan`, bukan port-nya.
- `only-one=yes` mencegah satu akun dipakai login ganda.
- Alamat lokal (`172.16.10.1`) tidak boleh sama dengan IP lain di router tersebut.

---

## 7. VLAN & Bridge

### 7a. VLAN sederhana di satu interface (tanpa bridge)

```routeros
/interface vlan add name=vlan10 interface=ether2 vlan-id=10
/interface vlan add name=vlan20 interface=ether2 vlan-id=20
/ip address add address=192.168.10.1/24 interface=vlan10
/ip address add address=192.168.20.1/24 interface=vlan20
```

Cocok untuk skenario router-on-a-stick: port `ether2` jadi **trunk** ke switch.

### 7b. Bridge VLAN Filtering (cara modern, sering diuji)

Contoh: `ether2` = access VLAN 10, `ether3` = access VLAN 20, `ether4` = trunk (VLAN 10 dan 20), router sebagai gateway kedua VLAN.

```routeros
# 1. Bridge (vlan-filtering JANGAN diaktifkan dulu)
/interface bridge add name=bridge-lan

# 2. Port: access diberi pvid, trunk tanpa pvid khusus
/interface bridge port add bridge=bridge-lan interface=ether2 pvid=10
/interface bridge port add bridge=bridge-lan interface=ether3 pvid=20
/interface bridge port add bridge=bridge-lan interface=ether4

# 3. Tabel VLAN: tagged = trunk + bridge sendiri, untagged = port access
/interface bridge vlan add bridge=bridge-lan vlan-ids=10 tagged=bridge-lan,ether4 untagged=ether2
/interface bridge vlan add bridge=bridge-lan vlan-ids=20 tagged=bridge-lan,ether4 untagged=ether3

# 4. Interface VLAN di atas bridge untuk IP gateway (inter-VLAN routing)
/interface vlan add name=vlan10 interface=bridge-lan vlan-id=10
/interface vlan add name=vlan20 interface=bridge-lan vlan-id=20
/ip address add address=192.168.10.1/24 interface=vlan10
/ip address add address=192.168.20.1/24 interface=vlan20

# 5. Aktifkan filtering PALING AKHIR
/interface bridge set bridge-lan vlan-filtering=yes

# Cek
/interface bridge vlan print
/interface bridge port print
/interface bridge host print
```

**Kenapa vlan-filtering diaktifkan terakhir?** Kalau diaktifkan sebelum tabel VLAN lengkap, kamu bisa terputus dari router sendiri.

### 7c. Bridge biasa (switching, tanpa VLAN)

```routeros
/interface bridge add name=bridge-lan
/interface bridge port add bridge=bridge-lan interface=ether2
/interface bridge port add bridge=bridge-lan interface=ether3
/ip address add address=192.168.10.1/24 interface=bridge-lan
```

---

## 8. Routing Dinamis: OSPF (+BGP singkat)

> **Cek versi dulu** dengan `/system resource print`. Sintaks v6 dan v7 berbeda.

### OSPF di RouterOS v6

```routeros
/interface bridge add name=lo
/ip address add address=1.1.1.1/32 interface=lo

/routing ospf instance set [find default=yes] router-id=1.1.1.1
/routing ospf network add network=10.0.0.0/30 area=backbone
/routing ospf network add network=192.168.10.0/24 area=backbone
/routing ospf network add network=1.1.1.1/32 area=backbone

# Interface pelanggan/LAN sebaiknya passive (tidak kirim hello)
/routing ospf interface add interface=ether2 passive=yes

# Sebarkan default route dari router yang punya jalur internet (opsional)
/routing ospf instance set [find default=yes] distribute-default=if-installed-as-type-1
```

### OSPF di RouterOS v7

```routeros
/routing ospf instance add name=ospf-main version=2 router-id=1.1.1.1
/routing ospf area add name=backbone area-id=0.0.0.0 instance=ospf-main
/routing ospf interface-template add area=backbone networks=10.0.0.0/30
/routing ospf interface-template add area=backbone networks=192.168.10.0/24 passive
/routing ospf interface-template add area=backbone networks=1.1.1.1/32 passive
```

### Verifikasi OSPF (v6 dan v7)

```routeros
/routing ospf neighbor print     # state harus "Full"
/ip route print                  # cari flag "o" (OSPF)
/ping 2.2.2.2 src-address=1.1.1.1
```

**Neighbor tidak mau Full? Cek:** subnet/mask link sama, area sama, hello/dead timer sama, tipe network sama, tidak ada firewall yang memblok protokol OSPF (protokol 89), MTU sama.

### BGP (kemungkinan kecil diuji, hanya kerangka)

```routeros
# v6
/routing bgp instance set default as=65001 router-id=1.1.1.1
/routing bgp peer add name=to-isp remote-address=10.0.0.2 remote-as=65002
/routing bgp network add network=192.168.10.0/24
/routing bgp peer print
```

Di v7 sintaksnya memakai `/routing bgp connection`. Cek parameter dengan `?`.

---

## 9. Firewall

### Proteksi router (chain input)

```routeros
/ip firewall filter
add chain=input action=accept connection-state=established,related comment="Izinkan koneksi sah"
add chain=input action=drop connection-state=invalid comment="Buang invalid"
add chain=input action=accept protocol=icmp comment="Izinkan ping"
add chain=input action=accept in-interface=ether2 comment="Izinkan dari LAN"
add chain=input action=drop in-interface=ether1 comment="Blok akses ke router dari WAN"
```

> ⚠️ **Urutan rule sangat penting** (diproses dari atas ke bawah). Kalau kamu mengelola router lewat WAN, rule `drop in-interface=WAN` akan mengunci kamu. Di lab, izinkan dulu sumber manajemen sebelum drop.
> Sisipkan di paling atas dengan `place-before=0`.

### Proteksi traffic pelanggan (chain forward)

```routeros
/ip firewall filter
add chain=forward action=accept connection-state=established,related
add chain=forward action=drop connection-state=invalid

# Contoh blokir satu IP/situs berdasarkan address-list
/ip firewall address-list add list=blokir address=1.2.3.4
/ip firewall filter add chain=forward src-address-list=blokir action=drop comment="Blokir IP tertentu"

# Contoh blokir antar VLAN
/ip firewall filter add chain=forward src-address=192.168.10.0/24 dst-address=192.168.20.0/24 action=drop
```

### Cek

```routeros
/ip firewall filter print
/ip firewall filter print stats     # lihat counter rule mana yang kena
/ip firewall connection print
```

---

## 10. Bandwidth Management (Queue)

```routeros
# Simple queue: max-limit = upload/download dari sisi TARGET
/queue simple add name=Limit-Pelanggan target=192.168.10.10/32 max-limit=5M/5M

# Limit satu subnet
/queue simple add name=Limit-LAN target=192.168.10.0/24 max-limit=20M/20M

# Dengan burst (kecepatan lebih tinggi sesaat)
/queue simple add name=Pelanggan-Burst target=192.168.10.20/32 max-limit=5M/5M \
    burst-limit=10M/10M burst-threshold=3M/3M burst-time=10s/10s

# Cek
/queue simple print stats
```

**Untuk PPPoE:** lebih praktis pakai `rate-limit` di `/ppp profile` (lihat bagian 6), karena queue dinamis dibuat otomatis per sesi.

> ⚠️ **CHR gratis di PNETLab dibatasi throughput upload sekitar 1 Mbps.** Hasil tes bandwidth tidak akurat untuk membuktikan queue. Cukup tunjukkan konfigurasi dan counter queue.

---

## 11. Failover Dua ISP

```routeros
# NAT untuk kedua jalur (pakai interface list)
/interface list add name=WAN
/interface list member add list=WAN interface=ether1
/interface list member add list=WAN interface=ether2
/ip firewall nat add chain=srcnat out-interface-list=WAN action=masquerade

# Dua default route: distance kecil = utama; check-gateway mematikan route otomatis kalau gateway tidak membalas
/ip route add dst-address=0.0.0.0/0 gateway=10.10.1.1 distance=1 check-gateway=ping comment=ISP1
/ip route add dst-address=0.0.0.0/0 gateway=10.10.2.1 distance=2 comment=ISP2

# Alternatif: Netwatch
/tool netwatch add host=10.10.1.1 interval=10s \
    down-script="/ip route disable [find where comment=ISP1]" \
    up-script="/ip route enable [find where comment=ISP1]"

# Uji: matikan ISP1 lalu cek jalur aktif
/ip route print where dst-address=0.0.0.0/0
/tool traceroute 8.8.8.8
```

> Gunakan **`comment`** untuk menandai route, lebih aman daripada mencari berdasarkan IP gateway.

---

## 12. Layanan Pendukung NOC (NTP, SNMP, Log)

```routeros
# NTP (RouterOS v6)
/system ntp client set enabled=yes primary-ntp=202.65.114.202 secondary-ntp=129.250.35.250

# NTP (RouterOS v7)
/system ntp client set enabled=yes
/system ntp client servers add address=pool.ntp.org

# SNMP agar bisa dimonitor (The Dude, Cacti, MRTG, LibreNMS)
/snmp set enabled=yes contact=noc@contoh.id location="POP-Tangerang"
/snmp community set [find default=yes] name=komunitas-rahasia addresses=192.168.10.0/24

# Log
/log print
/log print where message~"pppoe"
/log print follow                    # lihat real-time (Ctrl+C untuk berhenti)
```

> Karena kamu berpengalaman dengan The Dude, MRTG, dan Cacti, **SNMP** adalah titik temu yang bagus. Kalau ditanya "bagaimana router dimonitor?", jawab dengan SNMP + community yang dibatasi per alamat.

---

## 13. Monitoring & Troubleshooting

### Alat dasar

```routeros
/ping 8.8.8.8 count=4
/ping 8.8.8.8 src-address=192.168.10.1        # uji dari subnet tertentu
/ping 8.8.8.8 interface=ether1                 # paksa lewat interface tertentu
/tool traceroute 8.8.8.8
/tool torch interface=ether1                   # siapa yang memakai bandwidth
/tool sniffer quick interface=ether1 ip-protocol=icmp
/tool bandwidth-test address=10.0.0.2 user=admin direction=both   # butuh router lawan sebagai server

/interface print stats
/interface ethernet monitor ether1 once        # status link, speed, duplex
/ip arp print
/ip neighbor print                             # perangkat MikroTik/CDP/LLDP tetangga
/ip firewall connection print count-only
/system resource monitor                       # CPU/RAM real-time
```

### Alur troubleshooting "client tidak bisa internet" (urut dari bawah)

1. **Link:** `/interface print` → interface running? Kabel/port benar?
2. **IP client:** dapat IP? gateway benar? (`/ip dhcp-server lease print`, `/ppp active print`)
3. **Gateway:** `ping` gateway dari client, lalu dari router ke client.
4. **Route:** `/ip route print` → ada default route yang **active**?
5. **Ke luar:** dari router `ping 8.8.8.8`. Gagal = masalah upstream/routing. Berhasil tapi client gagal = NAT/firewall.
6. **NAT:** `/ip firewall nat print stats` → counter naik saat client mencoba?
7. **Firewall:** `/ip firewall filter print stats` → ada rule drop yang counter-nya naik?
8. **DNS:** `ping 8.8.8.8` berhasil tapi `ping google.com` gagal = DNS bermasalah.
9. **Queue:** apakah limit terlalu kecil atau salah target?

### Troubleshooting PPPoE

```routeros
/ppp active print                       # sesi aktif
/ppp secret print                       # user/password/profile benar?
/interface pppoe-server print
/ip pool used print                     # pool habis?
/log print follow where message~"pppoe" # lihat error saat login
```

Kemungkinan penyebab umum: password/service salah, profile tidak ada, **pool habis**, interface server salah, **IP lokal bentrok**, VLAN/bridge tidak meneruskan frame PPPoE.

---

## 14. Backup & Export

```routeros
# Binary (restore di perangkat/versi yang sama)
/system backup save name=hasil-tes-noc

# Teks (bisa dibaca dan di-import)
/export file=hasil-tes-noc
/export hide-sensitive file=hasil-tes-noc-aman     # tanpa password

# Tampilkan konfigurasi di layar (untuk copy/screenshot)
/export

# Cek file
/file print
```

> Biasanya penilai meminta **screenshot** atau **teks hasil `/export`**. Latih cara menyalin teks dari terminal PNETLab ke notepad sebelum hari tes.

---

## 15. Contoh Lab Lengkap (4 Router)

Latih ulang sampai bisa dikerjakan dari nol dalam waktu singkat.

### Topologi

```text
[R-CUST] --ether1--(PPPoE)--ether2-- [R-ACCESS] --ether1---ether2-- [R-CORE] --ether1---ether1-- [R-INET]
                                          10.0.0.0/30                    203.0.113.0/30
 192.168.50.0/24                 .2                .1  CORE .2                 .1  INET (8.8.8.8/32)
                              pool pelanggan: 172.16.10.0/24
```

| Link | Subnet | Ujung A | Ujung B |
|---|---|---|---|
| CORE ↔ INET | 203.0.113.0/30 | INET `.1` (ether1) | CORE `.2` (ether1) |
| CORE ↔ ACCESS | 10.0.0.0/30 | CORE `.1` (ether2) | ACCESS `.2` (ether1) |
| ACCESS ↔ CUST | PPPoE 172.16.10.0/24 | ACCESS `.1` (ether2) | CUST (pppoe-out1) |

Loopback: CORE `1.1.1.1/32`, ACCESS `2.2.2.2/32`, INET `8.8.8.8/32`.

### R-INET (simulasi internet)

```routeros
/system identity set name=R-INET
/ip address add address=203.0.113.1/30 interface=ether1
/interface bridge add name=lo
/ip address add address=8.8.8.8/32 interface=lo
```

### R-CORE

```routeros
/system identity set name=R-CORE
/interface bridge add name=lo
/ip address add address=1.1.1.1/32 interface=lo
/ip address add address=203.0.113.2/30 interface=ether1 comment=UPSTREAM
/ip address add address=10.0.0.1/30 interface=ether2 comment=TO-ACCESS

/ip route add dst-address=0.0.0.0/0 gateway=203.0.113.1
# Jalur balik ke subnet pelanggan PPPoE (agar balasan NAT bisa kembali)
/ip route add dst-address=172.16.10.0/24 gateway=10.0.0.2

# NAT untuk pelanggan
/ip firewall nat add chain=srcnat src-address=172.16.10.0/24 out-interface=ether1 action=masquerade

# OSPF (sesuaikan v6/v7, contoh v6)
/routing ospf instance set [find default=yes] router-id=1.1.1.1
/routing ospf network add network=10.0.0.0/30 area=backbone
/routing ospf network add network=1.1.1.1/32 area=backbone
```

### R-ACCESS

```routeros
/system identity set name=R-ACCESS
/interface bridge add name=lo
/ip address add address=2.2.2.2/32 interface=lo
/ip address add address=10.0.0.2/30 interface=ether1 comment=TO-CORE

/ip route add dst-address=0.0.0.0/0 gateway=10.0.0.1

# OSPF (contoh v6)
/routing ospf instance set [find default=yes] router-id=2.2.2.2
/routing ospf network add network=10.0.0.0/30 area=backbone
/routing ospf network add network=2.2.2.2/32 area=backbone

# PPPoE server
/ip pool add name=pool-pppoe ranges=172.16.10.10-172.16.10.254
/ppp profile add name=profile-isp local-address=172.16.10.1 remote-address=pool-pppoe \
    dns-server=8.8.8.8 rate-limit=5M/5M
/ppp secret add name=pelanggan1 password=rahasia service=pppoe profile=profile-isp
/interface pppoe-server server add service-name=isp interface=ether2 default-profile=profile-isp disabled=no
```

### R-CUST (pelanggan, router MikroTik)

```routeros
/system identity set name=R-CUST
/interface pppoe-client add name=pppoe-out1 interface=ether1 user=pelanggan1 password=rahasia \
    add-default-route=yes use-peer-dns=yes disabled=no

# LAN pelanggan + NAT lewat pppoe-out1
/ip address add address=192.168.50.1/24 interface=ether2
/ip firewall nat add chain=srcnat out-interface=pppoe-out1 action=masquerade
/ip pool add name=pool-cust ranges=192.168.50.10-192.168.50.100
/ip dhcp-server add name=dhcp-cust interface=ether2 address-pool=pool-cust disabled=no
/ip dhcp-server network add address=192.168.50.0/24 gateway=192.168.50.1 dns-server=192.168.50.1
/ip dns set allow-remote-requests=yes servers=8.8.8.8
```

### Uji end-to-end (urut)

```routeros
# Di R-CORE
/routing ospf neighbor print         # Full dengan R-ACCESS
/ping 203.0.113.1                    # ke R-INET

# Di R-ACCESS
/ping 1.1.1.1
/ppp active print                    # pelanggan1 muncul setelah CUST login

# Di R-CUST
/interface pppoe-client print        # R, IP didapat
/ping 172.16.10.1                    # gateway PPPoE
/ping 8.8.8.8                        # tujuan akhir
```

---

## 16. Tabel Gejala → Penyebab → Cek

| Gejala | Penyebab umum | Perintah cek |
|---|---|---|
| Ping gateway gagal | Interface down, IP/mask salah, VLAN salah | `/interface print`, `/ip address print` |
| Ping IP publik gagal dari router | Tidak ada default route, upstream mati | `/ip route print`, `/tool traceroute` |
| Router bisa ping luar, client tidak | NAT belum ada / salah interface, firewall | `/ip firewall nat print stats` |
| IP bisa, nama domain tidak | DNS belum diset / `allow-remote-requests` mati | `/ip dns print` |
| Client tidak dapat IP | DHCP server mati, pool habis, interface salah | `/ip dhcp-server print`, `/ip pool used print` |
| PPPoE tidak connect | Password/service salah, interface server salah | `/log print follow`, `/ppp secret print` |
| PPPoE connect tapi tidak ada internet | Tidak ada NAT/route untuk subnet PPPoE | `/ip route print`, `/ip firewall nat print` |
| OSPF neighbor tidak Full | Subnet/area/timer beda, protokol 89 diblok | `/routing ospf neighbor print` |
| Antar VLAN tidak bisa ping | `vlan-filtering` belum aktif, tag/untag salah, tidak ada IP di vlan interface | `/interface bridge vlan print` |
| Terputus setelah ubah firewall | Rule drop menutup akses manajemen | Gunakan Safe Mode, masuk via console |
| Queue tidak bekerja | Target salah, urutan queue, throughput CHR dibatasi | `/queue simple print stats` |

---

## 17. Jebakan Umum

1. **Subnet bentrok** (misalnya subnet PPPoE sama dengan WAN) → sering jadi sebab "sudah dikonfigurasi tapi tidak jalan".
2. **IP dipasang di interface yang sudah menjadi port bridge** → pindahkan IP ke bridge.
3. **NAT memakai interface yang salah** (terutama setelah pakai PPPoE client).
4. **Lupa route balik** ke subnet di belakang router lain.
5. **Pool DHCP/PPPoE mencakup IP gateway.**
6. **`vlan-filtering=yes` diaktifkan terlalu dini.**
7. **Rule firewall urutan salah** (`drop` di atas `accept established`).
8. **Rename interface** padahal soal mengacu ke nama asli.
9. **Tidak menyimpan hasil** (lupa `/export`).
10. **Tidak mengecek versi RouterOS** lalu memakai sintaks OSPF/BGP versi yang salah.

---

## 18. Tips Khusus PNETLab

- Node MikroTik biasanya **CHR**. Login default: `admin` dengan password kosong (di login pertama bisa diminta ganti).
- Nama interface `ether1`, `ether2`, … mengikuti urutan interface yang dipasang di topologi.
- Kalau paste konfigurasi panjang **terpotong atau lambat**, paste **per blok kecil** dan cek hasilnya.
- Jika salah routing/firewall dan terputus, masuk lagi lewat **console** node, atau reset node di lab (hati-hati hilang konfigurasi).
- **Kalau node pelanggan hanya VPCS**, ia tidak bisa PPPoE. Gunakan router MikroTik sebagai client.
- Simpan topologi/konfigurasi sesuai instruksi (hindari menutup lab tanpa menyimpan).
- Coba sebelum tes: buka PNETLab, tes login, jalankan satu node, uji paste dan screenshot.

---

## 19. Checklist Sebelum Selesai

- [ ] Semua interface sesuai topologi dan **running**
- [ ] IP address sesuai soal, tidak ada subnet bentrok
- [ ] Routing (static/OSPF) aktif, `ping` antar jaringan berhasil
- [ ] NAT jalan (counter naik)
- [ ] DHCP/PPPoE bekerja (lease/sesi aktif)
- [ ] VLAN/bridge benar, antar VLAN sesuai kebijakan
- [ ] Firewall/queue sesuai permintaan, **tidak mengunci akses**
- [ ] Hasil uji (`ping`, `traceroute`) sudah ditunjukkan
- [ ] `/export` atau screenshot sudah disiapkan
- [ ] Identity/nama router sesuai soal (jika diminta)

---

## 🧭 Catatan Perubahan dari Versi Awal

| Perubahan | Alasan |
|---|---|
| Subnet PPPoE diganti `172.16.10.0/24` | Sebelumnya bentrok dengan link WAN `10.10.10.0/30` |
| IP VLAN dipisah dari IP LAN | Sebelumnya `192.168.10.0/24` dipakai dua interface |
| Rename interface diganti `comment` | Aman jika soal memakai nama `etherX` |
| NAT memakai interface list | Aman untuk PPPoE client dan dual-ISP |
| Firewall input lebih lengkap | Menghindari terkunci dan mengizinkan LAN/ICMP |
| Ditambah OSPF, routing statis antar router, bridge VLAN filtering | Topik yang paling mungkin diuji di lab multi-router |
| Ditambah troubleshooting PPPoE/DHCP, SNMP, NTP | Relevan untuk tugas NOC |
| Ditambah contoh lab 4 router dan tabel troubleshooting | Bahan latihan end-to-end |
| Failover memakai `check-gateway` dan `comment` | Lebih sederhana dan tidak bergantung IP gateway |
