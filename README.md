
# DC-2 — VulnHub CTF Write-up

**Machine:** DC-2  
**Platform:** VulnHub  
**Difficulty:** Beginner/Intermediate  
**Goal:** Get root and read the final flag  
**Attack machine:** Kali Linux (`192.168.56.108`)  
**Target machine:** DC-2 (`192.168.56.105`)  
**Network:** VirtualBox Host-Only

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `nmap` | Port scanning and service detection |
| `wpscan` | WordPress enumeration and brute-force |
| `cewl` | Custom wordlist generation |
| `ssh` | Remote access to the target |
| `vi` | Escape restricted shell (rbash) |
| `sudo git` | Privilege escalation to root |

## Step 1 — Reconnaissance

**Command:**
```bash
sudo nmap -A 192.168.56.105
```

**Screenshot:**

<img width="1920" height="1080" alt="VirtualBox_kali_19_06_2026_15_05_46" src="https://github.com/user-attachments/assets/9d1df14f-eb1e-478e-892b-2540b1e97493" />

**What happened:**  
Nmap revealed only one open port — **port 80 running Apache 2.4.10 on Debian**.  
The HTTP title said "DC-2 — Just another WordPress site", which immediately told us the target runs **WordPress 4.7.10**.

**Why it matters:**  
Knowing the service and version lets us choose the right tool. WordPress has its own dedicated scanner — WPScan — so that's the next step.

---

## Step 2 — WordPress User Enumeration

**Command:**
```bash
wpscan --url http://dc-2/ --enumerate u
```

**Screenshot:**
<img width="1920" height="1080" alt="VirtualBox_kali_19_06_2026_15_06_17" src="https://github.com/user-attachments/assets/f99c786a-d587-4f7a-abc3-6694b9735d61" />

**What happened:**  
WPScan found **3 valid WordPress users**:
- `admin`
- `jerry`
- `tom`

**Why it matters:**  
WordPress usernames are the first half of the login puzzle. Now we need their passwords.

---

## Step 3 — Password Brute-Force

**Commands:**
```bash
cewl http://dc-2/ -w /tmp/dc2_wordlist.txt
wpscan --url http://dc-2/ -U tom,jerry -P /tmp/dc2_wordlist.txt
```

**Screenshot:**

<img width="1920" height="1080" alt="VirtualBox_kali_19_06_2026_15_06_52" src="https://github.com/user-attachments/assets/606be07a-c041-4e11-9f59-5adfde6547ae" />
**What happened:**  
CeWL crawled the WordPress site and built a custom wordlist from words found on the pages.  
WPScan then used that wordlist to brute-force the login via XML-RPC.

**Credentials found:**
- `tom` / `parturient`
- `jerry` / `adipiscing`

**Why it matters:**  
Website owners often use words related to their content as passwords. CeWL exploits this by generating a targeted wordlist instead of using a generic one like `rockyou.txt`. It's faster and more likely to succeed.

---

## Step 4 — Flag 2 (WordPress Dashboard)

**Screenshot:**
<img width="1920" height="1080" alt="VirtualBox_kali_19_06_2026_15_13_53" src="https://github.com/user-attachments/assets/65e68e17-a373-421a-83bb-f11ab9aaf22a" />
**What happened:**  
Logged into WordPress as `tom` and found **Flag 2** hidden in a private page at `/index.php/flag-2/`.

**Flag 2 message:**
> "If you can't exploit WordPress and take a shortcut, there is another way. Hope you found another entry point."

**Why it matters:**  
This was a hint pointing us toward SSH — the "other way in".

---

## Step 5 — SSH Access + rbash Escape + Flag 3

**Commands:**
```bash
ssh tom@192.168.56.105 -p 7744
```
Once connected (restricted shell):
```bash
vi
:set shell=/bin/bash
:shell
/bin/cat flag3.txt
```

**Screenshot:**

<img width="1920" height="1080" alt="VirtualBox_kali_19_06_2026_16_31_13" src="https://github.com/user-attachments/assets/d4b28973-2cd0-481c-9493-1c3573bf4c62" />

**What happened:**  
SSH was running on **port 7744** (not the default 22 — always scan all ports).  
After logging in as `tom`, we were dropped into **rbash** (restricted bash), which blocked many commands including `cat`, `su`, and paths with `/`.

We escaped using `vi` — a text editor available even in rbash — by launching a full bash shell from inside it.

**Flag 3 message:**
> "Poor old Tom is always running after Jerry. Perhaps he should su for all the stress he causes."

