# Digital-Echo - WRITEUP
# CTF Write-up: OSINT / Forensics Investigation Chain

**Rooms covered:** The First Trace → The Vanishing File → Provenance

---

## Task 1 — The First Trace

**Objective:** Identify the persistent identity embedded in the recovered photograph, then use it to pivot to a public code repository and extract a hidden identity from a non-text file inside it.

### Step 1 — Extract metadata from the image

The photo itself gives nothing away visually, so the file's metadata is examined instead:

```bash
exiftool <image-file>.jpg
```

The `Artist` field in the EXIF data reveals the persistent identity left behind by whoever created/exported the file:

```
Artist: lunarTracesz
```

**🚩 Flag 1:** `lunarTracesz`

### Step 2 — Pivot to the public footprint

Searching the recovered username surfaces a matching GitHub account:

```
https://github.com/lunarTracesz
```

Inside it sits a repository called `photo-journal`:

```
https://github.com/lunarTracesz/photo-journal.git
```

### Step 3 — Clone the repository and inspect the odd file

```bash
git clone https://github.com/lunarTracesz/photo-journal.git
cd photo-journal
ls
```

```
iamge-2.jpg  iamge_3.jpg  image-1.jpg  key.asc  README.md
```

`key.asc` stands out — an ASCII-armored file "never meant to be read like an ordinary text file." It's a GPG key. Importing it and listing keys recovers the identity attached to it:

```bash
gpg --import key.asc
gpg --list-secret-keys
```

```
sec   rsa4032 2026-07-18 [SC]
      002553C7CD6E2D209F6C642BC8B1F9425BFACBEC
uid   [ultimate] Elena Voss (History remembers what the latest commit forgot)
      <elena4sure@gmail.com>
ssb   rsa4032 2026-07-18 [E]
```

**🚩 Flag 2 (email):** `elena4sure@gmail.com`

**Identity attached to the key:** Elena Voss
**Key comment (a lead for later tasks):** *"History remembers what the latest commit forgot"* — a hint that Git history, not the working tree, holds the next clue.

---

## Task 2 — The Vanishing File

**Objective:** The repo *looks* clean, but the GPG comment hinted that something was scrubbed from commit history. Recover it, follow the surviving handle to X (Twitter), and decode a machine-readable artifact.

### Step 1 — Recover deleted file from Git history

```bash
git log --diff-filter=D --summary          # find commits that deleted files
git log --all --full-history -- ticket_log.md
git show <commit-hash>:ticket_log.md        # dump the deleted file's contents
```

This recovers a file that was clearly meant to be deleted before the repo went public — `ticket_log.md` — containing a personal FIFA World Cup ticket log:

```
X: @0nIyH4AIand
Internal Reference: ZmxhZ3tGbEZBX1cwUkxEQ1U5X1RSQUMzXzJPMjZ9
```

**🚩 Flag (surviving handle):** `@0nIyH4AIand` (X / Twitter)

*(Note: the Base64 string `ZmxhZ3tGbEZBX1cwUkxEQ1U5X1RSQUMzXzJPMjZ9` decodes to a leetspeak flag embedded in the commit itself — a secondary artifact confirming the file was intentionally obfuscated before its planned deletion.)*

### Step 2 — Follow the handle to X and find the machine-readable artifact

Visiting `https://x.com/0nIyH4AIand` turns up a post containing a **photo of the user's FIFA World Cup match ticket**, with a **QR code** printed on it — the "information meant for machines rather than human readers."

Scanning the QR code (via any QR decoder, e.g. `zbarimg`, an online scanner, or a phone camera) reveals:

**🚩 Flag 3:** `flag{FlF4_W0RLDCUP_L0V3R}`

---

## Task 3 — Provenance

**Objective:** A photo posted by the account was not original — trace it to its real source article, identify the correct event/venue among several red herrings, and corroborate the venue with a nearby international airport.

### Step 1 — Reverse-trace the profile/post thumbnail image

The image used as the thumbnail on the account's first post is a lift from a published news story — a **BBC News** article:

```
https://www.bbc.com/news/articles/ceqd3l4v7reo
```

### Step 2 — Extract the relevant details from the article

The article covers a Norwegian striker's (Haaland) unexpected popularity in the U.S. during the 2026 FIFA World Cup, and mentions multiple names/locations. Cross-referencing the reported quarter-final match with the photo narrows it down to:

| Detail | Value |
|---|---|
| Author | **Rebecka Pieder** |
| Stadium (Norway vs. England quarter-final) | **Hard Rock Stadium**, Miami Gardens, FL *(rebranded "Miami Stadium" for the tournament)* |
| Nearby international airport (3-letter code) | **MIA** (Miami International Airport) |

---

## Summary of Flags / Answers

| # | Question | Answer |
|---|---|---|
| 1 | Persistent identity in photo metadata | `lunarTracesz` |
| 2 | Email tied to identity in `key.asc` | `elena4sure@gmail.com` |
| 3 | External-platform handle surviving in Git history | `@0nIyH4AIand` |
| 4 | Value concealed in the machine-readable (QR) artifact | `flag{FlF4_W0RLDCUP_L0V3R}` |
| 5 | Publication/author of the original photo's source story | BBC News — **Rebecka Pieder** |
| 6 | Stadium hosting the connected quarter-final | **Hard Rock Stadium** (Miami) |
| 7 | Three-letter airport code of the venue's neighbour | **MIA** |

---

## Tools Used

- `exiftool` — image metadata extraction
- `git log` / `git show` — recovering deleted files from version history
- `gpg --import` / `gpg --list-secret-keys` — inspecting an ASCII-armored key file
- Online/CLI QR decoder — reading machine-encoded data from an image
- Manual OSINT — pivoting between GitHub, X, and news sources
