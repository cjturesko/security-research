# Threat Analysis: "Sailors" PhaaS Kit / Government Agency Toll Lure

**Date:** March 2026  
**Sample Source:** SMS received on personal device  
**TLP:** White (suitable for public sharing)

---

## Executive Summary

A phishing SMS impersonating a US state motor vehicle agency was received claiming an outstanding toll charge of $6.90. Analysis of the linked site uncovered an Adversary-in-the-Middle (AiTM) phishing kit named "Sailors," sold as a Phishing-as-a-Service (PhaaS). Unlike typical phishing kits that just log stolen credentials, this one uses real-time WebSocket communication with AES-128-CBC encryption and live OTP relay, allowing the operator to actively complete fraudulent transactions while the victim is still on the page. Chinese-language strings embedded in the source point to a Chinese-speaking developer who likely sold access to a separate campaign operator.

---

## Initial Lure

| Field | Value |
|---|---|
| Delivery method | SMS (Smishing) |
| Sender number | Philippines-registered VoIP number (+63) |
| Lure text | Outstanding state toll charge of $6.90 |
| Phishing domain | `[target-agency].org-yhkjk.bond` |
| Infrastructure | Cloudflare CDN |

Using a Philippines-registered VoIP number to impersonate a US state government agency is an obvious operational security mistake on the operator's part. It points to the campaign operator being a separate person from the kit developer, which is typical of the PhaaS model.

---

## Kit Identification

The kit references itself internally as **"Sailors"** based on the following:

- localStorage key: `sailors_form_data`
- CSS class names: `sailors-router-view`, `sailors-span`
- Framework: Vue.js single-page application bundled with Vite
- Obfuscation: string rotation with base64 decoder pattern across multiple function families

### Authorship Indicators

Chinese strings were found embedded as internal UI state labels. These are not shown to victims but exist in the source code:

| Chinese | Translation |
|---|---|
| 支付页 | Payment page |
| 手机验证页 | Phone verification page |
| 邮箱验证页 | Email verification page |
| 完成页 | Completion page |
| 加密异常 | Encryption exception |
| 解密异常 | Decryption exception |

The combination of Chinese internal labels, kit sophistication, and PhaaS delivery model is consistent with Chinese-developed crimeware sold through Telegram or Chinese cybercrime forums.

---

## Technical Architecture

### AiTM vs. Traditional Phishing

This is not a simple credential harvester. The kit runs a full Adversary-in-the-Middle setup where a live operator receives victim data in real time and uses it on the real service before the session or OTP expires.

```
Victim fills form
      |
      v
JS encrypts data (AES-128-CBC)
      |
      v
WebSocket to operator dashboard (/console)
      |
      v
Operator receives OTP live, uses it on real bank/service
      |
      v
Victim sees "success" page
```

The attacker is not storing credentials to use later. They are actively using them within the 30 to 60 second OTP validity window to complete real transactions in real time.

### Command and Control

- **Protocol:** WebSocket (Socket.IO) upgraded from HTTPS
- **Path:** `/console`
- **Endpoint construction:** built from `window.location.host` at runtime with no hardcoded C2 domain in the source
- **Encryption:** AES-128-CBC with PKCS7 padding (16-byte key, confirmed static across multiple deployments)
- **Key storage:** assembled from obfuscated string fragments at runtime
- **Session tracking:** UUID generated per victim and passed as a WebSocket query parameter

Building the WebSocket URL from `window.location.host` at runtime means static analysis tools scanning the JS file in isolation will find no hardcoded domain. The file looks clean to automated scanners.

The AES key and IV were found hardcoded in the obfuscated source rather than being negotiated at runtime or generated per session. Analysis of a second independent sample from a different deployment domain confirmed the values are identical across both, verifying that the same compiled bundle is distributed to multiple operators without recompilation. Any Socket.IO traffic captured from a Sailors-based phishing page can be decrypted with the following static values:

