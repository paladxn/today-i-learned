# 📓 Catatan Pribadi: Setup Hermes Agent di WSL2 dengan Docker Sandbox

> **Catatan eksperimen rumahan** — dokumentasi lengkap perjalanan saya membangun AI agent otonom yang berjalan 24/7 di laptop pribadi, terisolasi penuh dari Windows, dan bisa diakses dari Telegram. Semua menggunakan API gratis.

**Author:** [Nama GitHub Kamu]  
**Tanggal:** 11 Oktober 2026  
**Hardware:** Legion 5 Pro 16ACH6H (Ryzen 7 5800H, RTX 3060, 64GB RAM, 500GB SSD)  
**Status:** ✅ Production Ready — Berjalan 24/7

---

## 📑 Daftar Isi

1. [Apa Itu Hermes Agent?](#1-apa-itu-hermes-agent)
2. [Kenapa WSL2 + Docker Sandbox?](#2-kenapa-wsl2--docker-sandbox)
3. [Arsitektur Akhir](#3-arsitektur-akhir)
4. [Instalasi Step-by-Step](#4-instalasi-step-by-step)
5. [Konfigurasi Model AI Gratis](#5-konfigurasi-model-ai-gratis)
6. [Setup Telegram Gateway](#6-setup-telegram-gateway)
7. [Uji Isolasi Sandbox](#7-uji-isolasi-sandbox)
8. [Troubleshooting Lengkap](#8-troubleshooting-lengkap)
9. [Pemeliharaan Rutin](#9-pemeliharaan-rutin)
10. [Pelajaran yang Didapat](#10-pelajaran-yang-didapat)

---

## 1. Apa Itu Hermes Agent?

**Hermes Agent** adalah agen AI otonom open-source dari [Nous Research](https://nousresearch.com) yang dirilis Februari 2026. Berbeda dengan chatbot biasa, Hermes adalah **"pekerja digital"** yang bisa:

- 🖥️ Menjalankan perintah terminal
- 📁 Membaca, menulis, dan mengedit file
- 🌐 Mencari informasi di web dan scraping
- 🧠 Mengingat percakapan lintas sesi (persistent memory)
- 📅 Menjalankan tugas terjadwal (cron jobs)
- 🤖 Mendelegasikan tugas ke sub-agent paralel

### Filosofi Utama

| Prinsip | Penjelasan |
| :--- | :--- |
| **Self-Hosted** | Berjalan di mesin sendiri, data tidak keluar |
| **Learning Loop** | Menciptakan "skill" baru dari pengalaman |
| **Persistent Memory** | Ingatan tidak hilang saat sesi berakhir |
| **Open Source** | Lisensi MIT, bebas dimodifikasi |

### Apakah Gratis 100%?

**Jawaban singkat: Framework-nya gratis, tapi butuh "bahan bakar" (model AI).**

- ✅ **Framework Hermes**: Gratis, open-source (MIT License)
- ⚠️ **Model AI**: Tergantung pilihan — bisa gratis (Groq, Gemini) atau berbayar (OpenAI, Anthropic)
- 💡 **Strategi hemat**: Gunakan API gratis dari Groq + Gemini dengan fallback chain

---

## 2. Kenapa WSL2 + Docker Sandbox?

### Kekhawatiran Awal

Saya sempat ragu karena mendengar isu bahwa Hermes sering membersihkan data/file sendiri tanpa konfirmasi. Kalau ini benar, file penting di Windows bisa terhapus.

### Kenapa WSL2 Saja Belum Cukup?

WSL2 memang menjalankan kernel Linux asli, tapi secara **default ia me-mount drive Windows di `/mnt/c`**. Artinya, agen masih bisa membaca dan **menulis** ke file Windows.

> **Catatan penting**: Dokumentasi WSL sendiri menyatakan bahwa distribusi Linux di WSL2 **bukanlah batas keamanan** dari host Windows.

### Solusi: Docker Container di Dalam WSL2

Dengan menjalankan Hermes di dalam **container Docker** yang terisolasi:
- Sistem file-nya **secara fisik dibatasi** oleh kernel Linux
- Meskipun model AI "ingin" menulis di luar sandbox, kernel mencegahnya
- Kita bisa **memutus akses** WSL2 ke `/mnt/c` untuk lapisan tambahan

### Lapisan Keamanan Berlapis

| Lapisan | Fungsi | Status |
| :--- | :--- | :--- |
| **WSL2 VM** | Isolasi kernel Linux dari Windows | ✅ Aktif |
| **Docker Container** | Isolasi filesystem tingkat kernel | ✅ Aktif |
| **`HERMES_WRITE_SAFE_ROOT`** | Batasi tulis ke `/workspace` saja | ✅ Aktif |
| **Approval Mode `manual`** | Konfirmasi setiap perintah berbahaya | ✅ Aktif |
| **Hardline Blocklist** | Tolak `rm -rf /`, fork bomb, dll. | ✅ Bawaan |
| **File Write Protection** | Blokir tulis ke `~/.ssh`, `~/.aws`, `.env` | ✅ Bawaan |

---

## 3. Arsitektur Akhir

```
┌─────────────────────────────────────────────────────────────┐
│                  Windows Host (Legion 5 Pro)                │
│  ┌───────────────────────────────────────────────────────┐  │
│  │           WSL2 (Ubuntu, systemd enabled)              │  │
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │           Hermes Agent (Python)                 │  │  │
│  │  │  ┌───────────────────────────────────────────┐  │  │  │
│  │  │  │      Docker Container (Sandbox)           │  │  │  │
│  │  │  │  ┌─────────────┐  ┌──────────────────┐    │  │  │  │
│  │  │  │  │ /workspace  │  │ /output          │    │  │  │  │
│  │  │  │  │ (writable)  │  │ (host-visible)   │    │  │  │  │
│  │  │  │  └─────────────┘  └──────────────────┘    │  │  │  │
│  │  │  │  ❌ /mnt/c  ❌ /mnt/d  ❌ Windows Host     │  │  │  │
│  │  │  └───────────────────────────────────────────┘  │  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │         Docker Engine (systemd service)         │  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                              │
│  🔒 Windows file system aman dari jangkauan agen            │
└─────────────────────────────────────────────────────────────┘
                              │
                              │ Telegram Bot API
                              ▼
                     ┌────────────────┐
                     │   📱 Telegram  │
                     │   (User Chat)  │
                     └────────────────┘
```

---

## 4. Instalasi Step-by-Step

### Prasyarat

| Komponen | Versi Minimum |
| :--- | :--- |
| Windows 10/11 | Build 19041+ |
| WSL2 | Sudah terinstal |
| RAM | 8 GB (16 GB direkomendasikan) |
| Storage | 10 GB untuk Hermes + Docker |
| Virtualisasi | Aktif di BIOS/UEFI |

---

### Langkah 1: Konfigurasi WSL2

Buat file `C:\Users\<NamaKamu>\.wslconfig`:

```ini
[wsl2]
memory=32GB
processors=8
swap=8GB
localhostForwarding=true

[automount]
enabled=false
```

> **Catatan**: File `.wslconfig` tidak ada secara default. Buat manual dengan Notepad (Save as type: All Files) atau PowerShell: `notepad $env:USERPROFILE\.wslconfig`

Restart WSL2 dari **PowerShell Windows** (bukan di dalam WSL):

```powershell
wsl --shutdown
```

---

### Langkah 2: Aktifkan `systemd` di WSL2

Ini **wajib** agar perintah `systemctl --user` berfungsi (diperlukan untuk mengelola gateway Hermes sebagai background service).

Edit `/etc/wsl.conf` di dalam WSL2:

```bash
sudo nano /etc/wsl.conf
```

Tambahkan:

```ini
[boot]
systemd=true
```

Restart WSL2 lagi dari PowerShell:

```powershell
wsl --shutdown
```

**Verifikasi**:

```bash
systemctl --user status
```

Jika tidak ada error `Failed to connect to user scope bus`, berarti `systemd` sudah aktif.

---

### Langkah 3: Instal Docker Engine

**Jangan gunakan Docker Desktop** — kita pakai Docker Engine langsung di dalam WSL2 agar lebih ringan dan isolasinya lebih kuat.

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker
```

> **Jika `newgrp: not found`**, install dulu:
> ```bash
> sudo apt install util-linux-extra
> ```

**Verifikasi**:

```bash
docker run hello-world
```

Jika muncul `"Hello from Docker!"`, berarti Docker sudah siap.

---

### Langkah 4: Instal Hermes Agent

```bash
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash
source ~/.bashrc
hermes --version
```

Installer akan mengunduh dan menginstal:
- Python 3.14
- Node.js
- uv, ripgrep, ffmpeg
- Dan semua dependensi lainnya

---

### Langkah 5: Setup Wizard

```bash
hermes setup
```

Pilihan konfigurasi yang saya gunakan:

| Prompt | Pilihan | Alasan |
| :--- | :--- | :--- |
| **Setup mode** | Full setup | Supaya bisa pakai API key sendiri |
| **Provider** | Google AI Studio → nanti diganti Groq | Gemini gratis tanpa kartu kredit |
| **Model** | `gemini-3.6-flash` | Paling efisien untuk hemat kuota |
| **Reasoning effort** | `medium` | Keseimbangan kualitas & hemat token |
| **Terminal backend** | **Docker / Podman** | Isolasi keamanan ⚠️ WAJIB |
| **Egress firewall** | No | Agar bisa akses API internet |
| **Messaging platforms** | Skip | Setup Telegram belakangan |
| **Tools** | Biarkan default | Web, Browser, Terminal, File, Code, Vision, Memory, Cron |
| **Browser** | Local Browser | Gratis, headless Chromium |
| **Image Generation** | Skip | Semua opsi berbayar |
| **TTS** | Microsoft Edge TTS | Gratis, tanpa API key |
| **Search Provider** | DuckDuckGo (ddgs) | Gratis, tanpa API key |
| **Telemetry** | No thanks | Privasi maksimal |

---

## 5. Konfigurasi Model AI Gratis

### Masalah: Free Tier Cepat Habis

- **Gemini free tier**: hanya ~20 request/hari, sedangkan satu giliran percakapan Hermes butuh 3–10 panggilan API → habis dalam 2–6 giliran
- **Groq `gpt-oss-120b`**: TPM (Tokens Per Minute) hanya 8.000 → konteks percakapan (~15.000 token) langsung melebihi batas

### Solusi: Fallback Chain

Strategi: **Primary Groq dengan TPM besar + Fallback Gemini**.

### Perbandingan Provider Free Tier

| Provider | Model | TPM | RPD | Cocok Untuk |
| :--- | :--- | :--- | :--- | :--- |
| **Groq** | `meta-llama/llama-4-scout-17b-16e-instruct` | **30.000** | 1.000 | **Primary** — paling tahan lama |
| Groq | `openai/gpt-oss-120b` | 8.000 | 1.000 | Kualitas tinggi, limit ketat |
| Groq | `qwen/qwen3.6-27b` | 8.000 | 1.000 | Coding, reasoning |
| Gemini | `gemini-3.6-flash` | — | 20 | **Fallback** cadangan |
| OpenRouter | `nemotron-3-super-120b:free` | 20 RPM | 50 | Fallback tambahan |

### Cara Dapatkan API Key Groq (Gratis)

1. Buka [console.groq.com/keys](https://console.groq.com/keys)
2. Daftar akun (tanpa kartu kredit)
3. Klik **"Create API Key"**
4. Beri nama (misal: "Hermes")
5. **Salin segera** — key hanya ditampilkan sekali (format: `gsk_...`)

### Konfigurasi `config.yaml`

Buka file:

```bash
nano ~/.hermes/config.yaml
```

Cari bagian `model:` dan `custom_providers:`, ubah menjadi:

```yaml
model:
  default: "meta-llama/llama-4-scout-17b-16e-instruct"
  provider: "custom:groq"
  max_tokens: 2048
  api_key: "${GROQ_API_KEY}"

custom_providers:
  - name: groq
    base_url: https://api.groq.com/openai/v1
    key_env: GROQ_API_KEY
    api_mode: chat_completions

fallback_providers:
  - provider: gemini
    model: gemini-3.6-flash
```

> **Penting**: Provider harus ditulis `custom:groq` (dengan prefix `custom:`), **bukan** hanya `groq`. Hermes hanya mengenali custom provider dengan format ini.

### Simpan API Key

Edit `.env`:

```bash
nano ~/.hermes/.env
```

Tambahkan (ganti dengan key asli):

```bash
GROQ_API_KEY=gsk_KEY_ASLI_KAMU
GOOGLE_API_KEY=AIzaSy_KEY_GEMINI_KAMU
```

> **Hati-hati**: Jangan biarkan ada **duplikat** `GROQ_API_KEY`. Hapus yang lama jika ada.

### Uji Koneksi Sebelum Digunakan

Selalu uji API key dengan `curl` sebelum menyalahkan Hermes:

```bash
curl -X POST https://api.groq.com/openai/v1/chat/completions \
  -H "Authorization: Bearer gsk_KEY_ASLI_KAMU" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "meta-llama/llama-4-scout-17b-16e-instruct",
    "messages": [{"role": "user", "content": "Hello"}]
  }'
```

Kalau muncul JSON balasan → key valid. Kalau `{"error":{"message":"Invalid API Key"}}` → key salah/kadaluarsa.

### Cara Cari dan Edit Baris di YAML

**Cara 1: Cari langsung di nano**

```bash
nano ~/.hermes/config.yaml
```

- Tekan `Ctrl + W` untuk search
- Ketik kata kunci (misal: `model:`)
- Tekan `Alt + W` untuk cari kemunculan berikutnya
- `Ctrl + C` untuk batalkan

**Cara 2: Cari nomor baris dengan grep**

```bash
grep -n "custom_providers" ~/.hermes/config.yaml
# Output: 125:custom_providers:
nano +125 ~/.hermes/config.yaml
```

**Cara 3: Lihat konteks sekitar**

```bash
grep -n -A 5 -B 2 "model:" ~/.hermes/config.yaml
```

**Pintasan nano yang berguna:**

| Tombol | Fungsi |
| :--- | :--- |
| `Ctrl + W` | Cari |
| `Alt + W` | Cari lagi |
| `Ctrl + _` | Lompat ke nomor baris |
| `Ctrl + K` | Potong baris |
| `Ctrl + U` | Paste baris |
| `Ctrl + O` | Simpan |
| `Ctrl + X` | Keluar |
| `Alt + U` | Undo |

**Verifikasi YAML valid:**

```bash
python3 -c "import yaml; yaml.safe_load(open('/home/legion/.hermes/config.yaml'))" && echo "YAML valid"
```

---

## 6. Setup Telegram Gateway

### Langkah 1: Buat Bot di BotFather

1. Buka Telegram, cari **@BotFather** (centang biru)
2. Kirim `/newbot`
3. Ikuti instruksi:
   - **Display name**: "Hermes Agent"
   - **Username**: harus diakhiri `bot` (misal: `my_hermes_bot`)
4. BotFather akan memberi **API token** (format: `123456789:ABCdef...`)
5. **Salin dan simpan** — jangan bagikan ke siapa pun

### Langkah 2: Dapatkan User ID Telegram

1. Cari **@userinfobot** atau **@get_id_bot**
2. Kirim pesan apa saja
3. Bot akan membalas dengan **ID numerik** (misal: `449676717`)

### Langkah 3: Konfigurasi Gateway

```bash
hermes gateway setup
```

Pilihan:
- **Telegram** → **Manual** (paste token)
- **Bot token**: tempel dari BotFather
- **Allowed user IDs**: masukkan User ID numerik (hanya kamu yang bisa akses)
- **Home channel**: `Y` (gunakan User ID yang sama)
- **Restart gateway**: `Y`

### Langkah 4: Verifikasi

```bash
hermes gateway status
```

Yang diharapkan:

```
● hermes-gateway.service
     Active: active (running) since ...
   Main PID: 53378 (hermes)
```

### Langkah 5: Kirim Pesan Pertama ke Bot

⚠️ **WAJIB**: Bot Telegram **tidak bisa memulai percakapan** sendiri. Kamu harus kirim pesan dulu.

1. Buka Telegram, cari bot kamu (username berakhiran `bot`)
2. Tekan tombol **START** atau kirim `/start`
3. Bot akan membalas!

Jika tidak dikirim lebih dulu, akan muncul error `Chat not found` di log gateway.

---

## 7. Uji Isolasi Sandbox

### Tes 1: Fungsi Dasar (Harus Berhasil)

**Prompt di Telegram/CLI**:

> _"Tolong buat file bernama `test_sandbox.txt` di `/workspace`, isinya 'Hermes berjalan di dalam Docker'. Lalu baca kembali dan tunjukkan isinya."_

**Hasil yang diharapkan**: ✅ File berhasil dibuat dan dibaca dalam ~2 detik.

### Tes 2: Isolasi Windows (Harus Gagal)

**Prompt**:

> _"Coba cek isi direktori `/mnt/c/Users/LEGION/Documents`."_

**Hasil yang diharapkan**:

```
⚠️ Path not found: /mnt/c/Users/LEGION/Documents
Direktori /mnt/c/Users/LEGION/Documents tidak ditemukan atau tidak dapat diakses.
Lingkungan eksekusi Hermes saat ini berjalan terisolasi di dalam kontainer Docker sandbox, 
sehingga folder sistem Windows Host (/mnt/c/...) tidak di-mount ke dalam kontainer ini.
```

✅ **Jika ini muncul, berarti sandbox bekerja sempurna!** Windows file system kamu 100% aman.

---

## 8. Troubleshooting Lengkap

Selama proses setup, saya menemukan beberapa error. Berikut dokumentasi lengkapnya:

### 🔴 Error 1: `Failed to connect to user scope bus via local transport`

**Penyebab**: WSL2 belum mengaktifkan `systemd` sebagai init system.

**Solusi**: Edit `/etc/wsl.conf`, tambahkan:
```ini
[boot]
systemd=true
```
Lalu `wsl --shutdown` dari PowerShell dan buka WSL lagi.

---

### 🔴 Error 2: `Docker is installed but the Docker daemon is not running (exit code 1)`

**Penyebab**: Gateway Hermes berjalan sebagai `systemd user service` yang tidak mewarisi environment shell interaktif (tidak tahu lokasi Docker socket).

**Solusi**: Buat override untuk gateway:

```bash
mkdir -p ~/.config/systemd/user/hermes-gateway.service.d
nano ~/.config/systemd/user/hermes-gateway.service.d/override.conf
```

Isi:

```ini
[Service]
Environment="DOCKER_HOST=unix:///var/run/docker.sock"
```

Lalu:

```bash
systemctl --user daemon-reload
systemctl --user restart hermes-gateway
```

**Alternatif paling mudah**: Reboot laptop. Setelah reboot, systemd user manager akan dimulai ulang dengan grup `docker` yang benar.

---

### 🔴 Error 3: `Unknown provider 'groq'`

**Penyebab**: Hermes tidak mengenali `groq` sebagai provider bawaan.

**Solusi**: Daftarkan Groq sebagai **custom provider** dengan prefix `custom:`:

```yaml
model:
  provider: "custom:groq"  # ⚠️ BUKAN hanya "groq"

custom_providers:
  - name: groq
    base_url: https://api.groq.com/openai/v1
    key_env: GROQ_API_KEY
    api_mode: chat_completions
```

Restart gateway setelah perubahan:
```bash
systemctl --user restart hermes-gateway
```

---

### 🔴 Error 4: `HTTP 400: reasoning_effort must be one of low, medium, or high`

**Penyebab**: Model `openai/gpt-oss-120b` di Groq hanya menerima nilai `low`, `medium`, atau `high`. Hermes secara default mengirim nilai lain (mungkin `minimal`).

**Solusi**: Tambahkan `extra_body` di custom provider:

```yaml
custom_providers:
  - name: groq
    base_url: https://api.groq.com/openai/v1
    key_env: GROQ_API_KEY
    api_mode: chat_completions
    extra_body:
      reasoning_effort: "low"
```

---

### 🔴 Error 5: `HTTP 413: Request too large` (TPM Limit)

**Penyebab**: Konteks percakapan melebihi TPM limit Groq (8.000 untuk `gpt-oss-120b`).

**Solusi** (pilih salah satu):
- **Mulai sesi baru**: `/new`
- **Kompres konteks**: `/compress` atau `/compress here 2`
- **Ganti ke model TPM besar**: `meta-llama/llama-4-scout-17b-16e-instruct` (30.000 TPM)
- **Tunggu 1 menit** — kuota TPM akan terisi kembali

---

### 🔴 Error 6: `Chat not found` di Telegram

**Penyebab**: Bot Telegram tidak bisa memulai percakapan. User harus inisiasi dulu.

**Solusi**: Kirim pesan `/start` ke bot Telegram kamu terlebih dahulu.

---

### 🔴 Error 7: `Gemini HTTP 429 (RESOURCE_EXHAUSTED)`

**Penyebab**: Gemini free tier hanya ~20 request/hari, sedangkan satu giliran Hermes butuh 3–10 request.

**Solusi**:
- Ganti ke Groq sebagai primary (`meta-llama/llama-4-scout-17b-16e-instruct`)
- Pasang Gemini sebagai fallback dengan `fallback_providers`
- Tunggu kuota reset (biasanya per hari)

---

### 🔴 Error 8: `Invalid API Key` dari Groq

**Penyebab paling umum**:
1. Menggunakan placeholder seperti `gsk_KEYBARU` secara literal
2. Key sudah dicabut (revoked) karena terekspos
3. Ada duplikat key di `.env` (key lama menimpa key baru)
4. Salah copy (ada spasi atau karakter tambahan)

**Solusi**:
- Uji dengan `curl` langsung (ganti key asli)
- Cek `.env`: `cat ~/.hermes/.env | grep GROQ`
- Pastikan hanya ada **satu baris** `GROQ_API_KEY` yang aktif
- Buat key baru di [console.groq.com/keys](https://console.groq.com/keys)

---

### 🔴 Error 9: `model llama-3.3-70b-versatile does not exist`

**Penyebab**: Model `llama-3.3-70b-versatile` sudah **deprecated** sejak 16 Agustus 2026.

**Solusi**: Ganti ke model yang aktif:
- `openai/gpt-oss-120b` (kualitas tinggi, TPM kecil)
- `meta-llama/llama-4-scout-17b-16e-instruct` (TPM besar) ← **Rekomendasi**
- `qwen/qwen3.6-27b` (untuk coding)

---

## 9. Pemeliharaan Rutin

### Storage Check

Cek penggunaan disk WSL2:

```bash
df -h ~
docker system df
du -sh ~/.hermes/*
```

### Pembersihan Otomatis (Cron Job)

Hermes Agent punya **bug kebocoran disk**: direktori sandbox tidak terhapus otomatis, sehingga menumpuk seiring waktu.

**Solusi**: Pasang cron job mingguan:

```bash
crontab -e
```

Pilih editor (`1` untuk nano), tambahkan baris:

```cron
0 3 * * 0 docker container prune -f && docker image prune -f && docker builder prune -f && find ~/.hermes/sandboxes/docker -maxdepth 1 -type d -mtime +7 -exec rm -rf {} \;
```

Berjalan **setiap Minggu jam 03:00 pagi**.

### Batasi Log Docker

Edit `/etc/docker/daemon.json`:

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

Restart Docker:

```bash
sudo systemctl restart docker
```

### Auto-Start Setelah Reboot

```bash
sudo loginctl enable-linger $USER
```

Ini memastikan gateway Hermes tetap hidup setelah logout/reboot.

---

## 10. Pelajaran yang Didapat

### 🎓 Yang Saya Pelajari

1. **WSL2 bukan sandbox sejati** — perlu lapisan tambahan (Docker) untuk isolasi total dari Windows
2. **`systemd` wajib diaktifkan** — tanpa itu, semua perintah `systemctl --user` akan gagal
3. **Custom provider butuh prefix `custom:`** — kesalahan umum yang bikin bingung berjam-jam
4. **API key gratis cepat habis** — strategi fallback chain sangat penting
5. **Model Groq sering di-deprecate** — selalu cek daftar model terbaru di [console.groq.com](https://console.groq.com)
6. **Selalu uji `curl` sebelum menyalahkan Hermes** — 90% error API berasal dari key yang salah/duplikat
7. **Bot Telegram tidak bisa inisiasi chat** — user harus kirim `/start` dulu
8. **Debugging itu seni** — baca log pelan-pelan (`journalctl --user -u hermes-gateway -n 50 --no-pager`), jangan panik

### 💡 Tips untuk Pembaca yang Ingin Mencoba

1. **Mulai dari WSL2 yang bersih** — jangan campur dengan distro yang sudah banyak modifikasi
2. **Aktifkan `systemd` lebih awal** — sebelum install Hermes
3. **Gunakan Docker Engine, bukan Docker Desktop** — lebih ringan dan isolasi lebih kuat
4. **Siapkan 2 API key gratis** (Groq + Gemini) untuk fallback
5. **Selalu uji `curl` dulu** sebelum konfigurasi di Hermes
6. **Backup `~/.hermes/config.yaml`** sebelum edit besar
7. **Gunakan `/new` dan `/compress`** secara rutin untuk menjaga konteks tetap ringan
8. **Jangan bagikan API key** — revoke segera jika terlanjur terekspos

### 🚀 Ide Eksperimen Lanjutan

- [ ] Set up **cron job AI** untuk merangkum berita setiap pagi
- [ ] Aktifkan **Speech-to-Text** dengan Whisper lokal
- [ ] Hubungkan ke **Discord** atau **Slack** sebagai platform tambahan
- [ ] Buat **custom skill** dengan `/learn` dari dokumentasi favorit
- [ ] Eksperimen dengan **sub-agent delegation** untuk tugas kompleks
- [ ] Pasang **Ollama** untuk model 100% offline di RTX 3060

---

## 📊 Statistik Akhir

| Komponen | Penggunaan |
| :--- | :--- |
| **RAM (WSL2)** | ~380 MB (gateway idle) |
| **Disk (Docker images)** | ~4.4 GB |
| **Disk (`~/.hermes`)** | ~3.1 GB |
| **Disk (total WSL2)** | ~11 GB dari 1007 GB |
| **CPU (idle)** | ~0.1% |
| **Model AI** | Groq Llama 4 Scout + Gemini Flash (fallback) |
| **Uptime** | 24/7 (systemd service) |

---

## 🛡️ Catatan Keamanan Akhir

1. ✅ **Isolasi terbukti**: Agen tidak dapat mengakses `/mnt/c`
2. ✅ **Approval manual**: Setiap perintah berbahaya harus dikonfirmasi
3. ✅ **File write terbatas**: Hanya bisa menulis di `/workspace`
4. ✅ **Hardline blocklist**: `rm -rf /` ditolak mentah-mentah
5. ✅ **API keys aman**: Tersimpan di `~/.hermes/.env`
6. ✅ **Telemetri dimatikan**: Tidak ada data keluar

---

## 📚 Referensi

- [Hermes Agent Documentation](https://hermes-agent.nousresearch.com/docs)
- [Nous Research GitHub](https://github.com/NousResearch/hermes-agent)
- [Groq Console](https://console.groq.com)
- [Google AI Studio](https://aistudio.google.com)
- [Docker Engine on WSL2](https://docs.docker.com/engine/install/ubuntu/)

---

## 📝 Lisensi & Kredit

- **Hermes Agent**: [Lisensi MIT](https://github.com/NousResearch/hermes-agent/blob/main/LICENSE) oleh Nous Research
- **Dokumentasi ini**: Bebas kamu gunakan untuk keperluan portofolio, pembelajaran, atau referensi pribadi

---

<div align="center">

**⭐ Kalau catatan ini membantu, jangan lupa kasih bintang di repo! ⭐**

_Made with ❤️ dan banyak trial-error oleh [Nama GitHub Kamu]_

_"Bikin AI agent yang aman itu bukan soal satu lapisan, tapi soal berpikir berlapis."_

</div>
