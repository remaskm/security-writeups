# Security Writeups

Full-cycle penetration test reports and offensive-security lab exercises. Every target here is either a public CTF/training VM (VulnHub, Metasploitable) or a self-built training lab — nothing here touches a real client or production system.

## Penetration Tests

| Writeup | Target | Summary |
|---|---|---|
| [Matrix-Breakout: Morpheus](./morpheus-pentest.md) | VulnHub boot2root | Stored XSS → unauthenticated file-write RCE → DirtyPipe (CVE-2022-0847) kernel privesc → root, with SSH persistence and exfiltration demonstrated |
| [Metasploitable 2](./metasploitable2-pentest.md) | Metasploitable 2 | Full lifecycle test — recon, Nessus scanning, exploitation of two independent backdoored services (vsftpd, UnrealIRCd), mitigation, and honest retest |

## Lab Exercises

| Writeup | Topic |
|---|---|
| [DoS Attacks & Mitigations](./dos-attacks-mitigation.md) | UDP flood, measured DNS amplification (12.78×), SYN flood — with before/after mitigation metrics (SYN cookies + iptables rate-limiting) |

---

For the security tools I've built (VaultScan, Redhook, Six Eyes) and my privilege-escalation deep-dive ([Chroot Jail Escape](https://github.com/remaskm/Chroot-Jail-Escape-Write-up)), see my [profile](https://github.com/remaskm).
