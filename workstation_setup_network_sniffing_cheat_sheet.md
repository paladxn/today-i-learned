# 🛠️ High-Productivity Workstation & Network Engineering Cheat Sheet
> **Environment:** WSL2 (Ubuntu) + MobaXterm | **Focus:** CLI Ergonomics, Network Analysis, & Troubleshooting

Documented on: **October 4, 2026**  
Author: *Legion*

---

## 📌 Table of Contents
1. [Environment Overview](#1-environment-overview)
2. [Shell Productivity (Fish & Zsh)](#2-shell-productivity-fish--zsh)
3. [Network Packet Capture with `tshark` in WSL2](#3-network-packet-capture-with-tshark-in-wsl2)
4. [MobaXterm Local Terminal vs. WSL2 vs. Remote VMs](#4-mobaxterm-local-terminal-vs-wsl2-vs-remote-vms)
5. [Useful Diagnostics & Maintenance Commands](#5-useful-diagnostics--maintenance-commands)
6. [🚀 Pro-Tips for Future Growth](#6--pro-tips-for-future-growth)

---

## 1. Environment Overview

To achieve an enterprise network CLI experience (similar to Huawei VRP or Cisco IOS) inside Windows, we leverage a hybrid architecture:

* **MobaXterm:** Advanced terminal emulator and X11 server for Windows.
* **WSL2 (Ubuntu):** Native Linux kernel environment running inside Windows.
* **Cygwin (Local Terminal):** MobaXterm's built-in lightweight Linux-like environment for quick Windows-side commands.

---

## 2. Shell Productivity (Fish & Zsh)

### One-Liner Installation & Setup
Run the following in WSL2 to install core tools and Zsh autosuggestions:

```bash
sudo apt update && sudo apt install -y fish zsh fzf bat ripgrep sshpass git && git clone https://github.com/zsh-users/zsh-autosuggestions ~/.zsh/zsh-autosuggestions && echo "source ~/.zsh/zsh-autosuggestions/zsh-autosuggestions.zsh" >> ~/.zshrc
```

### Choosing Your Shell Experience
* **Fish Shell (Recommended for Cisco/Huawei CLI feel out-of-the-box):**
  * Type `fish` to start.
  * Auto-completes commands automatically with faint grey text. Press **Right Arrow ($\rightarrow$)** to accept.
  * Make Fish default: `chsh -s $(which fish)`
* **Zsh Shell (Customizable with plugins):**
  * Type `zsh` to start.

---

## 3. Network Packet Capture with `tshark` in WSL2

Capturing raw packets in WSL2 requires specific permissions and flags due to network virtualization constraints.

### A. One-Time Privilege Configuration
To run `tshark` / `dumpcap` without `sudo` (preventing permission errors when saving files):

```bash
# 1. Allow non-root packet capture
sudo dpkg-reconfigure wireshark-common   # Select <Yes>

# 2. Assign capabilities to dumpcap
sudo setcap cap_net_raw,cap_net_admin=eip /usr/bin/dumpcap

# 3. Add user to the wireshark group
sudo usermod -aG wireshark $USER
```

> ⚠️ **CRITICAL STEP:** Restart WSL2 from **Windows PowerShell/CMD** for group changes to apply:
> ```powershell
> wsl --shutdown
> ```

### B. WSL2 Promiscuous Mode Limitation & Solution
WSL2 virtual interfaces do not support promiscuous mode on pseudo-devices (`any`). Always specify the interface (`eth0`) and pass the **`-p`** flag to disable promiscuous mode.

#### Practice 1: Capture ICMP Ping Live
```bash
# Terminal Tab 1 (Listener)
tshark -i eth0 -p -f "icmp" -c 4

# Terminal Tab 2 (Trigger)
ping 8.8.8.8 -c 2
```

#### Practice 2: Capture to File & Analyze
```bash
# 1. Record traffic to PCAP file
tshark -i eth0 -p -f "icmp" -w ping_test.pcap -c 4

# 2. Read PCAP file contents
tshark -r ping_test.pcap
```

#### Practice 3: Capture DNS Traffic
```bash
# 1. Record DNS traffic
tshark -i eth0 -p -f "udp port 53" -w dns_traffic.pcap -c 5

# 2. Trigger DNS query in Tab 2
dig google.com

# 3. Read PCAP file
tshark -r dns_traffic.pcap
```

---

## 4. MobaXterm Local Terminal vs. WSL2 vs. Remote VMs

Understanding where commands execute is vital for shell configurations:

| Environment | Architecture | Package Manager | Shell Autosuggestion Location |
| :--- | :--- | :--- | :--- |
| **WSL2 (Ubuntu)** | Native Linux VM | `apt` / `dpkg` | Installed inside WSL2 |
| **MobaXterm Local** | Cygwin Emulation | `apt` (Cygwin) / `cygcheck` | Installed in MobaXterm local environment |
| **Remote VM (VirtualBox/SSH)** | External Linux Server | Server's `apt` | **Must be installed directly on the Remote VM!** |

> 💡 **Key Concept:** MobaXterm acts only as a pen-and-display screen (terminal emulator). When SSH-ing into a remote VirtualBox VM, shell features (like Fish/Zsh auto-completion) must be installed on that remote VM itself.

### Fix Cygwin Missing Tools in MobaXterm Local Terminal
If `apt install` throws `cygstart: command not found`:
```bash
# Do NOT use sudo in MobaXterm Local Terminal
apt install cygutils
apt install xterm
```

---

## 5. Useful Diagnostics & Maintenance Commands

### List Installed Packages
* **WSL2 (Ubuntu):**
  * `dpkg -l` (List all packages)
  * `dpkg -l | grep <name>` (Search for installed package)
  * `dpkg -L <package_name>` (View files created by package)
* **MobaXterm Local (Cygwin):**
  * `apt list`
  * `cygcheck -c`
  * `cygcheck -l <package_name>`

### Clean Up Failed `apt` Installs (WSL2)
```bash
sudo dpkg --configure -a
sudo apt --fix-broken install
sudo apt autoclean
sudo apt clean
sudo apt autoremove -y
sudo apt update
```

---

## 6. 🚀 Pro-Tips for Future Growth

1. **Set Up Shell Aliases for Faster Workflows:**
   Add these shortcuts to your `~/.config/fish/config.fish` or `~/.zshrc`:
   ```bash
   alias tsniff="tshark -i eth0 -p"
   alias myip="ip -4 addr show eth0 | grep inet"
   ```

2. **Automate Remote VM Management via SSH Config:**
   Create or edit `~/.ssh/config` to connect to VirtualBox VMs with simple names instead of remembering IPs:
   ```text
   Host lab-vm
       HostName 192.168.56.101
       User ubuntu
       IdentityFile ~/.ssh/id_rsa
   ```
   Then simply type: `ssh lab-vm`.

3. **Master Session Persistence with `tmux`:**
   When running long packet captures or Ansible playbooks, start a `tmux` session (`tmux new -s work`). If MobaXterm closes accidentally, your processes keep running on the background VM!

4. **Prepare for Ansible Network Automation:**
   Install Ansible inside WSL2 to manage multiple network routers or VMs from one terminal:
   ```bash
   sudo apt install -y software-properties-common
   sudo add-apt-repository --yes --update ppa:ansible/ansible
   sudo apt install -y ansible