**Why it matters:**  
rbash is a common restriction on CTF machines and real-world systems. `vi` is a classic escape because it can spawn a shell. This is documented on [GTFOBins](https://gtfobins.github.io/gtfobins/vi/).

---

## Step 6 — Switch to Jerry + Flag 4

**Commands:**
```bash
/bin/su jerry
# password: adipiscing
cd ~
/bin/cat flag4.txt
```

**Screenshot:**

<img width="1920" height="1080" alt="VirtualBox_kali_19_06_2026_16_41_47" src="https://github.com/user-attachments/assets/54aca352-558e-4c12-83a4-693cd5c523e8" />

**What happened:**  
Switched from `tom` to `jerry` using the password found in Step 3.  
Found **Flag 4** in jerry's home directory.

**Flag 4 message:**
> "Good to see that you've made it this far — but you're not home yet. You still need to get the final flag. No hints here — you're on your own now. Go on — git outta here!!!!"

**Why it matters:**  
The hint "git outta here" was a clear pointer toward `git` as the privesc vector.

---

## Step 7 — Privilege Escalation via sudo git

**Commands:**
```bash
sudo -l
sudo git -p help config
```
Inside the pager (`less`):
```
!/bin/bash
```

**Screenshots:**
<img width="1920" height="1080" alt="VirtualBox_kali_22_06_2026_09_12_35" src="https://github.com/user-attachments/assets/91507755-31cc-4fc1-8355-b799dfc169ea" />
<img width="1920" height="1080" alt="VirtualBox_kali_22_06_2026_09_12_21" src="https://github.com/user-attachments/assets/6fbdafab-69fe-420f-ad47-3b37600049d7" />
<img width="1920" height="1080" alt="VirtualBox_kali_22_06_2026_09_12_01" src="https://github.com/user-attachments/assets/1bf070e5-884d-451d-a6a3-e0fdf31d4141" />


**What happened:**  
`sudo -l` showed that jerry could run `/usr/bin/git` as root with **no password required**:
```
(root) NOPASSWD: /usr/bin/git
```

`sudo git -p help config` opened the git manual in the `less` pager running as root.  
From inside `less`, typing `!/bin/bash` spawned a **root shell**.

**Why it matters:**  
This is a well-known GTFOBins technique. When `sudo` allows a binary that can call system commands (like `less`, `vi`, `git`), it can be abused to escape into a root shell. The `-p` flag forces git to use a pager, which is the key to triggering this.

Reference: [GTFOBins — git](https://gtfobins.github.io/gtfobins/git/)

---

## Step 8 — Root + Final Flag

**Commands:**
```bash
ls -la /root/
cat /root/final-flag.txt
```

**Screenshot:**

<img width="1920" height="1080" alt="VirtualBox_kali_22_06_2026_09_19_19" src="https://github.com/user-attachments/assets/55363c29-ba8a-41d9-8005-6b3d996c32b0" />

**What happened:**  
Full root access confirmed. The final flag was in `/root/final-flag.txt`.

```
Well done!!!
Congratulations!!!
```

---

## Flags Summary

| Flag | Location | How |
|------|----------|-----|
| Flag 1 | WordPress page (source) | Browse the site |
| Flag 2 | WordPress private page `/flag-2/` | Login as tom |
| Flag 3 | `/home/tom/flag3.txt` | SSH + rbash escape |
| Flag 4 | `/home/jerry/flag4.txt` | su jerry |
| Final Flag | `/root/final-flag.txt` | sudo git privesc |

---

## Remediation

What a real sysadmin should fix on this machine:

- **Outdated WordPress (4.7.10)** — update to the latest version
- **Weak passwords** — `parturient` and `adipiscing` are dictionary words; enforce strong password policy
- **SSH on non-standard port** — security through obscurity, not a real fix; use key-based auth instead
- **rbash as a security control** — trivially bypassable via `vi`; not a reliable restriction
- **NOPASSWD sudo on git** — never give NOPASSWD sudo to binaries that can spawn shells
- **XML-RPC enabled** — disable it if not needed; it allows brute-force without lockout

---

## References

- [VulnHub — DC-2](https://www.vulnhub.com/entry/dc-2,311/)
- [GTFOBins — git](https://gtfobins.github.io/gtfobins/git/)
- [GTFOBins — vi](https://gtfobins.github.io/gtfobins/vi/)
- [WPScan Documentation](https://github.com/wpscanteam/wpscan)
