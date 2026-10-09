# AD Attack & Defend Lab — Week 1: Infrastructure

## Topology

- **DC01-LAB** — Windows Server 2019, domain controller for `pretorium.lab`, host-only adapter `192.168.56.10`
- **WS01-LAB** — Windows 10 Enterprise (Evaluation), domain-joined workstation, host-only adapter `192.168.56.14`
- **LinuxVM-LAB** — Ubuntu Server 22.04, `192.168.56.13`, Wazuh SIEM host (Months 2–3)
- Network: VirtualBox Host-Only Adapter, `192.168.56.0/24`, DHCP disabled, all addresses statically assigned

## What this week covers

Joined a second Windows 10 workstation to the `pretorium.lab` domain to support Week 2's Kerberoasting attack/detection work, which requires a domain-joined endpoint distinct from the DC itself.

## Troubleshooting log

This domain join did not work on the first attempt. Documented here because the root-cause sequence is more instructive than a clean success would have been.

**1. Overlapping virtual network subnets**
VirtualBox's Host-only Network and NAT Network were both configured on `192.168.56.0/24`, causing DC01-LAB to present two interfaces claiming the same address range. Fixed by moving the NAT Network to a non-overlapping range (`10.0.3.0/24`).

**2. Domain controller IP drift**
DC01-LAB's host-only adapter was obtaining its address via DHCP rather than a static assignment, so its IP changed across reboots (`.10` → `.15`). A domain controller's IP must never be dynamic — every client, GPO, and DNS record depends on it staying fixed. Fixed by disabling the Host-only Network's DHCP server and statically assigning `192.168.56.10` to DC01-LAB.

**3. Duplicate IP conflict**
After reassigning WS01-LAB's address, the interface briefly held two IPv4 addresses simultaneously (`.13`, already owned by LinuxVM-LAB, and the newly assigned `.14`) due to a stale DHCP lease persisting alongside the new static entry. Fixed by removing both addresses and re-adding a single static entry.

**4. Stale DNS host record**
`nslookup pretorium.lab` returned both the correct host-only address (`192.168.56.10`) and DC01-LAB's NAT adapter address (`10.0.3.10`) under the same domain-root record — the DC had registered both NICs into the DNS zone instead of only the one clients can actually route to. Cleaned up via DNS Manager on DC01-LAB.

**5. Root cause: Windows edition restriction**
After resolving DNS and network-layer connectivity, `Add-Computer` still failed with "Access is denied." The actual cause: WS01-LAB was running **Windows 10 Home**, which cannot join an Active Directory domain under any credentials — this is an OS-level restriction, not an authentication failure, and Windows's error message does not distinguish the two. Confirmed via `Get-ComputerInfo -Property WindowsProductName`, then resolved by rebuilding the VM on **Windows 10 Enterprise Evaluation**.

## What this demonstrates

Each layer — network addressing, DHCP behavior, DNS consistency, and OS licensing — was isolated and ruled out independently before moving to the next, rather than guessing at the fix. The final cause was two layers removed from the symptom.

## Evidence

See `vm-configs/` for:
- `ws01-enterprise-eval-confirmed.png`
- `ws01-network-connectivity-confirmed.png`
- `adcomputer-filter-both-workstations.png`
