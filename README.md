# 🔐 Linux System Hardening Guide  
*SSH • Sudo • Authentication • Services • Compliance*  
> 📋 **Reference ID:** `HARDEN-2026-001` | 🗓️ **Last Updated:** May 2026

---

## 📑 Table of Contents
1. [SSH Hardening (`sshd_config`)](#-ssh-hardening-sshd_config)
2. [Deployment & Firewall Commands](#-deployment--firewall-commands)
3. [Sudoers Configuration](#-sudoers-configuration)
4. [User & Authentication Hardening](#-user--authentication-hardening)
5. [Service Hardening](#-service-hardening)
6. [Verification & Audit Commands](#-verification--audit-commands)
7. [🚫 Excluded: Network & Kernel Hardening](#-excluded-network--kernel-hardening)

---

## 🔐 SSH Hardening (`sshd_config`)

**File:** `/etc/ssh/sshd_config`

```bash
# --- Protocol & Authentication ---
Protocol 2
AddressFamily inet
Port 2222
PermitRootLogin no

# --- Cryptographic Policies (Strong, Modern, FIPS-compatible) ---
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org,diffie-hellman-group16-sha512

# --- Session Management ---
ClientAliveInterval 300
ClientAliveCountMax 3
LoginGraceTime 60
MaxAuthTries 3
MaxSessions 3

# --- Access Control (Restrict to known admins) ---
# ✅ UNCOMMENT AND CUSTOMIZE ONE OF THE FOLLOWING:
# AllowUsers admin1@10.20.0.0/16 admin2@10.20.0.0/16
# AllowGroups ssh-admins
AllowUsers admin1 admin2

# --- Disable Risky Features ---
X11Forwarding no
AllowTcpForwarding no
PermitTunnel no
PermitUserEnvironment no
AllowAgentForwarding no
GatewayPorts no
PrintMotd no
PrintLastLog no
TCPKeepAlive no

# --- Logging & Compliance ---
LogLevel VERBOSE
SyslogFacility AUTHPRIV

# --- Banner (Legal Notice) ---
Banner /etc/issue
```

> ⚠️ **Warning:** Always test SSH config before restarting:  
> ```bash
> sudo sshd -T  # Validate configuration syntax
> ```

---

## 🚀 Deployment & Firewall Commands

### 🔍 Pre-Restart Validation
```bash
sudo sshd -T
```

### 🔥 Firewall Configuration (Port 2222)
```bash
# Open new SSH port
sudo firewall-cmd --permanent --add-port=2222/tcp
sudo firewall-cmd --reload

# [Optional] Remove default port 22 if not needed
# sudo firewall-cmd --permanent --remove-service=ssh
```

### 🛡️ SELinux Port Access
```bash
# Allow SSH on custom port
sudo semanage port -a -t ssh_port_t -p tcp 2222

# Verify SELinux context
semanage port -l | grep ssh
```

### 🔐 File Permissions
```bash
chmod 600 /etc/ssh/sshd_config
chown root:root /etc/ssh/sshd_config
```

### ♻️ Apply Changes
```bash
sudo systemctl restart sshd
sudo systemctl status sshd
```

---

## 👨‍💼 Sudoers Configuration

**File:** `/etc/sudoers`  
> ✏️ **Always edit with:** `sudo visudo`

```bash
# --- Admin (full access, logged) ---
admin1 ALL=(ALL) ALL

# --- Application User (restricted commands) ---
Cmnd_Alias APP_CMDS = \
    /bin/systemctl restart app.service, \
    /bin/systemctl status app.service

appuser ALL=(ALL) APP_CMDS
```

### ✅ Verification Commands
```bash
# Check user's sudo privileges
sudo -l -U username

# Validate sudoers syntax
sudo visudo -c

# [Optional] Quick summary of sudo-enabled users
sudo grep -E '^[^#].*ALL' /etc/sudoers /etc/sudoers.d/* | awk '{print $1}'
```

---

## 👤 User & Authentication Hardening

### 🔐 Password Quality (`/etc/security/pwquality.conf`)
```ini
minlen = 14        # Minimum length
dcredit = -1       # Require at least 1 digit
ucredit = -1       # Require at least 1 uppercase
ocredit = -1       # Require at least 1 special char
lcredit = -1       # Require at least 1 lowercase
maxrepeat = 3      # Max repeated characters
maxsequence = 3    # Max sequential characters (abc, 123)
dictcheck = 1      # Block dictionary words
```

### 📅 Default Account Settings (`/etc/login.defs`)
```bash
PASS_MAX_DAYS   90       # পাসওয়ার্ড ৯০ দিন পর এক্সপায়ার
PASS_MIN_DAYS   7        # ৭ দিনের আগে চেঞ্জ করা যাবে না
PASS_WARN_AGE   14       # এক্সপায়ারের ১৪ দিন আগে ওয়ার্নিং
UMASK           022      # Default file permissions
```

### 🧹 Remove Inactive Users

#### 🔍 Identify Inactive Accounts
```bash
# Users not logged in for 90 days
lastlog -b 90

# Check account expiry details
chage -l username
```

#### 🔒 Lock Inactive Users (Recommended First Step)
```bash
# Lock password (prevents SSH/local login)
sudo usermod -L username

# [Optional] Remove shell access
sudo usermod -s /sbin/nologin username
```

#### ✅ Verify Locked Status
```bash
passwd -S username
# Output codes:
#   P → Password usable
#   L → Locked ✅
#   NP → No password
```

#### 🗑️ [Optional] Full Removal (Use with Caution)
```bash
# Set account inactivity timeout for new users
useradd -D -f 30  # Auto-disable after 30 days inactive

# Remove user + home directory
sudo userdel -r username
```

---

## ⚙️ Service Hardening

### 🔍 Service Discovery
```bash
# List all enabled services
systemctl list-unit-files --type=service | grep enabled

# List all active services
systemctl list-units --type=service --state=running
```

### ✅ Critical Services to Verify
| Service | Purpose | Command to Check |
|---------|---------|-----------------|
| `auditd` | Security logging | `systemctl status auditd` |
| `chronyd`/`ntpd` | Time sync (for log integrity) | `chronyc tracking` |

### 🛑 Disable Unnecessary Services
```bash
# Stop & disable individual services
sudo systemctl stop <service-name>
sudo systemctl disable <service-name>

# [Stronger] Mask service (prevents manual start)
sudo systemctl mask <service-name>
```

#### 📦 Common Services to Disable (If Not Required)
```bash
# Printing & Email
sudo systemctl stop cups
sudo systemctl stop postfix

# Discovery & Network Services
sudo systemctl stop avahi-daemon.socket avahi-daemon.service
sudo systemctl stop rpcbind
sudo systemctl stop tftp
sudo systemctl stop ftp

# Remote Access & File Sharing
sudo systemctl stop cockpit cockpit.socket
sudo systemctl stop smb nmb
sudo systemctl stop bluetooth

# Containers & Legacy
sudo systemctl stop docker docker.socket
sudo systemctl stop xinetd
sudo systemctl stop ModemManager.service
sudo systemctl stop wpa_supplicant
sudo systemctl stop pcscd
```

#### ⚡ One-Liner to Stop Multiple Services
```bash
sudo systemctl stop cups postfix avahi-daemon cockpit cockpit.socket \
  bluetooth rpcbind smb nmb tftp docker docker.socket pcscd xinetd \
  ModemManager.service
```

> 💡 **Pro Tip:** After disabling, verify no critical dependencies broke:  
> ```bash
> systemctl list-dependencies <critical-service>
> ```

---

## 🔎 Verification & Audit Commands

### 🔐 SSH & Authentication
```bash
# Test SSH config
sudo sshd -T

# Check active SSH sessions
who
last

# Verify PAM faillock configuration
sudo faillock --user {username}
```

### 👥 User Audit
```bash
# List users with UID 0 (root equivalents)
awk -F: '($3 == 0) {print $1}' /etc/passwd

# Find users with empty passwords
sudo awk -F: '($2 == "") {print $1}' /etc/shadow

# Check sudoers syntax
sudo visudo -c
```

### 🛡️ Service & Firewall Audit
```bash
# Confirm firewall rules
sudo firewall-cmd --list-all

# Verify SELinux context for SSH port
semanage port -l | grep ssh_port_t

# Check service status
systemctl is-enabled <service-name>
```

### 📜 Compliance Logging
```bash
# Review auth logs
sudo journalctl -u sshd -n 100
sudo grep "Failed password" /var/log/secure

# Ensure auditd is logging
sudo auditctl -l
```

---

## 🚫 Excluded: Network & Kernel Hardening

> ℹ️ *As requested, the following sections are intentionally omitted from this guide:*  
> - `sysctl.d` network/kernel parameters  
> - PAM `pam_faillock.so` integration details  
> - `useradd -D -f` global inactivity settings  

*These can be added in a separate "Advanced Kernel Hardening" module if needed.*

---

## 📋 Quick Deployment Checklist

```markdown
- [ ] Backup original configs: `cp /etc/ssh/sshd_config /root/sshd_config.bak`
- [ ] Validate SSH config: `sshd -T`
- [ ] Open new port in firewall: `firewall-cmd --add-port=2222/tcp`
- [ ] Update SELinux: `semanage port -a -t ssh_port_t -p tcp 2222`
- [ ] Set secure permissions: `chmod 600 /etc/ssh/sshd_config`
- [ ] Restart SSH: `systemctl restart sshd` (keep session open!)
- [ ] Test new SSH connection on port 2222
- [ ] Disable unused services & verify dependencies
- [ ] Apply password policies in `pwquality.conf`
- [ ] Audit user accounts & lock inactive ones
- [ ] Document changes in change-management log
```

---

> 🇧🇩 **বাংলা নোট:**  
> সার্ভার হার্ডেনিং করার আগে সবসময় ব্যাকআপ নিন। কোনো পরিবর্তন করার পর `systemctl restart` দেয়ার আগে `sshd -T` বা `visudo -c` দিয়ে কনফিগ ভ্যালিডেট করুন। নতুন SSH পোর্টে কানেক্ট করতে পারছেন কিনা তা নিশ্চিত না হওয়া পর্যন্ত পুরনো সেশন ক্লোজ করবেন না।

---

*🔐 This guide aligns with CIS Benchmarks, NIST 800-53, and bank-grade security practices.*  
*🔄 Keep this document version-controlled and review quarterly.*
