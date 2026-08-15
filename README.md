# Digital Echo — CTF Write-up

> **Platform:** TryHackMe  
> **Room:** Digital Echo  
> **Category:** OSINT / Forensics  
> **Investigation Chain:** The First Trace → The Vanishing File → Provenance

## 🔗 Room Link

**TryHackMe Room:**  
https://tryhackme.com/room/digitalecho

## 🖥️ Room UI

The investigation is performed through the TryHackMe **Digital Echo** room interface.

![Digital Echo TryHackMe Room UI](assets/room-ui.png)

---

# Overview

**Digital Echo** is an OSINT and forensics investigation built around a fragmented digital footprint. The investigation begins with a seemingly ordinary photograph and gradually moves through image metadata, GitHub, GPG keys, Git history, X/Twitter, QR-code analysis, and open web sources.

The main investigation chain is:

```text
Recovered Photograph
        ↓
EXIF Metadata
        ↓
Artist: lunarTracesz
        ↓
GitHub Profile
        ↓
photo-journal Repository
        ↓
key.asc
        ↓
GPG Identity / Email
        ↓
Git History
        ↓
Deleted ticket_log.md
        ↓
X Username
        ↓
FIFA World Cup Ticket
        ↓
QR Code
        ↓
Flag
        ↓
BBC News Article
        ↓
Rebecka Pieder / Hard Rock Stadium / MIA
```

---

# Task 1 — The First Trace

## Objective

Identify the persistent identity embedded in the recovered photograph, then use it to pivot to a public code repository and extract a hidden identity from a non-text file.

## Step 1 — Extract Metadata from the Image

The photograph does not reveal much information visually, so the first step is to inspect its metadata.

I used `exiftool`:

```bash
exiftool <image-file>.jpg
```

The `Artist` field revealed the persistent identity associated with the photograph:

```text
Artist: lunarTracesz
```

### 🚩 Flag 1

```text
lunarTracesz
```

This username became the first OSINT pivot.

---

## Step 2 — Pivot to the Public Footprint

Searching for the recovered username led to a matching GitHub account:

https://github.com/lunarTracesz

The profile contained a repository named **photo-journal**:

https://github.com/lunarTracesz/photo-journal

---

## Step 3 — Clone the Repository

I cloned the repository locally:

```bash
git clone https://github.com/lunarTracesz/photo-journal.git
cd photo-journal
ls
```

The repository contained:

```text
iamge-2.jpg
iamge_3.jpg
image-1.jpg
key.asc
README.md
```

The file `key.asc` immediately stood out because it was an ASCII-armored GPG key rather than an ordinary text file.

---

## Step 4 — Investigate `key.asc`

I imported the key and inspected the available GPG identities:

```bash
gpg --import key.asc
gpg --list-secret-keys
```

The output revealed:

```text
sec   rsa4032 2026-07-18 [SC]
      002553C7CD6E2D209F6C642BC8B1F9425BFACBEC
uid   [ultimate] Elena Voss (History remembers what the latest commit forgot)
      <elena4sure@gmail.com>
ssb   rsa4032 2026-07-18 [E]
```

The identity attached to the key was:

**Elena Voss**

The associated email address was:

```text
elena4sure@gmail.com
```

### 🚩 Flag 2

```text
elena4sure@gmail.com
```

### Important Clue

The GPG identity contained the comment:

> **History remembers what the latest commit forgot**

This strongly suggested that the next clue would be found in the repository's **Git history**, rather than in the current working tree.

---

# Task 2 — The Vanishing File

## Objective

Recover information that had been removed from the current repository, follow the surviving handle to X/Twitter, and decode a machine-readable artifact.

## Step 1 — Recover the Deleted File

Following the clue from the GPG key, I investigated the repository's Git history.

First, I searched for deleted files:

```bash
git log --diff-filter=D --summary
```

I then specifically searched for `ticket_log.md`:

```bash
git log --all --full-history -- ticket_log.md
```

After identifying the relevant commit, I recovered the contents of the deleted file:

```bash
git show <commit-hash>:ticket_log.md
```

Although `ticket_log.md` was no longer present in the latest version of the repository, Git preserved its previous contents.

The recovered file contained:

```text
X: @0nIyH4AIand
Internal Reference: ZmxhZ3tGbEZBX1cwUkxEQ1U5X1RSQUMzXzJPMjZ9
```

The surviving X/Twitter handle became the next OSINT pivot:

```text
@0nIyH4AIand
```

---

## Step 2 — Investigate the Base64 Artifact

The deleted commit also contained the Base64 string:

```text
ZmxhZ3tGbEZBX1cwUkxEQ1U5X1RSQUMzXzJPMjZ9
```

It can be decoded with:

```bash
echo 'ZmxhZ3tGbEZBX1cwUkxEQ1U5X1RSQUMzXzJPMjZ9' | base64 -d
```