| Parameter | Value |
|---|---|
| Algorithm | AES-128-CBC + PKCS7 |
| Key (variable `Re`) | `KIKOCCJCFILDLDND` |
| IV (variable `Oe`) | `OJGJHHFMJEJBDFAI` |

### Data Captured

The kit sends the following to the operator dashboard in sequence:

- Payment card details (number, expiry, CVV)
- SMS one-time passcodes
- Email one-time passcodes
- Authenticator app codes
- PIN numbers

### Bot Detection

The kit loads **FingerprintJS Botd** from `openfpcdn.io`. If the visitor is flagged as a bot or headless browser the kit does not run, which makes dynamic analysis harder without a genuine browser environment.

---

## Infrastructure Analysis

### Domain and Certificate Data

| Field | Value |
|---|---|
| Registered domain | `org-yhkjk.bond` |
| Phishing subdomain | `[target-agency].org-yhkjk.bond` |
| Certificate issued | March 2026 |
| Certificate type | Wildcard (`*.org-yhkjk.bond`) |
| CAs used | Let's Encrypt (x2) + Sectigo DV (x2) |
| CDN | Cloudflare (origin IP masked) |

Four certificates were issued on the same day across two certificate authorities, which points to automated provisioning rather than manual setup. The wildcard certificate covers any subdomain under the registered domain, so the operator can run multiple lures under different agency names without requesting additional certificates.

## Possible Related Infrastructure

Certificate Transparency log analysis via crt.sh identified a cluster of 21 domains sharing the same structural and provisioning pattern, all registered on 2026-03-04:

| Domain | Issued | CAs |
|---|---|---|
| org-tvp.bond | 2026-01-25 | Let's Encrypt + Sectigo DV |
| org-nyw.bond | 2026-01-25 | Let's Encrypt + Sectigo DV |
| org-yhkjk.bond | 2026-03-05 | Let's Encrypt + Sectigo DV |
| org-gqeeb.bond | 2026-03-04 | Let's Encrypt + Sectigo DV |
| org-gqrza.bond | 2026-03-04 | Let's Encrypt + Sectigo DV |
| org-gqtla.bond | 2026-03-04 | Let's Encrypt + Sectigo DV |
| org-gquia.bond | 2026-03-04 | Let's Encrypt + Sectigo DV |
| org-gquib.bond | 2026-03-04 | Let's Encrypt + Sectigo DV |
| org-gquic.bond | 2026-03-04 | Let's Encrypt + Sectigo DV |
| org-gqwxa.bond | 2026-03-04 | Let's Encrypt + Sectigo DV |
| org-gqyeb.bond | 2026-03-04 | Let's Encrypt + Sectigo DV |
| org-gqyga.bond | 2026-03-04 | Let's Encrypt + Sectigo DV |
| org-rhmas.bond | 2026-03-04 | Let's Encrypt + Sectigo DV |
| org-rhmer.bond | 2026-03-04 | Let's Encrypt + Sectigo DV |
| org-rhmio.bond | 2026-03-04 | Let's Encrypt + Sectigo DV |
| org-rhmpa.bond | 2026-03-04 | Let's Encrypt + Sectigo DV |
| org-rhmqw.bond | 2026-03-04 | Let's Encrypt + Sectigo DV |
| org-rhmrt.bond | 2026-03-04 | Let's Encrypt + Sectigo DV |
| org-rhmwe.bond | 2026-03-04 | Let's Encrypt + Sectigo DV |
| org-rhmyu.bond | 2026-03-04 | Let's Encrypt + Sectigo DV |
| org-tfupa.bond | 2026-03-04 | Let's Encrypt + Sectigo DV |
| org-tfuvb.bond | 2026-03-04 | Let's Encrypt + Sectigo DV |
| org-ykopa.bond | 2026-03-04 | Let's Encrypt + Sectigo DV |
| org-ykovb.bond | 2026-03-04 | Let's Encrypt + Sectigo DV |

