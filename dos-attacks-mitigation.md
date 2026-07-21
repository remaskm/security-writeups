# Denial-of-Service Attacks & Mitigations — Lab Report

📄 [Full report with screenshots (PDF)](./reports/dos-attacks-report.pdf)

**Environment:** Kali Linux (attacker) → Metasploitable 2 (victim), isolated host-only/NAT lab
**Tools:** hping3, Wireshark, iptables, sysctl, netstat, tcpdump, dig

> Three DoS attack classes were executed and measured, then layered kernel-level mitigations were applied and validated with **before/after metrics** — not just "it feels more secure."

---

## Summary

| Attack | What it exploits | Result |
|---|---|---|
| UDP flood (port 53) | Kernel processes UDP with no rejection cost | Confirmed via Wireshark packet spike + `netstat -su` receive errors |
| DNS amplification | Legitimate open resolvers reflect + multiply traffic | Measured a real **12.78× amplification factor** |
| SYN flood (port 80) | Finite memory for half-open TCP connections | SYN_RECV queue dropped 256 → 198 after mitigation; iptables actively DROPed 1,459 packets mid-flood |

## 1. UDP Flood (Port 53)

```bash
ping -c 3 <victim>                     # confirm connectivity
sudo wireshark &                       # capture traffic, filter: udp port 53
sudo hping3 --udp -p 53 --flood -V <victim>
sudo netstat -su                       # check receive errors on victim
```

**Observation:** Wireshark showed an immediate, sustained spike in UDP packets targeting port 53. `netstat -su` on the victim confirmed UDP receive errors incrementing rapidly — the kernel's UDP buffer was being overwhelmed by connectionless traffic that carries no cost to send and no handshake to reject.

## 2. DNS Amplification — Measuring the Real Factor

Rather than just describing amplification theoretically, the actual factor was measured against a live public resolver:

```bash
dig +stats ANY google.com @8.8.8.8
```

| Measurement | Value |
|---|---|
| DNS query size | 107 bytes |
| DNS response size | 1,367 bytes |
| **Amplification factor** | **1367 / 107 ≈ 12.78×** |

This was then simulated against the lab victim by sending DNS-sized (60-byte) UDP payloads:

```bash
sudo hping3 --udp -p 53 --flood -d 60 <victim>
```

**Why spoofed UDP makes this effective:** UDP has no three-way handshake, so the receiving resolver never verifies the source IP. An attacker spoofs the victim's IP as the query source — the resolver's (larger) response goes straight to the victim instead of the attacker. At scale, thousands of spoofed queries to open resolvers can generate hundreds of Gbps aimed at a single target while costing the attacker a fraction of that bandwidth. This is why DNS amplification remains one of the most damaging DDoS vectors observed in the wild, and why BCP38 source-address validation at the ISP edge exists.

## 3. SYN Flood + Layered Mitigation

**Baseline (before mitigation):**

```bash
sysctl net.ipv4.tcp_syncookies          # → 0 (disabled)
sudo hping3 -S --flood -V -p 80 <victim>
watch -n 1 'netstat -an | grep SYN_RECV | wc -l'
```

**Mitigation 1 — SYN cookies** (lets the kernel handle SYN requests without allocating memory per half-open connection):

```bash
sudo sysctl -w net.ipv4.tcp_syncookies=1
echo 'net.ipv4.tcp_syncookies = 1' | sudo tee -a /etc/sysctl.conf
```

**Mitigation 2 — iptables rate-limiting** (drops excess SYN packets before they consume kernel state):

```bash
sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A INPUT -p tcp --syn -m limit --limit 25/sec --limit-burst 50 -j DROP
sudo iptables -A INPUT -p tcp --syn -j LOG --log-prefix 'SYN-FLOOD: '
```

### Before vs. After

| Metric | Before | After |
|---|---|---|
| SYN_RECV queue (peak) | 256 | 198 |
| iptables DROP counter | N/A | 1,459 packets |
| iptables LOG counter | N/A | ~671,000 packets |
| SYN cookies status | 0 (disabled) | 1 (enabled) |

**Analysis:** The SYN_RECV reduction looks modest at first glance (256 → 198), but the supporting counters tell the real story: the iptables DROP rule was actively intercepting **1,459 SYN packets mid-flood**, and the LOG rule recorded **~671,000 total flood packets** hitting the interface — confirming the true scale of what the victim was absorbing. SYN cookies and iptables are complementary, not redundant: iptables limits the *rate* of new SYN packets reaching the kernel, while SYN cookies make the *remaining* traffic survivable by removing the per-connection memory cost.

## Conclusion

Each DoS class exploits a different, fundamental property of the protocols involved:

- **UDP flood** — cheap to execute because UDP gives the kernel no mechanism to reject packets before processing them.
- **DNS amplification** — dangerous because it weaponizes legitimate infrastructure; open resolvers are doing exactly what they're designed to do, just against an unwilling target.
- **SYN flood** — exploits the memory cost TCP must pay to track half-open connections, a property that can't be removed without changing the protocol itself.

The mitigations applied here (SYN cookies, iptables rate-limiting) are software-based, single-host defenses — a realistic first layer, but production environments add BCP38 filtering at the ISP edge, anycast routing to spread flood traffic across data centers, and stateful hardware firewalls operating at line rate. No single defense covers every vector, which is why real-world DDoS protection is always layered.
