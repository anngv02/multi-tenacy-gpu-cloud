# Firmware and Management Access

[Overview](../README.md) | [Rollback](rollback/README.md) | [Previous: Initial Environment](environment.md) | [Next: DPU-managed LAG Deployment](lag-deployment.md)

> **Conventions:** Run HOST commands on the HGX, preferably from the KVM console. Run DPU commands in the BlueField ARM OS. All `bf3-lag.sh` commands below run on the **HGX Host** in `/root/bf3-lag` (download and extract the attached [`bf3-lag.zip`](../bf3-lag.zip) first). Step 4 onward describes a planned cutover procedure; do not rerun these steps on a system where deployment is already complete.

For failure handling, use [Recovery and Rollback by Scenario](rollback/README.md).

## Step 1 — Switch BlueField from NIC Mode to DPU Mode

**Purpose:** Transfer eSwitch management from the Host PF to ECPF/ARM.

**Inspect the HOST before making changes:**

```bash
lspci -Dnn | grep -Ei 'BlueField|Ethernet controller'
ethtool -i enp39s0f0np0
ethtool -i enp39s0f1np1
cat /proc/net/bonding/bond0
ip -4 addr show
ip -4 route
sudo devlink dev eswitch show pci/0000:27:00.0
```

**Expected results:**

- The HOST detects both BlueField-3 ports (27:00.0, 27:00.1), using the mlx5_core driver.
- bond0, IP addresses, and the routing table are operating normally.
- The eSwitch is in legacy mode.

**Run on the HOST:**

```bash
sudo apt-get install -y mstflint
sudo mstconfig -d 27:00.0 -e query | \
  grep -E 'INTERNAL_CPU_OFFLOAD_ENGINE|INTERNAL_CPU_ESWITCH_MANAGER'

sudo mstconfig -d 27:00.0 set INTERNAL_CPU_OFFLOAD_ENGINE=0
```

Power cycle the HGX through the power management/KVM system according to the operating procedure, then verify:

```bash
sudo mstconfig -d 27:00.0 -e query | \
  grep -E 'INTERNAL_CPU_OFFLOAD_ENGINE|INTERNAL_CPU_ESWITCH_MANAGER'
```

**Expected results:** `INTERNAL_CPU_OFFLOAD_ENGINE=ENABLED(0)` and `INTERNAL_CPU_ESWITCH_MANAGER=ECPF(0)` under both `Current` and `Next Boot`; BlueField is operating in **DPU Mode**.

## Step 2 — Enable RShim/TMFIFO Management Access and Inspect OVS

**Purpose:** Access the ARM OS independently of the public Ethernet path being modified.

**Run on the HOST:**

```bash
sudo apt-get install -y rshim
sudo systemctl start rshim
journalctl -u rshim -b -n 30 --no-pager
ls -l /dev/rshim0/
cat /dev/rshim0/misc

sudo ip link set tmfifo_net0 up
# Add the IP address only if it is not already assigned to TMFIFO:
if ! ip -4 addr show dev tmfifo_net0 | grep -q '192.168.100.1/'; then
  sudo ip addr add 192.168.100.1/30 dev tmfifo_net0
fi

ip -4 route get 192.168.100.2
ssh ubuntu@192.168.100.2
```

The RShim log should contain:

```text
Probing pcie-0000:27:00.2(vfio)
Create rshim pcie-0000:27:00.2
rshim0 attached
```

**Run on DPU ARM:**

```bash
cat /etc/os-release
uname -r
sudo ovs-vsctl show
sudo ovs-vsctl get Open_vSwitch . other_config
```

**Expected results:** Management PCI device `27:00.2` is bound to `vfio-pci`, and RShim attaches successfully; the Host can SSH into ARM over TMFIFO at `192.168.100.2`. ARM runs Ubuntu 22.04.3, kernel `5.15.0-1032-bluefield`, OVS `2.17.8+nvidia.c5eb44be9`, with `hw-offload="true"`; after switching to DPU Mode, `ovsbr1` connects `pf0hpf ↔ p0`, and `ovsbr2` connects `pf1hpf ↔ p1`.

**Intermediate state:** The two independent bridges do not yet form a shared LAG. The Host still has `bond0`, but Host LACP no longer aggregates both ports as before. LACP management therefore needs to move to the DPU.

## Step 3 — Enable Firmware Hash Mode for DPU-managed LAG

**Purpose:** Prepare the firmware for BlueField to implement LAG in Hash Mode.

**Run on the HOST:**

```bash
sudo mstconfig -d 27:00.0 set LAG_RESOURCE_ALLOCATION=1
```

**On DPU ARM:** Ensure `/etc/mellanox/mlnx-bf.conf` contains:

```ini
LAG_HASH_MODE="yes"
```

Power cycle the HGX to apply the firmware configuration, then read it back:

```bash
sudo mstconfig -d 27:00.0 -e query | \
  grep -E 'INTERNAL_CPU_OFFLOAD_ENGINE|LAG_RESOURCE_ALLOCATION'
```

**Expected results:** `LAG_RESOURCE_ALLOCATION=PRE_ALLOCATION(1)` under `Current/Next Boot`, with DPU Mode still enabled. LACPDUs received on `p0/p1` have the same ToR Actor System ID `44:38:39:ff:00:06` and Key `31`, consistent with deployment in the same LAG/MLAG.
