# 🧠 Today I Learned (TIL) & Knowledge Base

> Log pembelajaran teknis harian, troubleshooting jaringan, dan konfigurasi environment kerja sebagai self-reminder dan pelacak progres.

---

## 📌 Tujuan Repositori

* **Self-Reminder:** Dokumentasi privat yang mudah diakses saat menghadapi kendala teknis serupa di masa depan.
* **Tracking Progress:** Mengukur pertumbuhan pemahaman konsep, tools, dan arsitektur sistem secara konsisten.
* **Knowledge Sharing:** Catatan terstruktur yang siap dibagikan atau dirapikan kembali kapan saja.

---

## 🗓️ Log Pembelajaran

### 2026-10-11: Self-Hosted AI Agent — Hermes di WSL2 + Docker Sandbox
* **Setup Hermes Agent dengan isolasi Docker di WSL2:**
  * Menjalankan agen AI otonom 24/7 sebagai `systemd user service` agar tetap hidup setelah logout/reboot.
  * Isolasi berlapis: WSL2 VM → Docker Container → `write_safe_root` → approval mode `manual`.
  * Verifikasi isolasi: agen **tidak bisa mengakses `/mnt/c`** meskipun ter-mount di WSL2.
* **Strategi API gratis tanpa kartu kredit:**
  * Fallback chain: Groq (TPM 30.000) sebagai primary, Gemini Flash sebagai cadangan.
  * Model deprecated `llama-3.3-70b-versatile` → ganti ke `meta-llama/llama-4-scout-17b-16e-instruct`.
  * Selalu uji API key dengan `curl` sebelum konfigurasi di Hermes.
* **Setup Telegram Gateway:**
  * Bot dibuat via **@BotFather**, allowlist user ID via **@userinfobot**.
  * Bot Telegram **tidak bisa inisiasi chat** — user harus kirim `/start` dulu.
* **Troubleshooting yang ditemui:**
  * `systemctl --user` gagal → aktifkan `systemd` di `/etc/wsl.conf`.
  * `Unknown provider 'groq'` → daftarkan sebagai `custom_providers` dengan prefix `custom:`.
  * `HTTP 400: reasoning_effort` → tambahkan `extra_body` di custom provider.
  * `HTTP 413: Request too large` → TPM limit, ganti model TPM lebih besar atau `/compress`.
  * `Chat not found` di Telegram → user harus kirim `/start` dulu ke bot.

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

### 📄 Dokumentasi Lengkap
* [Hermes Agent — WSL2 + Docker Sandbox](hermes_agent_wsl2_docker_sandbox.md)
* [Workstation & Network Sniffing Cheat Sheet](workstation_setup_network_sniffing_cheat_sheet.md)
