# 🧠 Today I Learned (TIL) & Knowledge Base

> Log pembelajaran teknis harian, troubleshooting jaringan, dan konfigurasi environment kerja sebagai self-reminder dan pelacak progres.

---

## 📌 Tujuan Repositori

* **Self-Reminder:** Dokumentasi privat yang mudah diakses saat menghadapi kendala teknis serupa di masa depan.
* **Tracking Progress:** Mengukur pertumbuhan pemahaman konsep, tools, dan arsitektur sistem secara konsisten.
* **Knowledge Sharing:** Catatan terstruktur yang siap dibagikan atau dirapikan kembali kapan saja.

---

## 🗓️ Log Pembelajaran

### 2026-10-04: Network Analysis & Environment Setup
* **Troubleshooting tshark di WSL2:**
  * Menjalankan capture tanpa root dengan dumpcap capabilities (`cap_net_raw`, `cap_net_admin`).
  * Menggunakan flag `-p` (*disable promiscuous mode*) pada interface `eth0` untuk mengatasi error *Promiscuous mode not supported*.
  * Perintah capture ICMP: `tshark -i eth0 -p -f "icmp" -c 4`
* **MobaXterm vs WSL2:**
  * Fitur *autocorrection/autosuggestion* diproses oleh shell target OS (Fish/Zsh di WSL2/VM), bukan di terminal emulator MobaXterm.
  * Paket `cygutils` wajib terpasang di Local Terminal MobaXterm untuk menyediakan utilitas `cygstart` agar `apt` berjalan normal.
* **Manajemen APT:**
  * Pemulihan paket gantung/rusak: `sudo dpkg --configure -a && sudo apt --fix-broken install`
