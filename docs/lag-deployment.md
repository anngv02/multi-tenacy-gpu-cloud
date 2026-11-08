# DPU-managed LAG Deployment

[Overview](../README.md) | [Rollback](rollback/README.md) | [Previous: Firmware and Management Access](firmware-and-management.md) | [Next: Persistence and Reboot Validation](persistence.md)

Run Host commands from HGX KVM and DPU commands on BlueField ARM. See the [execution conventions](firmware-and-management.md) before making changes.

## Step 4 — Back Up, Prepare, and Stage Scripts (No Datapath Changes Yet)

**Purpose:** Prepare the target Netplan configuration, DPU installation scripts, and rollback before interrupting the Ethernet path.

**Run on the HOST through KVM:**

```bash
sudo -i
cd /root/bf3-lag

bash bf3-lag.sh prepare

diff -u /etc/netplan/01-cloud-net.yaml \
  /root/bf3-lag/state/host-target.yaml

bash bf3-lag.sh stage
```

`prepare` backs up the Host Netplan configuration to `/root/bf3-lag/state/orig-netplan/` and generates the target configuration; `stage` transfers the scripts to ARM over TMFIFO. The target configuration **removes Linux bond0 from the Host**, changes the parent of VLANs 1302/3048 to `enp39s0f0np0`, and preserves IP addresses, routes, and DNS. The VLAN names `bond0.1302` and `bond0.3048` remain, but **no longer depend on Host bond0**.

**Expected results:** `READY` appears during prepare, and `DPU preflight passed (topology only)` appears during stage; the DPU apply/rollback scripts have been copied to `/home/ubuntu/bf3-lag/`. Public and private networking continue to operate as before.

## Step 5 — Cutover: Move LACP and VLAN Connectivity from the Host to the DPU

**Purpose:** Combine the two physical uplinks into a bond on the DPU, then switch the Host to PF0.

**Prerequisites:** An independent KVM console is available; ToR/MLAG allows tagged VLANs `1302/3048` with MTU 9000; scripts, backups, and rollback are ready.

**Run on the HOST through KVM:**

```bash
cd /root/bf3-lag
ACK_TOR_LACP=YES bash bf3-lag.sh cutover
```

**The script performs these steps in order:**

1. On the DPU: Remove `p0/p1` from the two existing OVS bridges; create Linux `bond0` with `802.3ad`, `layer3+4`, `lacp_rate=fast`, MTU 9000, and members `p0+p1`; attach **DPU** `bond0` to `ovsbr1` alongside `pf0hpf`.
2. Wait for and verify **two members** in DPU LACP before changing the Host.
3. On the Host: Remove Linux `bond0`, move both VLANs to PF0 using the prepared Netplan configuration, and check connectivity to the public peer.

**Expected result:**

```text
SUCCESS: two-member DPU LAG, Host VLAN parents/IPs and public peer validated.
```

**Architecture after cutover:**

```text
HGX Host (Ubuntu 24.04)
  enp39s0f0np0 (PF0)
   ├── bond0.1302 [VLAN ID 1302] → 171.232.252.202/24
   └── bond0.3048 [VLAN ID 3048] → 10.29.252.239/25
            │ PCIe / pf0hpf
            ▼
BlueField-3 ARM: ovsbr1
            │
       DPU Linux bond0 (802.3ad)
         ├── p0 ── 200GbE ── ToR
         └── p1 ── 200GbE ── ToR
```

If cutover fails, the script attempts an automatic rollback. To trigger rollback manually **from KVM** (only when restoring the previous configuration is necessary):

```bash
bash bf3-lag.sh rollback-runtime
```

## Step 6 — Validate DPU-managed LAG at Runtime

**Purpose:** Confirm both uplinks and forwarding before configuring persistence across reboots.

**Run on the HOST:**

```bash
cd /root/bf3-lag
bash bf3-lag.sh verify-runtime
```

**Expected results:**

- DPU `bond0` uses **802.3ad**, with `p0` and `p1` in the same aggregator and `ad_num_ports=2`.
- Each member operates at **200 Gb/s**, with actor/partner states `in_sync`, `collecting`, and `distributing`; MTU is 9000.
- OVS `ovsbr1` contains `bond0 + pf0hpf`; VLAN 1302 forwarding can be offloaded to the eSwitch hardware.
- The Host no longer has Linux `bond0`, but public and private VLANs remain attached to PF0 with their original IP addresses.
- The public peer `171.232.252.212` is reachable.

**Completion message:**

```text
PASS: runtime topology/LACP/MTU/public reachability.
```

*Note:* The two 200GbE links provide **400 Gb/s of nominal aggregate link capacity in each direction**; this does not mean that applications have measured 400 Gb/s. The private VLAN retains its address; validating end-to-end private connectivity requires a valid private peer.

For behavior during a single-member outage, see the separate [LACP failover report](lacp-failover.md). Runtime validation above checks the healthy two-member topology; it does not measure traffic interruption during failover.
