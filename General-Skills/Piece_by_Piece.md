# 🧩 Piece by Piece

| Field | Details |
|-------|---------|
| **CTF** | picoCTF 2026 |
| **Category** | General Skills |
| **Difficulty** | Medium |
| **Points** | 50 pts |
| **Author** | YAHAYA MEDDY |
| **Technique** | File Reconstruction + Password-Protected Archive Extraction |

---

## 🧩 Challenge Description

> *After logging in, you will find multiple file parts in your home directory. These parts need to be combined and extracted to reveal the flag.*

We are given SSH access to a remote machine:

```
ssh -p 57852 ctf-player@dolphin-cove.picoctf.net
Password: 84b12bae
```

The flag is hidden inside a zip archive that has been **split into multiple parts**. Our job is to reassemble the parts, extract the archive, and read the flag.

---

## 🔍 Reconnaissance

### Step 1 — SSH Into the Instance

```bash
ssh -p 57852 ctf-player@dolphin-cove.picoctf.net
# Password: 84b12bae
```

---

### Step 2 — List the Files

```bash
ctf-player@pico-chall:~$ ls
```

**Output:**
```
instructions.txt  part_aa  part_ab  part_ac  part_ad  part_ae
```

We can see multiple `part_*` files and a helpful `instructions.txt`. The parts are fragments of a split zip archive.

---

### Step 3 — Read the Instructions

Always read any provided hints or instruction files first:

```bash
ctf-player@pico-chall:~$ cat instructions.txt
```

**Output:**
```
Hint:
- The flag is split into multiple parts as a zipped file.
- Use Linux commands to combine the parts into one file.
- The zip file is password protected. Use this "supersecret" password to extract the zip file.
- After unzipping, check the extracted text file for the flag.
```

> 💡 The instructions directly tell us the zip password is `supersecret`. In a harder challenge you'd need to find or crack it — always check instruction files before going further.

---

## 💥 Exploitation

### Step 4 — Reassemble the Parts

The `part_aa`, `part_ab`, etc. files are the output of Linux's `split` command, which breaks a file into sequentially named chunks. We reverse this using `cat` with a wildcard:

```bash
ctf-player@pico-chall:~$ cat part_* > combined.zip
```

The `*` wildcard matches all `part_` files **in alphabetical order** (`part_aa` → `part_ab` → ... → `part_ae`), concatenating them back into the original file.

---

### Step 5 — Verify the File Type

Confirm the reassembled file is a valid zip archive:

```bash
ctf-player@pico-chall:~$ file combined.zip
```

**Output:**
```
combined.zip: Zip archive data, at least v1.0 to extract
```

✅ Valid zip archive — reassembly was successful.

---

### Step 6 — Extract the Archive

Unzip the archive. When prompted for a password, enter `supersecret` (from `instructions.txt`):

```bash
ctf-player@pico-chall:~$ unzip combined.zip
```

**Output:**
```
Archive:  combined.zip
[combined.zip] flag.txt password:
  extracting: flag.txt
```

> ⚠️ Note: You may need to enter the password twice if the first attempt shows `password incorrect--reenter`. This can happen due to a terminal quirk — just type `supersecret` again.

---

### Step 7 — Read the Flag

```bash
ctf-player@pico-chall:~$ cat flag.txt
```

**Output:**
```
picoCTF{z1p_and_spl1t_f1l3s_4r3_fun_78b76e61}
```

---

## 🚩 Flag

```
picoCTF{z1p_and_spl1t_f1l3s_4r3_fun_78b76e61}
```

---

## 📚 Concepts & Further Reading

### How `split` and `cat` Work Together

The Linux `split` command breaks a file into smaller chunks with sequential names:

```bash
# Splitting a file into parts (how the challenge was created)
split -b 1024 archive.zip part_

# This produces: part_aa, part_ab, part_ac ...
```

To **reconstruct** the original file, `cat` with a wildcard does the job:

```bash
# Reassembling the parts (what we did)
cat part_* > combined.zip
```

> ⚠️ **Order matters.** The `*` wildcard expands files in alphabetical order, which matches the way `split` names them (`aa` → `ab` → `ac`...). This ensures the bytes are concatenated in the correct sequence.

### Verifying File Integrity

The `file` command reads the **magic bytes** at the start of a file to determine its type — it doesn't rely on the file extension. This is a useful sanity check after reassembly:

```bash
file combined.zip   # → Zip archive data
file image.png      # → PNG image data
file mystery        # → ELF 64-bit LSB executable
```


---

## 🛡️ Defensive Perspective

- **Split archives** are commonly used to bypass file size limits on email or file-sharing platforms. Malware authors also use this technique to evade detection by splitting payloads.
- **Password-protected zips** with weak passwords (like `supersecret`, `password`, `123456`) offer little real security. Always use strong, random passwords for sensitive archives.
- During incident response, always check for split file artifacts — tools like `foremost` and `binwalk` can help identify and reassemble them.

---

## 🗺️ Attack Summary

```
SSH into machine
       ↓
ls → part_aa, part_ab, part_ac, part_ad, part_ae + instructions.txt
       ↓
cat instructions.txt → password is "supersecret"
       ↓
cat part_* > combined.zip  ← reassemble split archive
       ↓
file combined.zip → confirmed Zip archive data
       ↓
unzip combined.zip → password: supersecret
       ↓
cat flag.txt → picoCTF{z1p_and_spl1t_f1l3s_4r3_fun_78b76e61}
```

---

> **Key lesson:** `cat part_* > output` is the standard one-liner to reassemble split files on Linux. The `split` command creates them; `cat` reverses it. Always read `instructions.txt` or any hint files before spending time guessing passwords — the answer is often right there.
