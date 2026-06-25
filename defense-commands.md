# 🛡️ Defense & Detection Commands (Cisco SW1)

> Apply the defense, then use these to PROVE it's working and to
> DETECT the attack in progress.

## DHCP Snooping

```cisco
! Enable
ip dhcp snooping
ip dhcp snooping vlan 1
interface Ethernet0/0
 ip dhcp snooping trust
interface range Ethernet0/1 - 2
 ip dhcp snooping limit rate 10

! Verify / detect
show ip dhcp snooping              ! enabled? trusted ports?
show ip dhcp snooping binding      ! legit IP<->MAC<->port table
show ip dhcp snooping statistics   ! dropped packets (attack signal)
```

## Dynamic ARP Inspection (needs snooping first)

```cisco
! Enable
ip arp inspection vlan 1
interface Ethernet0/0
 ip arp inspection trust

! Verify / detect
show ip arp inspection vlan 1          ! enabled, active
show ip arp inspection statistics      ! dropped forged ARPs rising = attack
show ip arp inspection interfaces      ! trust state per port
```

## Port Security (vs MAC flooding)

```cisco
! Enable
interface range Ethernet0/1 - 2
 switchport mode access
 switchport port-security
 switchport port-security maximum 2
 switchport port-security violation restrict
 switchport port-security mac-address sticky

! Verify / detect
show port-security                 ! per-port max/violations
show port-security address         ! learned sticky MACs
show port-security interface Ethernet0/1
show interfaces status err-disabled   ! ports shut by a violation
```

## SYN Flood Defense (vs DoS)

```bash
# --- On the victim host (Linux) — SYN cookies survive the flood ---
sudo sysctl -w net.ipv4.tcp_syncookies=1
sudo sysctl -w net.ipv4.tcp_max_syn_backlog=2048
sudo sysctl -w net.ipv4.tcp_synack_retries=2

# Make permanent: add to /etc/sysctl.conf then: sudo sysctl -p

# Verify / detect
cat /proc/sys/net/ipv4/tcp_syncookies          # -> 1
netstat -tn | grep SYN_RECV | wc -l            # half-open count
dmesg | grep -i "syn"                          # "possible SYN flooding" notice
```

```cisco
! --- On Cisco — police SYN-heavy traffic toward the server ---
ip access-list extended SYN-RATE
 permit tcp any host TARGET_IP
class-map match-all SYN-CLASS
 match access-group name SYN-RATE
policy-map SYN-LIMIT
 class SYN-CLASS
  police 1000000 conform-action transmit exceed-action drop
interface Ethernet0/1
 service-policy input SYN-LIMIT

! Verify
show policy-map interface Ethernet0/1          ! exceed/drop counters
```

## (Optional) DTP / CDP Hardening — general good practice

```cisco
interface range Ethernet0/1 - 2
 switchport mode access
 switchport nonegotiate
 no cdp enable
 spanning-tree portfast
 spanning-tree bpduguard enable
```

## Quick "is it secure?" sweep

```cisco
show ip dhcp snooping
show ip arp inspection vlan 1
show port-security
show interfaces switchport | include Negotiation|Mode
```

---

## Recovery — port stuck in err-disabled

```cisco
! Manually bring it back
interface Ethernet0/1
 shutdown
 no shutdown

! Or auto-recover after a timeout
errdisable recovery cause psecure-violation
errdisable recovery interval 300
```
