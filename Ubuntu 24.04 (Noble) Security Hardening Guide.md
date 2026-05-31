# Ubuntu 24.04 (Noble) Security Hardening Guide
# Created and Curated by SafetyDanc3

A complete, step-by-step hardening guide for a personal Ubuntu device, ordered roughly by impact-to-effort. You don't have to do all of it in one sitting — work top to bottom and you'll be more secure at every step.

**The authoritative reference for all of this is the [CIS Ubuntu Linux Benchmark](https://www.cisecurity.org/benchmark/ubuntu_linux). This guide is the practical, plain-language version of that standard for a personal machine.**

---

## Read this first — things that can lock you out

A few steps below can brick your access if done carelessly. Pay attention to the **CAUTION** callouts on:

- **SSH key auth** — set up and test your key *before* disabling password login, or you'll be locked out of remote access.
- **USBGuard** — generate the policy *with your keyboard and mouse plugged in*, or your input devices get blocked.
- **GRUB password** — write it down; if you forget it you can't edit boot parameters.
- **Firewall** — if you use SSH, allow it *before* enabling the firewall.

Take a Timeshift snapshot (Phase 9) before the more invasive changes so you can roll back.

---

## Priority order (if you only do a few things)

1. Updates + automatic security updates (Phase 1)
2. Firewall (Phase 4)
3. Full disk encryption (Phase 2 — ideally at install)
4. Strong auth + password policy (Phase 3)
5. Run a Lynis audit to score yourself (Phase 8)

Everything else is depth on top of that foundation.

---

# Phase 1 — Patching

The single highest-impact control. Most real-world compromise is unpatched software.

**Update now:**
```bash
sudo apt update && sudo apt full-upgrade -y
```

**Turn on automatic security updates:**
```bash
sudo apt install unattended-upgrades apt-listchanges -y
sudo dpkg-reconfigure -plow unattended-upgrades
```

Then tune the config:
```bash
sudo nano /etc/apt/apt.conf.d/50unattended-upgrades
```
Useful lines to uncomment/set:
```
Unattended-Upgrade::Remove-Unused-Dependencies "true";
Unattended-Upgrade::Automatic-Reboot "false";
Unattended-Upgrade::Automatic-Reboot-Time "03:00";
```
(Set auto-reboot to `true` only if unattended reboots are acceptable.)

---

# Phase 2 — Full Disk Encryption (LUKS)

Protects your data if the laptop is lost or stolen. **This is best done at install time** — the Ubuntu installer has an "Encrypt the new Ubuntu installation" checkbox (LUKS).

- **Already installed without encryption?** You can't cleanly encrypt the root filesystem after the fact without a reinstall. Your options:
  - Back up your data, reinstall with encryption enabled (cleanest, recommended).
  - Or encrypt secondary drives / a data partition with LUKS:
    ```bash
    sudo cryptsetup luksFormat /dev/sdX
    sudo cryptsetup open /dev/sdX securedata
    sudo mkfs.ext4 /dev/mapper/securedata
    ```

Pair this with a BIOS/UEFI password (Phase 7) so the encryption can't be trivially bypassed at boot.

---

# Phase 3 — Accounts & Authentication

**Ubuntu already disables direct root login.** Belt-and-suspenders:
```bash
sudo passwd -l root
```

**Enforce strong passwords:**
```bash
sudo apt install libpam-pwquality -y
sudo nano /etc/security/pwquality.conf
```
Reasonable settings:
```
minlen = 14
dcredit = -1
ucredit = -1
lcredit = -1
ocredit = -1
retry = 3
```
(Requires at least 14 chars with a digit, upper, lower, and special char.)

**Lock accounts after failed attempts (pam_faillock):**
Edit `/etc/security/faillock.conf` and set:
```
deny = 5
unlock_time = 900
```

**Set password aging on your account:**
```bash
sudo chage -M 365 -W 14 yourusername
```

---

# Phase 4 — Firewall (UFW)

```bash
sudo apt install ufw -y
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw logging on
```

> **CAUTION:** If you use SSH, allow it before enabling:
> ```bash
> sudo ufw limit ssh
> ```
> (`limit` rate-limits connection attempts — better than plain `allow`.)

Then enable and verify:
```bash
sudo ufw enable
sudo ufw status verbose
```

Only open ports you actually need.

---

# Phase 5 — SSH Hardening *(skip if you don't run SSH)*

**Generate a strong key (on the machine you connect *from*):**
```bash
ssh-keygen -t ed25519 -C "your-label"
ssh-copy-id youruser@yourserver
```

> **CAUTION:** Confirm you can log in with the key *before* disabling passwords.

Edit `/etc/ssh/sshd_config`:
```
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
MaxAuthTries 3
AllowUsers youruser
X11Forwarding no
```
(Optionally change `Port` to a non-standard number — minor obscurity benefit.)

Apply:
```bash
sudo systemctl restart ssh
```

---

# Phase 6 — Intrusion Prevention (Fail2ban)

Bans IPs that brute-force your services.
```bash
sudo apt install fail2ban -y
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo nano /etc/fail2ban/jail.local
```
In the `[sshd]` section (and any others you expose):
```
enabled = true
maxretry = 3
findtime = 600
bantime = 3600
```
Start it:
```bash
sudo systemctl enable --now fail2ban
sudo fail2ban-client status sshd
```

---

# Phase 7 — System & Kernel Hardening

### AppArmor (mandatory access control)
Ubuntu ships this enabled. Verify and extend:
```bash
sudo aa-status
sudo apt install apparmor-profiles apparmor-utils -y
```
Put profiles into enforce mode as needed with `sudo aa-enforce /etc/apparmor.d/<profile>`.

### Kernel & network sysctl hardening
```bash
sudo nano /etc/sysctl.d/99-hardening.conf
```
Paste:
```
# Spoofing / source-route / redirect protection
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
net.ipv4.conf.all.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.all.accept_source_route = 0
net.ipv6.conf.all.accept_source_route = 0

# SYN flood + logging
net.ipv4.tcp_syncookies = 1
net.ipv4.conf.all.log_martians = 1
net.ipv4.icmp_echo_ignore_broadcasts = 1

# Kernel hardening
kernel.randomize_va_space = 2
kernel.kptr_restrict = 2
kernel.dmesg_restrict = 1
kernel.yama.ptrace_scope = 1
fs.protected_hardlinks = 1
fs.protected_symlinks = 1
```
Apply:
```bash
sudo sysctl --system
```

### Disable services you don't use
```bash
sudo systemctl list-unit-files --type=service --state=enabled
```
Common candidates on a personal machine: `cups` (no printer), `avahi-daemon`, `bluetooth` (if unused). Disable with:
```bash
sudo systemctl disable --now <service>
```

### Boot & physical security
- Set a **BIOS/UEFI password** and disable booting from USB/external media.
- Keep **Secure Boot** enabled.
- **GRUB password** (prevents editing kernel boot params at the menu):
  ```bash
  grub-mkpasswd-pbkdf2
  ```
  Add the resulting hash to `/etc/grub.d/40_custom`, then `sudo update-grub`.

  > **CAUTION:** Save this password somewhere safe.

---

# Phase 8 — Detection, Auditing & Monitoring

### Lynis — audit & score your hardening
This is your measuring stick. Run it before and after changes.
```bash
sudo apt install lynis -y
sudo lynis audit system
```
It produces a **hardening index** and a list of specific suggestions.

### auditd — system call / event auditing
```bash
sudo apt install auditd audispd-plugins -y
sudo systemctl enable --now auditd
```

### AIDE — file integrity monitoring
Detects unexpected changes to system files.
```bash
sudo apt install aide -y
sudo aideinit
sudo mv /var/lib/aide/aide.db.new /var/lib/aide/aide.db
```
Schedule regular checks (e.g. daily cron):
```bash
sudo aide --check
```

### Rootkit & malware scanning
```bash
sudo apt install rkhunter chkrootkit clamav clamav-daemon -y

# rkhunter — set baseline then scan
sudo rkhunter --update
sudo rkhunter --propupd
sudo rkhunter --check

# chkrootkit
sudo chkrootkit

# ClamAV
sudo freshclam
clamscan -r /home
```

---

# Phase 9 — Backups (3-2-1 Rule)

Security includes recoverability. **3** copies, on **2** different media, **1** offsite.

- **System snapshots — Timeshift** (great for rolling back bad changes):
  ```bash
  sudo apt install timeshift -y
  ```
- **Encrypted data backups — restic or Borg:**
  ```bash
  sudo apt install restic -y
  ```
  Both encrypt and deduplicate. Keep at least one copy offsite/cloud.

---

# Phase 10 — Application Sandboxing

- **Firejail** for isolating individual apps:
  ```bash
  sudo apt install firejail -y
  firejail firefox
  ```
- **Flatpak** apps run sandboxed by default — prefer Flatpak/Snap over raw `.deb` for untrusted or internet-facing apps.
- Inspect/limit Flatpak permissions with **Flatseal**.

---

# Phase 11 — USB Protection (USBGuard)

Blocks unauthorized USB devices (BadUSB / rubber ducky style attacks).

> **CAUTION:** Generate the policy *while your keyboard and mouse are connected*, or you'll block your own input devices.

```bash
sudo apt install usbguard -y
sudo usbguard generate-policy | sudo tee /etc/usbguard/rules.conf
sudo systemctl enable --now usbguard
```
New devices are blocked until you explicitly allow them with `usbguard allow-device`.

---

# Phase 12 — Privacy & Network Layer

- **DNS:** Use encrypted DNS (DNS-over-HTTPS/TLS) or point at your own Pi-hole. Configure in `systemd-resolved` (`/etc/systemd/resolved.conf`).
- **VPN:** Use a reputable VPN for untrusted networks.
- **Browser:** Harden Firefox with the **arkenfox user.js**, or use a privacy-focused browser. Add uBlock Origin.
- **Password manager:** Keep everything in KeePassXC (or similar) with a strong master password + key file.
- **Reduce telemetry / crash reporting:**
  ```bash
  sudo systemctl disable --now apport.service
  sudo systemctl disable --now whoopsie.service
  sudo apt purge popularity-contest -y
  ```

---

# Phase 13 — Desktop & Session

- Set screen to **auto-lock** on a short timeout (Settings → Privacy → Screen Lock).
- Require password on wake.
- Don't run as a daily admin where avoidable — use a standard account for routine use and `sudo` for admin tasks.

---

# Ongoing Maintenance Checklist

| Frequency | Task |
|-----------|------|
| Daily (automated) | Unattended security updates, fail2ban, AIDE check |
| Weekly | Review `sudo lynis audit system` if you've made changes; run ClamAV scan |
| Monthly | `sudo rkhunter --check`; review enabled services; check backups restore correctly |
| Quarterly | Re-run full Lynis audit and chase the hardening index up; review user accounts |
| Always | Patch promptly, verify before disabling auth methods, keep offsite backups |

---

## Where to go deeper

- **CIS Ubuntu Linux Benchmark** — the formal standard; Lynis maps closely to it.
- **OpenSCAP / `oscap`** — automated compliance scanning against the CIS profile if you want to formalize it.
- **Ubuntu Security Guide (USG)** — Canonical's tool for applying CIS/DISA-STIG profiles (requires Ubuntu Pro for some profiles).

Run Lynis first, fix the high-priority warnings it flags, and re-run it — that loop is the fastest path to a measurably hardened machine.
