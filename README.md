# DC-2 — VulnHub Walkthrough

A clear, step-by-step penetration testing writeup of the DC-2 Capture The Flag (CTF) machine.

## About this machine

| | |
|---|---|
| Name | DC-2 |
| Author | DCAU7 |
| Source | [VulnHub — DC-2](https://www.vulnhub.com/entry/dc-2,311/) |
| Target OS | Debian GNU/Linux (WordPress 4.7.10) |
| Goal | Find 5 flags, escalate to `root`, read the final flag |

DC-2 is a deliberately vulnerable virtual machine made for learning. The whole lab runs in a private, isolated network. Everything below was done on my own lab; never use these techniques against a system you do not own or do not have written permission to test.

The attacker machine is Kali Linux (`192.168.56.108`); the target is DC-2 (`192.168.56.105`).

---

## Step 1 — Reconnaissance

First I need to know which doors are open and what runs behind them. This decides where to attack.

```bash
sudo nmap -A 192.168.56.105
```

![Nmap Scan](https://github.com/user-attachments/assets/9d1df14f-eb1e-478e-892b-2540b1e97493)

Nmap found only port 80 (Apache on Debian). The page title said "DC-2 — Just another WordPress site", which confirmed a WordPress 4.7.10 website, so the next tool was clear: WPScan. Flag 1 is on the WordPress page itself — it tells you to build a password list from the site with CeWL.

---

## Step 2 — WordPress user enumeration

To brute-force a login, I first need valid usernames. WPScan can list the WordPress users automatically.

```bash
wpscan --url http://dc-2/ --enumerate u
```

![WPScan Users](https://github.com/user-attachments/assets/f99c786a-d587-4f7a-abc3-6694b9735d61)

WPScan found three users: `admin`, `jerry`, and `tom`.

---

## Step 3 — Password brute-force (CeWL + WPScan)

Flag 1 hinted at making a wordlist from the website's own words. Many people reuse words from their own site as passwords, so a custom list built from the site is often far better than a generic one.

```bash
cewl http://dc-2/ -w /tmp/dc2_wordlist.txt
wpscan --url http://dc-2/ -U tom,jerry -P /tmp/dc2_wordlist.txt
```

![WPScan Brute-force](https://github.com/user-attachments/assets/606be07a-c041-4e11-9f59-5adfde6547ae)

CeWL crawled the site and built the list; WPScan then tested it and found two passwords: `tom : parturient` and `jerry : adipiscing`.

---

## Step 4 — Flag 2 (WordPress dashboard)

With valid credentials, I can log into WordPress and look for private content that a normal visitor cannot see.

I logged in as `tom` and found Flag 2 on a private page at `/index.php/flag-2/`.

![Flag 2](https://github.com/user-attachments/assets/65e68e17-a373-421a-83bb-f11ab9aaf22a)

The message said that if you cannot exploit WordPress, there is another way in — a clear hint toward SSH.

---

## Step 5 — SSH access, rbash escape, Flag 3

SSH was open, and I already had Tom's password. But Tom's shell was a restricted shell (rbash) that blocks many commands, so I needed to escape it to do real work.

```bash
ssh tom@192.168.56.105 -p 7744
```

SSH was on the unusual port 7744. Inside the restricted shell, I escaped using `vi`, which can start another shell:

```bash
vi
:set shell=/bin/bash
:shell
```

![SSH + rbash escape](https://github.com/user-attachments/assets/d4b28973-2cd0-481c-9493-1c3573bf4c62)

Now in a full `/bin/bash`, I read Flag 3 in Tom's home folder. It hinted at switching to the user `jerry`.

---

## Step 6 — Switch to jerry, Flag 4

Flag 3 pointed to `jerry`, and I already had Jerry's password from the brute-force, so I could change user.

```bash
su jerry
cd ~
cat flag4.txt
```

![su jerry](https://github.com/user-attachments/assets/54aca352-558e-4c12-83a4-693cd5c523e8)

Flag 4 was in Jerry's home folder, and the message hinted at using `git` for the last step.

---

## Step 7 — Privilege escalation (sudo git)

The last question is always what this user can run as root.

```bash
sudo -l
```

Jerry could run `/usr/bin/git` as root with no password. `git` can open its help text inside the `less` pager, and from a pager I can start a shell — so a root-allowed `git` means a root shell:

```bash
sudo git -p help config
```

Then, inside the pager, I typed `!/bin/bash`.

![sudo git privesc](https://github.com/user-attachments/assets/91507755-31cc-4fc1-8355-b799dfc169ea)
![sudo git privesc 2](https://github.com/user-attachments/assets/6fbdafab-69fe-420f-ad47-3b37600049d7)
![sudo git privesc 3](https://github.com/user-attachments/assets/1bf070e5-884d-451d-a6a3-e0fdf31d4141)

This opened a shell as root.

---

## Step 8 — Root and final flag

```bash
cat /root/final-flag.txt
```

![Final Flag](https://github.com/user-attachments/assets/55363c29-ba8a-41d9-8005-6b3d996c32b0)

Full root access achieved; the final flag was in `/root/final-flag.txt`.

---

## Flags summary

| Flag | Location | Method |
|------|----------|--------|
| Flag 1 | WordPress page | Read the site |
| Flag 2 | `/index.php/flag-2/` | Login as `tom` |
| Flag 3 | `/home/tom/flag3.txt` | SSH + rbash escape via `vi` |
| Flag 4 | `/home/jerry/flag4.txt` | `su jerry` |
| Final | `/root/final-flag.txt` | `sudo git` privilege escalation |

---

## What I learned (defender's view)

- Keep WordPress updated — version 4.7.10 is old and well known.
- Use strong passwords that are not words taken from your own website.
- Do not rely on rbash as real protection — it is easy to escape.
- Changing the SSH port is not security — it only hides the service for a moment.
- Never give passwordless sudo to programs that can open a shell (`git`, `vi`, `less`, and similar).
- Disable XML-RPC if you do not need it, as it helps brute-force attacks.

---

## References

- [VulnHub — DC-2](https://www.vulnhub.com/entry/dc-2,311/)
- [GTFOBins — git](https://gtfobins.github.io/gtfobins/git/)
- [GTFOBins — vi](https://gtfobins.github.io/gtfobins/vi/)
