# Lab 1: SRWE Practical Assessment Part 1

## Objective
Build a two-LAN network that uses LACP EtherChannels between switches,
802.1Q trunking, Layer 3 switch inter-VLAN routing (LAN 1),
router-on-a-stick inter-VLAN routing (LAN 2), and SSH management on a switch.

## Topology
![Topology before configuration](screenshots/01-topology-before.png)

## Design summary
| Area | Design |
|------|--------|
| LAN 1 | VLANs 10/20/30 routed by the Layer 3 switch MSW-1 (SVIs) |
| LAN 2 | VLANs 40/50/60/99 routed by RTR-1 using subinterfaces |
| Uplink | MSW-1 G1/1/1 (routed port) to RTR-1 G0/0/0 |
| EtherChannels | Group 1 MSW-1/SW-1, Group 2 MSW-1/SW-2, Group 3 SW-1/SW-2 (LACP active) |
| Management | SW-3 SVI on VLAN 99, SSHv2 with a 1024-bit RSA key |

## VLAN plan
| VLAN | Name | Network | Gateway |
|------|------|---------|---------|
| 10 | SALES | 10.0.10.0/24 | MSW-1 |
| 20 | ACCT | 10.0.20.0/24 | MSW-1 |
| 30 | EXEC | 10.0.30.0/24 | MSW-1 |
| 40 | FAC | 10.0.40.0/24 | RTR-1 |
| 50 | FAB1 | 10.0.50.0/24 | RTR-1 |
| 60 | FAB2 | 10.0.60.0/24 | RTR-1 |
| 99 | ADMIN | 10.0.99.16/28 | RTR-1 |

## What I configured
- RTR-1: enable secret, console and VTY passwords, MOTD banner, encrypted passwords, interface descriptions
- VLANs and static access ports on SW-1, SW-2 and SW-3
- Three LACP EtherChannels with static trunks and DTP disabled
- SW-3 uplink as a static trunk with VLAN 99 as the native VLAN
- Inter-VLAN routing on MSW-1 (`ip routing`, SVIs) and RTR-1 (router-on-a-stick)
- SSH on SW-3 (see below)
- Default gateways on all hosts

## SSH configuration (SW-3)
SSH access to the SW-3 management SVI (VLAN 99, 10.0.99.18/28):

```
ip domain-name acad.pt
username admin secret <REDACTED>
crypto key generate rsa general-keys modulus 1024
ip ssh version 2
line vty 0 15
 transport input ssh
 login local
```

What each part does:
- `modulus 1024` and `ip ssh version 2` set the key size the lab required and force SSHv2.
- `transport input ssh` blocks Telnet on every VTY line.
- `login local` makes the VTY lines authenticate against the local `admin` account.
- `ip default-gateway 10.0.99.17` lets the SVI be reached from other networks.

Verification commands for this feature: `show ip ssh` and
`show running-config | section line vty`. The SSH login screenshot was not
captured before the assessment session was submitted, so the
configuration above is the evidence for this feature.

## Troubleshooting story
**EtherChannel Group 2 would not form.** `show etherchannel summary`
showed `Po2(SD)` with member ports suspended `(s)`. The log said
`%EC-5-CANNOT_BUNDLE2 ... duplex of Gig0/2 is half, Gig0/1 is full`.
Member ports in a bundle must have matching speed and duplex. After I
corrected the ports on both ends, the ports moved from `(s)` to `(I)`
and then to `Po2(SU)` with the member ports bundled.

**STP orange links.** Once the bundles formed, Group 3 showed orange
dots. `show interfaces trunk` on SW-2 listed Po3 with no VLANs in the
forwarding state. That is normal: STP blocks the redundant path in the
triangle and keeps it as a backup.

## Lessons learned
1. All member ports of an EtherChannel must match (speed, duplex, trunk
   mode, native VLAN).
2. Read the log message first. It named the exact fault.
3. `switchport trunk encapsulation dot1q` is rejected on dot1q-only switches.
4. Check the CLI prompt before pasting commands.

## Files
- [`configs/`](configs/): device configurations (passwords redacted)
- [`verification/`](verification/): show command output
- [`screenshots/`](screenshots/): evidence from Packet Tracer
