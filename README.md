<<div align="center">

# 🔎 Digital Echo — CTF Write-up

![Platform](https://img.shields.io/badge/Platform-TryHackMe-1CB89C?style=for-the-badge&logo=tryhackme&logoColor=white)
![Category](https://img.shields.io/badge/Category-OSINT%20%2F%20Forensics-0EA5E9?style=for-the-badge)
![Difficulty](https://img.shields.io/badge/Difficulty-Medium-F5A623?style=for-the-badge)
![Time](https://img.shields.io/badge/Time-60%20min-6C5CE7?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Completed-22C55E?style=for-the-badge)

### 🕵️ OSINT / Forensics Investigation

**Category:** OSINT / Forensics  
**Platform:** TryHackMe  
**Room:** Digital Echo  
**Investigation Chain:**  
🖼️ Image Metadata → 👤 GitHub → 🔑 GPG Identity → 🕵️ Git History → 🐦 X/Twitter → 🔳 QR Code → 📰 News Provenance

<br>

<a href="https://tryhackme.com/room/digitalecho">
  <img src="./assets/DigEcho.jpeg" alt="TryHackMe Digital Echo Room" width="900">
</a>

<br>

**🔗 [View Digital Echo on TryHackMe](https://tryhackme.com/room/digitalecho)**

</div>



## 🧩 Room 1: The First Trace

> **🎯 Objective**
> Identify the persistent identity left behind by an image artifact, then use that identity to recover a hidden email address from a repository file.

### 🏷️ Step 1 — Download the image and pull metadata

The challenge hints that *"the file itself may know more about its origin than the pixels do"* — a direct pointer to EXIF/metadata inspection rather than visual analysis.

```bash
exiftool image.jpg
```

Inside the metadata, the `Artist` field contained a value that was not visible anywhere in the image itself:

```text
Artist: lunarTracesz
```

> 🚩 **Flag 1 (persistent identity / username)**
> ```text
> lunarTracesz
> ```

### 🔍 Step 2 — Pivot to the public footprint

Using the recovered username as a pivot point (per the prompt: *"use it as your initial pivot"*), a search for the handle on GitHub resolved to a real account:

🔗 https://github.com/lunarTracesz

That account hosts a repository called `photo-journal`:

🔗 https://github.com/lunarTracesz/photo-journal.git

### 📦 Step 3 — Clone the repo and inspect the suspicious file

```bash
git clone https://github.com/lunarTracesz/photo-journal.git
cd photo-journal
ls
```

<details>
<summary>📂 Repository contents</summary>

```text
iamge-2.jpg  iamge_3.jpg  image-1.jpg  key.asc  README.md
```

</details>

`key.asc` is the file *"never meant to be read like an ordinary text file"* — it's an ASCII-armored **GPG key**, not plaintext. Importing it and listing secret keys reveals the identity bound to it:

```bash
gpg --import key.asc
gpg --list-secret-keys
```

<details>
<summary>🖥️ Output</summary>

```text
sec   rsa4032 2026-07-18 [SC]
      002553C7CD6E2D209F6C642BC8B1F9425BFACBEC
uid   [ultimate] Elena Voss (History remembers what the latest commit forgot) <elena4sure@gmail.com>
ssb   rsa4032 2026-07-18 [E]
```

</details>

The UID comment — *"History remembers what the latest commit forgot"* — is a deliberate nod toward the next room: the repo's **git history**, not its current working tree, holds the next lead.

> 🚩 **Flag 2 (recovered identity / email)**
> ```text
> elena4sure@gmail.com
> ```

---

## 🧩 Room 2: The Vanishing File

> **🎯 Objective**
> Recover a file that was deleted before the repo was "cleaned up," extract a surviving handle from it, and pull a machine-readable value out of the image it leads to.

### 🕵️ Step 1 — Recover deleted history

The current working tree looks clean — the deleted file (`ticket_log.md`) doesn't exist in the latest commit. Full history has to be walked instead:

```bash
git log --diff-filter=D --summary          # find commits that deleted files
git log --all --full-history -- ticket_log.md
```

Once the commit *before* deletion is found, the file's content can be dumped directly from the object store:

```bash
git show <commit-hash>:ticket_log.md
```

This recovered a personal "FIFA World Cup ticket log" file that was never meant to survive the cleanup, containing an X (Twitter) handle left behind in the notes.

> 🚩 **Flag 1 (surviving external-platform handle)**
> ```text
> 0nIyH4AIand
> ```

### 🔳 Step 2 — Follow the handle to X and decode the QR

Pivoting to 🔗 `https://x.com/0nIyH4AIand`, the account's post shows a photo of the user's FIFA World Cup match ticket. Embedded in that image is a **QR code** — the *"information meant for machines rather than human readers"* referenced in the prompt.

Decoding the QR (any online QR reader, or `zbarimg`/`zxing` locally) revealed the concealed value:

```bash
zbarimg ticket_qr.jpg
```

> 🚩 **Flag 2 (value concealed in the visual artifact)**
> ```text
> flag{FlF4_W0RLDCUP_L0V3R}
> ```

---

## 🧩 Room 3: Provenance

> **🎯 Objective**
> Trace an image shared by the account back to its original published source, identify the associated event, and independently corroborate the venue.

### 📰 Step 1 — Identify the original source

The X account's first post uses, as its thumbnail, an image lifted from a BBC News article rather than an original photo:

🔗 https://www.bbc.com/news/articles/ceqd3l4v7reo

Reading the article surfaces:

- ✍️ **Author:** Rebecka Pieder
- 📰 **Publication:** BBC News

### 🌍 Step 2 — Corroborate the venue

The article ties the photo to a specific World Cup quarter-final. Cross-referencing the match details identifies:

**🏟️ Stadium:**
```text
Hard Rock Stadium
```
(officially rebranded "Miami Stadium" for the tournament under FIFA's naming-rights policy; hosted the Norway vs. England quarter-final).

**✈️ Nearby international airport (three-letter code):**
```text
MIA
```
(Miami International Airport — the geographical neighbor "capable of receiving international travellers.")

---

## 🏁 Summary of Flags

| Room | Step | Flag / Value |
|---|---|---|
| The First Trace | Persistent identity (EXIF Artist field) | `lunarTracesz` |
| The First Trace | Identity behind `key.asc` (GPG UID) | `elena4sure@gmail.com` |
| The Vanishing File | Surviving handle (recovered deleted commit) | `@0nIyH4AIand` |
| The Vanishing File | QR code payload | `flag{FlF4_W0RLDCUP_L0V3R}` |
| Provenance | Original publication / author | BBC News — Rebecka Pieder |
| Provenance | Match venue | Hard Rock Stadium (Miami Stadium) |
| Provenance | Nearby international airport code | `MIA` |

## 🛠️ Key Techniques Used

1. 🏷️ **EXIF/metadata extraction** (`exiftool`) — the first and most overlooked lead in an "innocent-looking" image.
2. 🔍 **OSINT pivoting** — treating a recovered username as a search key across platforms (GitHub, X).
3. 🔑 **GPG key inspection** (`gpg --import`, `gpg --list-secret-keys`) — identity data is often embedded in key UIDs, not just certificates.
4. 🕵️ **Git forensics** (`git log --diff-filter=D`, `git show <hash>:<path>`) — recovering content that was deleted from the working tree but never purged from history.
5. 🔳 **QR/steganographic decoding** — recognizing "machine-readable" language as a cue to look for QR codes or embedded metadata rather than visible text.
6. 📰 **Source verification** — tracing a reused image back to its original publisher/author to corroborate real-world facts (venue, event, location).