This Base64 value acts as a **secondary artifact/clue** embedded in the commit. It supports the conclusion that information was intentionally obfuscated before the file was removed.

> **Note:** This secondary Base64 artifact should not be confused with the main QR-code flag obtained later.

---

## Step 3 — Follow the Handle to X

I investigated the recovered account:

https://x.com/0nIyH4AIand

The account contained a post featuring a **FIFA World Cup match ticket**.

The ticket included a **QR code**, representing information intended to be read by machines rather than humans.

I decoded the QR code using a QR-code decoder.

The decoded value was:

### 🚩 Flag 3

```text
flag{FlF4_W0RLDCUP_L0V3R}
```

---

# Task 3 — Provenance

## Objective

Trace an image posted by the account back to its original source and identify the relevant author, venue, and nearby international airport.

## Step 1 — Identify the Original Source

I returned to the X account and examined its first post.

The thumbnail image used in the post appeared to originate from a published **BBC News** article:

https://www.bbc.com/news/articles/ceqd3l4v7reo

---

## Step 2 — Extract the Relevant Details

After investigating the article and cross-referencing the reported event and locations, the relevant details were identified as follows:

| Detail | Value |
|---|---|
| Author | **Rebecka Pieder** |
| Stadium | **Hard Rock Stadium** |
| Location | **Miami Gardens, Florida** |
| Nearby international airport | **Miami International Airport** |
| Airport code | **MIA** |

The connected stadium was **Hard Rock Stadium** in Miami Gardens, Florida.

---

# Summary of Flags / Answers

| # | Question | Answer |
|---|---|---|
| 1 | Persistent identity in photo metadata | `lunarTracesz` |
| 2 | Email tied to identity in `key.asc` | `elena4sure@gmail.com` |
| 3 | External-platform handle recovered from Git history | `@0nIyH4AIand` |
| 4 | Value concealed in the QR artifact | `flag{FlF4_W0RLDCUP_L0V3R}` |
| 5 | Author of the original source story | **Rebecka Pieder** |
| 6 | Stadium hosting the connected quarter-final | **Hard Rock Stadium** |
| 7 | Three-letter airport code | **MIA** |

---

# Tools Used

| Tool | Purpose |
|---|---|
| `exiftool` | Extracting image metadata |
| `git clone` | Cloning the public repository |
| `git log` | Investigating repository history |
| `git show` | Recovering the deleted file |
| `gpg --import` | Importing the ASCII-armored key |
| `gpg --list-secret-keys` | Inspecting the identity attached to the key |
| QR decoder | Extracting machine-readable information from the FIFA ticket |
| Manual OSINT | Pivoting between GitHub, X/Twitter, and BBC News |

---

# Key Takeaways

### 1. Do not rely only on what is visible

A photograph may look completely ordinary while its metadata contains useful investigative information.

### 2. Metadata can provide a persistent identity

The `Artist` EXIF field revealed:

```text
lunarTracesz
```

This provided the first pivot.

### 3. Non-text files can contain valuable information

The `key.asc` file was not intended to be interpreted as a normal text document. Treating it as a GPG key revealed:

```text
Elena Voss
elena4sure@gmail.com
```

### 4. Deleted files may still survive in Git history

Removing a file from the current working tree does not necessarily erase its historical versions.

The following commands were useful:

```bash
git log --all --full-history -- ticket_log.md
```

```bash
git show <commit-hash>:ticket_log.md
```

### 5. Every discovery can become an OSINT pivot

The investigation followed a continuous chain:

```text
lunarTracesz
    ↓
GitHub
    ↓
key.asc
    ↓
Elena Voss
    ↓
Git History
    ↓
ticket_log.md
    ↓
0nIyH4AIand
    ↓
X/Twitter
    ↓
FIFA Ticket
    ↓
QR Code
    ↓
Flag
```

---

# Conclusion

The **Digital Echo** investigation demonstrates how OSINT and digital forensics can be combined to reconstruct a fragmented digital footprint.

The investigation started with a single photograph. Its metadata revealed the username `lunarTracesz`, which led to a public GitHub profile. The associated repository contained a GPG key that revealed another identity and included a clue pointing toward Git history.

Investigating the historical commits uncovered the deleted `ticket_log.md` file. The file provided an X/Twitter handle, which led to a FIFA World Cup ticket containing a QR code. Decoding that QR code produced the main flag.

The investigation then continued through the account's earlier post to its original BBC News source, revealing the relevant author, stadium, and airport information.

The key lesson is simple:

> **Every file has a history. Some of it simply isn't visible.**

---

## References

- TryHackMe — Digital Echo: https://tryhackme.com/room/digitalecho
- GitHub — lunarTracesz: https://github.com/lunarTracesz
- GitHub — photo-journal: https://github.com/lunarTracesz/photo-journal
- X/Twitter — recovered account: https://x.com/0nIyH4AIand
- BBC News — source article: https://www.bbc.com/news/articles/ceqd3l4v7reo
