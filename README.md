# CTF Wiki — English Edition

> **An English fork of [ctf-wiki](https://github.com/ctf-wiki/ctf-wiki)** — a comprehensive, community-driven guide to Capture The Flag (CTF) competitions and cybersecurity techniques.

[![Discord](https://dcbadge.vercel.app/api/server/ekv7WDa9pq)](https://discord.gg/ekv7WDa9pq)

---

## What is CTF Wiki?

**CTF** (Capture The Flag) competitions have been a cornerstone of the cybersecurity community since the first **DEFCON CTF** in 1996. They cover a wide range of fields — from binary exploitation and reverse engineering to web security and cryptography.

This wiki provides **structured, beginner-friendly documentation** covering the knowledge and techniques used in modern CTF challenges. Whether you're just getting started or looking to sharpen advanced skills, you'll find something useful here.

---

## 📖 Table of Contents

| Section | Description |
|---------|-------------|
| [**Introduction**](docs/en/docs/introduction/history.md) | History of CTFs, competition formats, and getting started |
| [**Pwn (Binary Exploitation)**](docs/en/docs/pwn/) | Stack overflows, heap exploitation, format strings, ROP chains, and more |
| [**Reverse Engineering**](docs/en/docs/reverse/) | Static & dynamic analysis, common tools, and techniques |
| [**Web Security**](docs/en/docs/web/) | SQL injection, XSS, CSRF, SSRF, and other web vulnerabilities |
| [**Cryptography**](docs/en/docs/crypto/) | Classical ciphers, symmetric/asymmetric crypto, hash attacks |
| [**Miscellaneous**](docs/en/docs/misc/) | Forensics, steganography, network traffic analysis |
| [**Blockchain**](docs/en/docs/blockchain/) | Smart contract security and blockchain CTF challenges |
| [**Android**](docs/en/docs/android/) | Android reverse engineering and mobile security |
| [**Assembly**](docs/en/docs/assembly/) | x86, ARM, MIPS assembly language fundamentals |
| [**Executable Formats**](docs/en/docs/executable/) | ELF, PE, and other binary format internals |
| [**ICS (Industrial Control)**](docs/en/docs/ics/) | Industrial control system security |

### Pwn Deep Dives

| Topic | Link |
|-------|------|
| Stack Overflow (x86) | [Basic ROP](docs/en/docs/pwn/linux/user-mode/stackoverflow/x86/basic-rop.md) · [Medium ROP](docs/en/docs/pwn/linux/user-mode/stackoverflow/x86/medium-rop.md) · [Advanced ROP](docs/en/docs/pwn/linux/user-mode/stackoverflow/x86/advanced-rop/advanced-rop.md) |
| Stack Overflow (ARM/MIPS) | [ARM ROP](docs/en/docs/pwn/linux/user-mode/stackoverflow/arm/rop.md) · [MIPS ROP](docs/en/docs/pwn/linux/user-mode/stackoverflow/mips/rop.md) |
| Heap Exploitation | [Overview](docs/en/docs/pwn/linux/user-mode/heap/ptmalloc2/heap-overview.md) · [Heap Structure](docs/en/docs/pwn/linux/user-mode/heap/ptmalloc2/heap-structure.md) · [Use After Free](docs/en/docs/pwn/linux/user-mode/heap/ptmalloc2/use-after-free.md) |
| Heap Techniques | [House of Force](docs/en/docs/pwn/linux/user-mode/heap/ptmalloc2/house-of-force.md) · [House of Orange](docs/en/docs/pwn/linux/user-mode/heap/ptmalloc2/house-of-orange.md) · [House of Einherjar](docs/en/docs/pwn/linux/user-mode/heap/ptmalloc2/house-of-einherjar.md) · [Unlink](docs/en/docs/pwn/linux/user-mode/heap/ptmalloc2/unlink.md) |
| Format Strings | [Intro](docs/en/docs/pwn/linux/user-mode/fmtstr/fmtstr-intro.md) · [Exploitation](docs/en/docs/pwn/linux/user-mode/fmtstr/fmtstr-exploit.md) |

---

## 🚀 Quick Start

### Browse Online

The content lives under [`docs/en/docs/`](docs/en/docs/) — just open any `.md` file on GitHub to start reading.

### Build Locally

CTF Wiki uses [MkDocs](https://github.com/mkdocs/mkdocs) with the Material theme:

```shell
# 1. Clone this repo
git clone https://github.com/WildWestCyberSecurity/ctf-wiki-ENGLISH.git
cd ctf-wiki-ENGLISH

# 2. Install dependencies
pip install -r requirements.txt

# 3. Build the site
python3 scripts/docs.py build-all

# 4. Serve locally at http://127.0.0.1:8008
python3 scripts/docs.py serve
```

Or use Docker:

```shell
docker run -d --name=ctf-wiki -p 4100:80 ctfwiki/ctf-wiki
# Then open http://localhost:4100/
```

---

## 🏋️ How to Practice

1. **Read the wiki** — work through the topics that interest you
2. **Try the challenges** — all referenced challenges are in [ctf-challenges](https://github.com/ctf-wiki/ctf-challenges)
3. **Use the tools** — curated tool list at [ctf-tools](https://github.com/ctf-wiki/ctf-tools)

---

## 🤝 Contributing

Contributions are welcome! Please read the [Contributing Guide](docs/en/docs/contribute/before-contributing.md) before submitting changes.

---

## 💡 Tips for Beginners

- Learn to ask [smart questions](http://www.catb.org/~esr/faqs/smart-questions.html)
- Get comfortable with at least one programming language (Python is a great choice)
- Practice is everything — reading alone won't make you good at CTFs
- Stay curious and keep learning new techniques

---

## About This Fork

This is an **English-only fork** of the original [CTF Wiki](https://github.com/ctf-wiki/ctf-wiki) project, maintained by [WildWestCyberSecurity](https://github.com/WildWestCyberSecurity). The original project was primarily in Chinese — this fork provides translated English content to make these excellent resources accessible to the broader English-speaking security community.

**CTF Wiki** advocates **freedom of knowledge** and will always remain free, open-source, and non-commercial.
