<div align="center">

# 🛡 Named Extended ACL: One Host, One Web Server, HTTP and HTTPS

### ACL Series · Lab 4 · Cisco Packet Tracer

![Cisco](https://img.shields.io/badge/Cisco-IOS-1BA0D7?style=for-the-badge&logo=cisco&logoColor=white)
![Packet Tracer](https://img.shields.io/badge/Packet%20Tracer-Lab-049fd9?style=for-the-badge)
![ACL](https://img.shields.io/badge/Extended-ACL-red?style=for-the-badge)
![Named](https://img.shields.io/badge/Named-ACL-purple?style=for-the-badge)
![Sequence](https://img.shields.io/badge/Sequence-Numbers-yellow?style=for-the-badge)
![Direction](https://img.shields.io/badge/Direction-Outbound-orange?style=for-the-badge)
![Level](https://img.shields.io/badge/Level-CCNA-green?style=for-the-badge)

*Match the source AND the destination AND the protocol AND the port.*

</div>

---

## 📑 Table of Contents
1. [Overview](#-overview)
2. [Topology](#-topology)
3. [Addressing Table](#-addressing-table)
4. [Standard vs Extended ACL](#-standard-vs-extended-acl)
5. [Server Preparation](#-server-preparation)
6. [Configuration](#-configuration)
7. [ACL Logic: Line by Line](#-acl-logic-line-by-line)
8. [Why Sequence 15?](#-why-sequence-15)
9. [Verification: Stage by Stage](#-verification-stage-by-stage)
10. [Test Matrix](#-test-matrix)
11. [Placement Logic](#-placement-logic-why-r1-serial-outbound)
12. [Key Takeaways](#-key-takeaways)
13. [Troubleshooting Notes](#-troubleshooting-notes)
14. [Next Labs](#-next-labs-in-the-acl-series)
15. [Author](#-author)

---

## 🎯 Overview

Labs 1–3 used **Standard ACLs**, which only see the source IP. This lab introduces the **Extended ACL**, which can also match the **destination**, **protocol** and **port**.

**Goal:** Only PC1 (`192.168.1.3`) may use the web server at `200.100.100.2`. Every other host is blocked from it, but **all other traffic** (ping, Telnet and so on) must keep working.

The lab runs in two phases:

| Phase | What happens |
|---|---|
| **1. Build** | A Named Extended ACL lets PC1 use **HTTP (80)** and denies everyone else HTTP and HTTPS |
| **2. Edit** | PC1 also needs **HTTPS (443)**, so a line is inserted with **sequence 15** |

| Item | Detail |
|---|---|
| ACL type | **Extended**, **named** (`cisco`) |
| Applied on | R1, Serial0/0/0 |
| Direction | **Outbound** |
| Protected service | HTTP (TCP 80) and HTTPS (TCP 443) on 200.100.100.2 |
| Only permitted web client | 192.168.1.3 (PC1) |
| Everything else | Permitted (`permit ip any any`) |

---

## 🗺 Topology

```
                  ACL OUT SIDE ──► (ip access-group cisco out)
                         1.0.0.0 (Serial WAN)
        1.1.1.1  ●━━━━━━━━━━━━━━⚡━━━━━━━━━━━━━━●  1.1.1.2
              ┌──────┐                          ┌──────┐
              │  R1  │                          │  R2  │
              │ 1941 │                          │ 1941 │
              └──┬───┘                          └──┬───┘
        192.168.1.1/24                       200.100.100.1/24
                 │                                 │
            ┌────┴────┐                       ┌────┴────┐
            │ Switch0 │                       │ Switch1 │
            └─┬──┬──┬─┘                       └─┬──┬──┬─┘
              │  │  │                           │  │  │
           PC0  PC1  PC2                       PC    PC   Server-PT
      192.168.1.2 .1.3 .1.4              200.100.100.4 .100.3  200.100.100.2
       (🚫 web)  (✅ web) (🚫 web)                          🌐 HTTP + HTTPS
```

---

## 📋 Addressing Table

| Device | Interface | IP Address | Role |
|---|---|---|---|
| R1 (1941) | LAN interface | 192.168.1.1/24 | Gateway, LAN 1 |
| R1 (1941) | Serial0/0/0 | 1.1.1.1 | WAN link (**ACL applied here**) |
| R2 (1941) | Serial | 1.1.1.2 | WAN link |
| R2 (1941) | LAN interface | 200.100.100.1/24 | Gateway, LAN 2 |
| PC0 | NIC | 192.168.1.2/24 | LAN 1, 🚫 web denied |
| **PC1** | NIC | **192.168.1.3/24** | ✅ **Only web-permitted host** |
| PC2 | NIC | 192.168.1.4/24 | LAN 1, 🚫 web denied |
| PC | NIC | 200.100.100.4/24 | LAN 2 |
| PC | NIC | 200.100.100.3/24 | LAN 2 |
| **Server-PT** | NIC | **200.100.100.2/24** | 🌐 **Protected web server** |

---

## 🔄 Standard vs Extended ACL

| | Standard ACL (Labs 1–3) | **Extended ACL (this lab)** |
|---|---|---|
| Matches on | Source IP only | **Source + destination + protocol + port** |
| Number ranges | 1–99, 1300–1999 | 100–199, 2000–2699 |
| Granularity | Whole host | **One service on one host** |
| Best placement | Close to the **destination** | Close to the **source** |
| Example | `permit host 192.168.1.3` | `permit tcp host 192.168.1.3 host 200.100.100.2 eq 80` |

---

## 🖥 Server Preparation

On Server-PT, the **Services** tab has **HTTP: On** and **HTTPS: On**. The files served are `index.html`, `helloworld.html`, `copyrights.html`, `image.html` and the Cisco logo.

| Service | Port | Status | ACL line that controls it |
|---|---|---|---|
| HTTP | TCP 80 | ✅ On | `eq 80` lines |
| HTTPS | TCP 443 | ✅ On | `eq 443` lines |

---

## ⚙️ Configuration

### R1: remove the leftover ACL from Lab 1 (if still present)
> A leftover inbound ACL on the serial interface would block the server's replies.

```
R1(config)# interface serial 0/0/0
R1(config-if)# no ip access-group 1 in
R1(config-if)# exit
```

### Phase 1: build the Named Extended ACL

```
R1(config)# ip access-list extended cisco
R1(config-ext-nacl)# permit tcp host 192.168.1.3 host 200.100.100.2 eq 80
R1(config-ext-nacl)# deny tcp any host 200.100.100.2 eq 80
R1(config-ext-nacl)# deny tcp any host 200.100.100.2 eq 443
R1(config-ext-nacl)# permit ip any any
R1(config-ext-nacl)# exit

R1(config)# interface serial 0/0/0
R1(config-if)# ip access-group cisco out
R1(config-if)# end
```

### Phase 2: insert sequence 15 so PC1 can use HTTPS

```
R1(config)# ip access-list extended cisco
R1(config-ext-nacl)# 15 permit tcp host 192.168.1.3 host 200.100.100.2 eq 443
R1(config-ext-nacl)# end
R1# write memory
```

### Routing between the two LANs
> Replace this block with the routing you actually used.

```
R1(config)# ip route 200.100.100.0 255.255.255.0 1.1.1.2
R2(config)# ip route 192.168.1.0   255.255.255.0 1.1.1.1
```

### Final ACL

```
Extended IP access list cisco
    10 permit tcp host 192.168.1.3 host 200.100.100.2 eq www
    15 permit tcp host 192.168.1.3 host 200.100.100.2 eq 443
    20 deny tcp any host 200.100.100.2 eq www
    30 deny tcp any host 200.100.100.2 eq 443
    40 permit ip any any
```

> IOS displays `eq 80` as `eq www`. They are the same.

---

## 🧩 ACL Logic: Line by Line

| Seq | Rule | Who it matches | Action |
|---|---|---|---|
| 10 | `permit tcp host 192.168.1.3 host 200.100.100.2 eq 80` | PC1's HTTP requests | ✅ Permit |
| **15** | `permit tcp host 192.168.1.3 host 200.100.100.2 eq 443` | PC1's HTTPS requests (**added in Phase 2**) | ✅ Permit |
| 20 | `deny tcp any host 200.100.100.2 eq 80` | Everyone else's HTTP | ❌ Deny |
| 30 | `deny tcp any host 200.100.100.2 eq 443` | Everyone else's HTTPS | ❌ Deny |
| 40 | `permit ip any any` | Everything else (ICMP, Telnet and so on) | ✅ Permit |

```
 PC1 ──HTTP──►  line 10 matches ──► ✅ page loads
 PC1 ──HTTPS─►  10 no ► line 15 matches ──► ✅
 PC0 ──HTTP──►  10 no ► 15 no ► line 20 matches ──► ❌ timeout
 PC0 ──ping──►  10 no ► 15 no ► 20 no ► 30 no ► line 40 matches ──► ✅
```

| Order mistake | Result |
|---|---|
| Deny **before** the permit | ❌ PC1 is blocked too, because the deny matches first |
| No `permit ip any any` at the end | ❌ The implicit `deny any` kills ping and Telnet too |

---

## 🔢 Why Sequence 15?

ACLs are read **top to bottom, first match wins**. Where a new line lands decides whether it works.

| How the line is added | Where it lands | Result |
|---|---|---|
| `permit tcp host ... eq 443` (**no number**) | After 40, as seq 50 | ❌ PC1's HTTPS hits **line 30 (deny)** first, so the new line is never reached |
| `15 permit tcp host ... eq 443` | Between 10 and 20 | ✅ Matches before line 30, so PC1's HTTPS is permitted |

The gaps in numbering (10, 20, 30, 40) exist so lines can be inserted later. Any number from 11 to 29 would work here.

> If you run out of gaps: `ip access-list resequence cisco 10 10` renumbers the ACL in steps of 10.

### Named vs numbered ACLs

| | Numbered | **Named (this lab)** |
|---|---|---|
| Create | `access-list 101 permit ...` | `ip access-list extended cisco` |
| Readability | Just a number | **Descriptive name** |
| Editing a line | Hard, often rebuild | **Insert, delete or replace by sequence number** |

---

## ✅ Verification: Stage by Stage

## Phase 1: HTTP policy

### 🔴 Stage 1: PC0 (192.168.1.2) → web server over HTTP → BLOCKED

```
URL: http://200.100.100.2
→ Request Timeout
```
**Annotations**
- Source 192.168.1.2 does **not** match the PC1 permit, so it falls to the deny on line 20.
- The browser shows a **timeout** because the HTTP request never completes.

### 🔴 Stage 2: PC2 (192.168.1.4) → web server over HTTP → BLOCKED

```
URL: http://200.100.100.2
→ Request Timeout
```
**Annotation:** The same result as PC0. Only the exact host in the permit line gets through.

### 🟢 Stage 3: PC1 (192.168.1.3) → web server over HTTP → ALLOWED

```
URL: http://200.100.100.2
→ Page loads:
   "Maaz Khan"
   "Welcome to Cisco Packet Tracer. Opening doors to new opportunities. Mind Wide Open."
   Quick Links: A small page · Copyrights · Image page · Image
```
**Annotation:** Source 192.168.1.3 + destination 200.100.100.2 + TCP 80 matches **line 10**. This is the Extended ACL working: **source AND destination AND port**.

### 🟢 Stage 4: PC0 → ping the same server → ALLOWED

```
C:\> ping 200.100.100.2

Reply from 200.100.100.2: bytes=32 time=1ms TTL=126  (x4)

Packets: Sent = 4, Received = 4, Lost = 0 (0% loss)
Minimum = 1ms, Maximum = 1ms, Average = 1ms
```
**Annotations**
- The **same PC** that cannot browse the server can still ping it. ICMP is not TCP 80, so it reaches `permit ip any any`.
- **TTL = 126**: 128 minus 2 routers.

### 🟢 Stage 5: PC0 → Telnet to R2 → ALLOWED

```
C:\> telnet 1.1.1.2
Trying 1.1.1.2 ...Open

User Access Verification

Password:
R2>en
Password:
R2#
```
**Annotation:** Telnet is TCP 23, so it is not matched by the web deny lines and passes through line 40. **Only web access to that server is restricted.**

### 📊 Stage 6: Counters at the end of Phase 1 (run on R1)

```
R1# show access-lists
Extended IP access list cisco
    10 permit tcp host 192.168.1.3 host 200.100.100.2 eq www (10 match(es))
    20 deny tcp any host 200.100.100.2 eq www (87 match(es))
    30 deny tcp any host 200.100.100.2 eq 443
    40 permit ip any any (154 match(es))
```

| Line | Matches | What it shows |
|---|---|---|
| 10 | 10 | PC1's web sessions (permitted) |
| 20 | **87** | PC0 and PC2 browser attempts and retries (denied) |
| 30 | none | **No HTTPS test yet**, which leads to Phase 2 |
| 40 | 154 | Ping, Telnet and all other permitted traffic |

---

## Phase 2: add HTTPS for PC1

### 🔧 Stage 7: Insert sequence 15

```
R1(config)# ip access-list extended cisco
R1(config-ext-nacl)# 15 permit tcp host 192.168.1.3 host 200.100.100.2 eq 443
R1(config-ext-nacl)# end
```

### 🟢 Stage 8: PC1 → web server over HTTPS → ALLOWED

```
URL: https://200.100.100.2
→ (paste your screenshot here: the page should load)
```
**Annotation:** TCP 443 from PC1 now matches **line 15** before it can reach the deny on line 30.

### 🔴 Stage 9: PC0 / PC2 → web server over HTTPS → BLOCKED

```
URL: https://200.100.100.2
→ (paste your screenshot here: expected Request Timeout)
```
**Annotation:** These sources skip lines 10 and 15 (not PC1) and match **line 30**.

### 📊 Stage 10: Counters after the edit (run on R1)

```
R1# show access-lists
Extended IP access list cisco
    10 permit tcp host 192.168.1.3 host 200.100.100.2 eq www (x match(es))
    15 permit tcp host 192.168.1.3 host 200.100.100.2 eq 443 (x match(es))
    20 deny tcp any host 200.100.100.2 eq www (x match(es))
    30 deny tcp any host 200.100.100.2 eq 443 (x match(es))
    40 permit ip any any (x match(es))
```
> 📸 Paste your own counters. Line 15 should increase after PC1's HTTPS test, and line 30 after PC0's.

---

## 🧪 Test Matrix

| # | Source | Test | Result | Line |
|---|---|---|---|---|
| 1 | PC1 (192.168.1.3) | HTTP to web server | ✅ Page loads | 10 |
| 2 | PC1 (192.168.1.3) | HTTPS to web server | ✅ Expected (add screenshot) | **15** |
| 3 | PC0 (192.168.1.2) | HTTP to web server | ❌ Request Timeout | 20 |
| 4 | PC2 (192.168.1.4) | HTTP to web server | ❌ Request Timeout | 20 |
| 5 | PC0 / PC2 | HTTPS to web server | ❌ Expected (add screenshot) | 30 |
| 6 | PC0 (192.168.1.2) | Ping the server | ✅ 4/4, TTL 126 | 40 |
| 7 | PC0 (192.168.1.2) | Telnet to R2 | ✅ R2# | 40 |

---

## 📍 Placement Logic: Why R1 Serial Outbound?

| ACL type | Where to place it | Why |
|---|---|---|
| Standard | Close to the **destination** | It only sees the source, so it would over-block near the source |
| **Extended** | Close to the **source** | It can name the exact destination and port, so unwanted traffic is stopped **before it crosses the WAN** |

```
 PC0 ──HTTP──► R1 ──► [ACL cisco OUT on Se0/0/0] ──✖ dropped here
 PC1 ──HTTP──► R1 ──► [ACL cisco OUT on Se0/0/0] ──✔──► R2 ──► Web server
```

- One ACL on R1's serial interface covers **every** host in LAN 1 at a single point.
- The server's **reply** comes back **inbound** on Serial0/0/0. This ACL is `out` only, so replies are not filtered.

---

## 💡 Key Takeaways

| # | Takeaway |
|---|---|
| 1 | Extended ACLs match **source, destination, protocol and port** |
| 2 | Place Extended ACLs close to the **source**, and Standard ACLs close to the destination |
| 3 | Named ACLs can be **edited in place** with sequence numbers |
| 4 | A new line without a number goes to the **bottom** and may never be matched |
| 5 | Insert permits **above** the denies they must override |
| 6 | HTTP is TCP 80 and HTTPS is TCP 443, and the server needs both services On |
| 7 | Without the final `permit ip any any`, the implicit deny breaks ping and Telnet |
| 8 | `show access-lists` counters show which line handled which traffic |

---

## 🛠 Troubleshooting Notes

| Symptom | Check |
|---|---|
| PC1 cannot browse either | Is the permit line **above** the deny? Is the server's HTTP service On? |
| PC1's HTTPS still fails | Is line 15 numbered **below 30**? Is the server's HTTPS service On? |
| New line appears at the bottom | You forgot the sequence number |
| Everyone can browse | Is the ACL applied outbound on Serial0/0/0? |
| Ping and Telnet also fail | Is `permit ip any any` missing at the end? |
| Counters stay at 0 | Is the traffic really leaving R1 through that interface? |

---

## 🚀 Next Labs in the ACL Series

- [x] Lab 1: Standard ACL, whitelist a host ✅
- [x] Lab 2: Standard ACL, blacklist a host with placement logic ✅
- [x] Lab 3: Standard ACL, restrict Telnet (VTY) to one admin PC ✅
- [x] Lab 4: Named Extended ACL, one host to a web server over HTTP and HTTPS ✅
- [ ] Extended ACL with ICMP and port ranges
- [ ] Extended ACL with the `established` keyword
- [ ] Reflexive / time-based ACLs

---

## 👨‍💻 Author

**Maaz Khan**
CCNA-certified Network & NOC Engineer · Lower Dir, KPK, Pakistan

[![LinkedIn](https://img.shields.io/badge/LinkedIn-maazkhanms-0A66C2?style=for-the-badge&logo=linkedin)](https://www.linkedin.com/in/maazkhanms)
[![GitHub](https://img.shields.io/badge/GitHub-maazkhanms-181717?style=for-the-badge&logo=github)](https://github.com/maazkhanms)

⭐ If this lab helped you, give the repo a star!
