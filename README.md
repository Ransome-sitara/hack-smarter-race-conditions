# Hack Smarter — Race Conditions Lab (Medium)

Penetration test report and full writeup for the [Hack Smarter](https://www.hacksmarter.org/) Race Conditions Lab (Medium difficulty).

Featured on Hack Smarter's [Community Walkthroughs](https://www.hacksmarter.org/) page as **Race Conditions - Daniel Etim**.

This repository contains the complete engagement documentation — five race condition exploits covering web applications, APIs, and operating system privilege escalation.

---

## Overview

The lab simulates a banking and e-commerce environment:

- **MWR Bank** — banking web application with transaction and login features
- **MWR MoMo** — mobile wallet application
- **MWR Swag** — meme/merch shop integrated with the wallet
- **Ubuntu 24.04 target** — privilege escalation challenge

Five race condition challenges were exploited end to end. Every finding includes reproduction steps, tooling, evidence screenshots, and remediation guidance.

---

## Findings Summary

| # | Challenge | Technique | Severity | Status |
|---|---|---|---|---|
| 1 | Transaction Limits | HTTP single-packet race | High | Solved |
| 2 | Account Login Limit | HTTP/2 race vs. lockout counter | High | Solved |
| 3 | Mo Money Mo Problems | Cross-system race (Bank → MoMo) | Critical | Solved |
| 4 | Shopping Cart Full of Wonder | Cross-endpoint race (checkout + add) | Critical | Solved |
| 5 | Local Privilege Escalation | Filesystem TOCTOU | Critical | Solved |

---

## Highlights

### Race 2 — Login Lockout Bypass

The login endpoint enforced a permanent lockout after N failed password attempts using a TOCTOU-vulnerable counter. By firing 29 password candidates in a **single atomic HTTP/2 packet** using `httpx`, the entire wordlist reached the server before any single attempt incremented the counter.

```python
async with httpx.AsyncClient(http2=True) as client:
    tasks = [client.post(URL, content=body, headers=headers)
             for pwd in candidates]
    responses = await asyncio.gather(*tasks)

```
The target account was identified by reverse-engineering the lab's password pattern from a leaked user list — pw_Violet!<NN> where NN = position + 1.
Race 4 — Cross-Endpoint Checkout Race

Two endpoints (POST /cart for checkout, POST /product for add-to-cart) operated on the same cart without a row-level lock. Racing them in a single burst caused the cart to ship with more items than the amount debited. The attack required two browser tabs — one to hold the checkout request, one to keep adding products.
Race 5 — Filesystem TOCTOU

A root-owned Python updater script ran via sudo NOPASSWD with a classic TOCTOU pattern: hash verification, a ~150ms delay loop, then a file copy. By swapping the source file during the delay, an attacker-controlled payload replaced the root-owned script. The next sudo execution ran the payload as root.
bash

# Race window: hash check → ~150ms delay → file copy
sudo /lpe/script.py &                    # start in background
sleep 0.08                               # land inside the delay loop
cat payload.py > /lpe/tmp/script.py      # atomic overwrite
wait                                     # sudo copies our payload as root
sudo /lpe/script.py                      # executes payload as root

Repository Structure
text

.
├── README.md                          # This file
├── report.md                          # Full pentest report (markdown)
├── Daniel_Etim_Race_Conditions_Report.pdf  # Rendered PDF report
├── LICENSE                            # MIT License
└── evidence/                          # All screenshots referenced in the report
    ├── Race2_Flag.png
    ├── Race2_Violet_account.png
    ├── Race2_understandingthe_abituaryLimit.png
    ├── Race2_testing_Account.png
    ├── Race2_token.png
    ├── Race2_Faild.png
    ├── Race3_Flag.png
    ├── Race4_Flag.png
    ├── Race5_Flag.png
    ├── Race5_Script1.png
    ├── Race5_Script-continuation.png
    ├── RACE1-FLAG.png
    ├── RACE1-CORRECT.png
    ├── Race-Lab1-Solved.png
    ├── Race-Lab2-solved.png
    ├── Race-Lab3-solved.png
    ├── Race-La4-solved.png
    └── Race-Lab5-Solved.png

# Report

The full pentest report is available in two formats:

  report.md — Markdown source with inline evidence

  Daniel_Etim_Race_Conditions_Report.pdf — Rendered PDF with embedded screenshots

Rebuilding the PDF

Requires pandoc and a LaTeX distribution (texlive-latex-extra, texlive-xetex).
bash
```
cd ~/home/RACE
pandoc report.md \
    -o "Daniel_Etim_Race_Conditions_Report.pdf" \
    --pdf-engine=xelatex \
    --toc --toc-depth=3 --number-sections \
    -V geometry:margin=1in -V fontsize=11pt \
    -V colorlinks=true -V linkcolor=blue -V urlcolor=blue \
    -V documentclass=report \
    -V mainfont="DejaVu Sans" -V monofont="DejaVu Sans Mono" \
    --resource-path=.:evidence
```
# Tooling
Tool	Purpose
Burp Suite Community	HTTP interception, Repeater, last-byte sync
Python 3.14 + httpx[http2]	HTTP/2 single-packet race attacks
curl	Manual API probing
OpenVPN	Lab connectivity
JWT.io	Token decoding
Firefox DevTools	Network capture, JS bundle analysis
SSH	Target access for Race 5
Key Techniques
HTTP/2 Single-Packet Attack

Traditional parallel HTTP/1.1 requests open multiple TCP connections. Even with ThreadPoolExecutor, each connection introduces milliseconds of jitter — enough time for a TOCTOU window to close.

HTTP/2 multiplexing solves this by packing multiple requests into one TCP segment. The server receives all requests in the same event loop iteration — no state changes between them.
Burp Repeater Last-Byte Sync

Burp's "Send group (parallel) → Last-byte sync" is the manual equivalent. All grouped requests are held at the TCP layer, and only the final byte of each is released at the same millisecond.
Filesystem TOCTOU with Shebang Awareness

When replacing an executable script, the payload must include the shebang (#!/usr/bin/python3) or the kernel will invoke it with /bin/sh, causing a parse error. This was a lesson learned the hard way during Race 5.
Remediation Themes

Across all five findings, the underlying fix is the same:

  Atomic operations — no check-then-act patterns without a lock

  Row-level locking — SELECT ... FOR UPDATE for state-changing endpoints

  Idempotency keys — prevent replay of the same logical operation

  Rate limiting — at both network and application layers

  Never trust user-writable paths in privileged operations

# Full remediation per finding is documented in the report.
MITRE ATT&CK Mapping
Finding	Tactic	Technique
Race 1	Initial Access	T1190 — Exploit Public-Facing Application
Race 2	Credential Access	T1110.001 — Password Guessing
Race 3	Initial Access	T1190 — Exploit Public-Facing Application
Race 4	Initial Access	T1190 — Exploit Public-Facing Application
Race 5	Privilege Escalation	T1548.003 — Sudo and Sudo Caching
Disclaimer

This repository documents a controlled lab engagement on the Hack Smarter training platform. All testing was performed within the lab's provided scope and Rules of Engagement.

Flags and target credentials have been redacted in the report and screenshots to preserve the integrity of the lab for future learners.

Do not use these techniques against systems you do not own or have explicit written permission to test.
Credits

  Lab: Hack Smarter — Race Conditions (Medium)

  Report Author: Daniel Etim

   Engagement Dates: October 7–12, 2026