Three separate registration dates, all producing the same org-[random].bond naming pattern and the same dual-CA cert provisioning. The January domains predate the analyzed sample by about six weeks and likely represent an earlier campaign wave. The March 4th cluster of 21 domains provisioned on a single day points to automated bulk registration rather than anything being set up by hand. At this volume, manual domain setup is not realistic.

Each domain received four certificates on registration day, two from Let's Encrypt and two from Sectigo DV E36, which is consistent with a hosting panel auto-provisioning certs across multiple CAs simultaneously. Every certificate SHA256 across the cluster is unique, meaning no certs were shared or reused between domains. That rules out a simple copy-paste setup and suggests each domain was independently provisioned, likely through the same automated process.

A source familiar with this campaign noted that over 1,000 domains matching this naming pattern may have been registered in the last 90 days. That figure has not been independently verified, but the provisioning consistency across the sample examined here makes it plausible. If accurate, the domain analyzed in this report is a small slice of an active, ongoing campaign. The January 2026 domains suggest the operator has been running this infrastructure for at least two months before the March cluster was registered.

### File Hashes (Sample 1 - 'org-yhkjk.bond')

| Algorithm | Hash |
|---|---|
| MD5 | `a6f2adaf6bd771efa29369e40ae0e7d8` |
| SHA1 | `8c5c409ade18be8dc6befc069d5300e974a2f6e6` |
| SHA256 | `6339e92bf4560087496817a41d7df9fd1b426373de6cb060e7e13352a46905f9` |

### File Hashes (Sample 2 - 'org-rfxpa.bond')

| Algorithm | Hash |
|---|---|
| SHA256 | `48749265e299afa4941b4412cde536d85201ea83f976d773c077c66580fa3cb3` |

---

## Pivot Analysis and Why It Came Up Empty

Pivoting was attempted across Shodan, URLscan.io, VirusTotal, FOFA, Censys, and crt.sh. Nothing came back. Here is why each avenue failed:

**Cloudflare beacon token** is unique per domain. If it were reused across deployments it would have been the most reliable pivot point, but each new domain gets its own token automatically.

**Kit strings** like `sailors_form_data` returned zero results across all platforms and search engines. The kit either had not been crawled yet or the operator kept public exposure to a minimum.

**No hardcoded C2** means there is no IP or domain sitting in the JS file for static analysis to pull out. The WebSocket URL is built at runtime so it does not exist in the code as a string.

**Hash not in VirusTotal** at time of analysis. The file had not been submitted anywhere publicly, which fits a targeted campaign rather than something blasted out at scale.

Getting nothing back across all platforms is itself a finding. This kit has better operational security than most commodity phishing tools and the zero results appear intentional.

---

## MITRE ATT&CK Mapping

| Observed Behavior | Tactic | Technique |
|---|---|---|
| SMS lure impersonating government agency | Initial Access | T1660 Phishing: Smishing |
| Fake payment form capturing card data | Collection | T1056 Input Capture |
| Live OTP relay to real service | Credential Access | T1557 Adversary-in-the-Middle |
| AES-128-CBC encrypted WebSocket channel | Command and Control | T1573 Encrypted Channel |
| WebSocket / Socket.IO C2 | Command and Control | T1071 Application Layer Protocol |
| Cloudflare CDN masking origin server | Defense Evasion | T1090 Proxy |
| Obfuscated JavaScript (string rotation) | Defense Evasion | T1027 Obfuscated Files or Information |
| FingerprintJS Botd anti-analysis check | Defense Evasion | T1497 Virtualization/Sandbox Evasion |
| Runtime URL construction with no hardcoded C2 | Defense Evasion | T1027 Command Obfuscation |

---

## Takeaways

**For defenders:**

AiTM kits bypass standard MFA completely. SMS codes, email codes, and authenticator app codes can all be relayed in real time. The only MFA that cannot be relayed this way is a hardware security key using FIDO2 or WebAuthn. If your threat model includes targeted phishing, hardware keys are the only reliable protection.

