# Initial Environment

[Overview](../README.md) | [Rollback](rollback/README.md) | [Next: Firmware and Management Access](firmware-and-management.md)

## 2.1. HGX and BlueField-3 Hardware

| Component | Initial Configuration |
| ------------------------- | -------------------------------------------------------------------------- |
| Server | One NVIDIA HGX H200 node running bare-metal Ubuntu |
| DPU                       | NVIDIA BlueField-3 P-Series, PN `900-9D3B6-00CV-AA0`, PSID `MT_0000000884` |
| DPU specifications | 2 QSFP112 ports at **200GbE/port**, PCIe Gen5 x16, 16 ARM cores, 32 GB DDR5 |
| BlueField firmware | `32.39.5124` at the time of inspection |
| Host Ethernet driver | `mlx5_core`, version `2607.0.100` |
| PF0                       | PCI `0000:27:00.0` ↔ `enp39s0f0np0`                                        |
| PF1                       | PCI `0000:27:00.1` ↔ `enp39s0f1np1`                                        |
| SoC management | PCI `0000:27:00.2` — BlueField-3 Management Interface, used for RShim PCIe |

The two 200GbE ports provide a nominal aggregate link capacity of **2×200 = 400 Gb/s in each direction** when used through LAG; this does not guarantee that a single TCP/UDP flow can reach 400 Gb/s.

## 2.2. Operating Systems, Tools, and Initial Firmware State

| Item | State Before Deployment |
| --------------------------- | ---------------------------------------------------------------------- |
| Host OS                     | Ubuntu 24.04.4 LTS, kernel `6.8.0-147-generic`                         |
| Host network management | `systemd-networkd` with Netplan `/etc/netplan/01-cloud-net.yaml` |
| Host OVS | Not the active datapath; the Host uses Linux bonding |
| Firmware tools | `mstconfig` was initially unavailable; the `mstflint` package was installed later |
| DPU management | RShim/TMFIFO was not yet used to operate the ARM datapath |
| Host-side BlueField eSwitch | `devlink ... eswitch show` reports `legacy` mode |

Key firmware parameters before switching modes:

| `mstconfig` Parameter | Initial Current Value | Meaning |
| ------------------------------ | ---------------- | -------------------------------------------- |
| `INTERNAL_CPU_OFFLOAD_ENGINE` | `DISABLED(1)` | BlueField is in **NIC Mode** |
| `INTERNAL_CPU_ESWITCH_MANAGER` | `EXT_HOST_PF(1)` | The Host PF manages the eSwitch |
| `INTERNAL_CPU_PAGE_SUPPLIER` | `EXT_HOST_PF(1)` | Related resources are managed by the Host PF |
| `INTERNAL_CPU_RSHIM` | `ENABLED(0)` | The internal CPU RShim channel is supported |

**Note:** `devlink ... mode legacy` reflects the Host-side eSwitch mode; it does not replace checking the firmware parameter `INTERNAL_CPU_OFFLOAD_ENGINE` to determine NIC/DPU Mode.

## 2.3. Initial Ethernet, Bond, and VLAN Configuration

| Component | Configuration |
| -------------------------- | --------------------------------------------------------------------- |
| Host bond                  | `bond0`, IEEE `802.3ad` (LACP), TX hash `layer3+4`                    |
| Bond member 1              | `enp39s0f0np0`, **200 Gb/s**, UP                                      |
| Bond member 2              | `enp39s0f1np1`, **200 Gb/s**, UP                                      |
| Public VLAN / IP           | VLAN `1302`, `bond0.1302` — `171.232.252.202/24`                      |
| Public gateway             | `171.232.252.254`                                                     |
| Private VLAN / IP          | VLAN `3048`, `bond0.3048` — `10.29.252.239/25`                        |
| LACP in NIC Mode | Active aggregator with **2 ports**, partner System MAC `44:38:39:ff:00:06` |

**Initial architecture:**

```text
HGX H200 Host — Ubuntu 24.04.4
    VLAN 1302: 171.232.252.202/24
    VLAN 3048: 10.29.252.239/25
                    │
              Linux bond0
              (802.3ad)
              /       \
 enp39s0f0np0         enp39s0f1np1
   PCI 27:00.0          PCI 27:00.1
     200GbE               200GbE
          \              /
           BlueField-3 NIC Mode
                    │
             Ethernet ToR / MLAG
```

In this state, the Host handles Linux bonding and IP configuration. After switching to DPU Mode, OVS on ARM creates two datapaths, `pf0hpf ↔ p0` and `pf1hpf ↔ p1`; Host LACP no longer aggregates both paths as before. To retain 2×200GbE connectivity while enforcing network policies independently of the Host, LACP management must move to the DPU.
