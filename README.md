# 🧠 Today I Learned (TIL)

> Log pembelajaran teknis harian, troubleshooting jaringan, dan konfigurasi *environment* kerja.

---

## 🛠️ Ringkasan Catatan Teknis

### 1. Packet Capture di WSL2 (`tshark`)
* **Masalah:** Error *Promiscuous mode not supported* atau capture gantung (0 packets captured).
* **Penyebab:** Interface `any` tidak mendukung promiscuous mode di WSL2, dan traffic DNS sering tertahan di cache/loopback.
* **Solusi:** Gunakan flag `-p` (*disable promiscuous mode*) pada interface `eth0`:
  ```bash
  # Capture ICMP (ping)
  tshark -i eth0 -p -f "icmp" -c 4

  # Capture DNS Traffic ke file pcap
  tshark -i eth0 -p -f "udp port 53" -w dns.pcap -c 5