No Telegram API calls in network traffic does not mean the kit is less capable. It may mean the operator channel is more sophisticated and harder to detect.

A newly registered domain on an uncommon TLD like `.bond`, `.top`, or `.xyz` sitting behind Cloudflare and impersonating a government or utility service is a reliable pattern for this type of attack.

**For threat intel:**

Certificate Transparency logs via crt.sh are still useful even when every paid platform comes up empty. In this case crt.sh confirmed the provisioning date and surfaced a cluster of related domains that nothing else found. The pattern was consistent enough across 24 domains that the infrastructure relationship is more likely than not, even without direct code or credential overlap to confirm it.

When a kit shows clean OPSEC on infrastructure but sloppy delivery via a foreign phone number, that gap usually means the kit developer and the campaign operator are two different people. The developer built something technically solid and sold access to someone who did not put the same care into the delivery side.

Zero pivot results across multiple platforms on a live kit is not a dead end. It is a data point. It means either the kit is very recently deployed or someone was deliberate about keeping it off public infrastructure scanners.

---

## Indicators of Compromise

| Type | Value |
|---|---|
| Domain (analyzed) | org-yhkjk.bond |
| Domain (related, Jan 2026) | org-tvp.bond |
| Domain (related, Jan 2026) | org-nyw.bond |
| Domain (related, Mar 2026) | org-gqeeb.bond |
| Domain (related, Mar 2026) | org-gqrza.bond |
| Domain (related, Mar 2026) | org-gqtla.bond |
| Domain (related, Mar 2026) | org-gquia.bond |
| Domain (related, Mar 2026) | org-gquib.bond |
| Domain (related, Mar 2026) | org-gquic.bond |
| Domain (related, Mar 2026) | org-gqwxa.bond |
| Domain (related, Mar 2026) | org-gqyeb.bond |
| Domain (related, Mar 2026) | org-gqyga.bond |
| Domain (related, Mar 2026) | org-rhmas.bond |
| Domain (related, Mar 2026) | org-rhmer.bond |
| Domain (related, Mar 2026) | org-rhmio.bond |
| Domain (related, Mar 2026) | org-rhmpa.bond |
| Domain (related, Mar 2026) | org-rhmqw.bond |
| Domain (related, Mar 2026) | org-rhmrt.bond |
| Domain (related, Mar 2026) | org-rhmwe.bond |
| Domain (related, Mar 2026) | org-rhmyu.bond |
| Domain (related, Mar 2026) | org-tfupa.bond |
| Domain (related, Mar 2026) | org-tfuvb.bond |
| Domain (related, Mar 2026) | org-ykopa.bond |
| Domain (related, Mar 2026) | org-ykovb.bond |
| JS SHA256 | 6339e92bf4560087496817a41d7df9fd1b426373de6cb060e7e13352a46905f9 |
| JS MD5 | a6f2adaf6bd771efa29369e40ae0e7d8 |
| Cloudflare beacon | 71fabe53af6b4350a6c4c1e7459c0adf |
| WebSocket path | /console |
| localStorage key | sailors_form_data |

Note: Four additional domains from the March 2026 cluster returned no crt.sh results at time of analysis (org-xaska.bond, org-xasla.bond, org-xaslb.bond, org-gqela.bond) and are omitted from the IOC table pending verification.

---

*Analysis conducted March 2026. Sample obtained via unsolicited SMS on personal device. No systems were compromised during analysis.*

---
## Updates

**March 22, 2026 — Second sample analysis and corrections**

- Obtained a second sample from a different deployment domain (`org-rfxpa.bond`). Reverse engineering confirmed the AES key and IV are identical across both samples, verifying that the same compiled bundle is distributed to multiple operators without recompilation. Key/IV values and second sample hash added.
- Corrected AES-256-CBC to AES-128-CBC throughout. The extracted key is 16 bytes (128-bit). The original description was incorrect.

