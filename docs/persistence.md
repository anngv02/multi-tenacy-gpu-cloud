# Persistence and Reboot Validation

[Overview](../README.md) | [Rollback](rollback/README.md) | [Previous: DPU-managed LAG Deployment](lag-deployment.md) | [Next: IPv4/ARP IP/MAC Anti-Spoofing](anti-spoofing.md)

Run Host commands from HGX KVM and DPU commands on BlueField ARM. See the [execution conventions](firmware-and-management.md) before making changes.

## Step 7 — Install Persistence for DPU LAG and Host Management Access

**Purpose:** Automatically recreate the bond/OVS configuration after DPU boot and restore RShim/TMFIFO after Host boot.

**Run on the HOST through KVM:**

```bash
cd /root/bf3-lag
bash bf3-lag.sh install-persistence
bash bf3-lag.sh verify-staged
```

**Installed configurations:**

| Location | Persistence Component |
| ------ | -------------------------------------------------------------------------------------------- |
| DPU | `/etc/netplan/70-bf3-lag.yaml`: `systemd-networkd` manages `p0`, `p1`, and `bond0` |
| DPU | `/etc/NetworkManager/conf.d/98-bf3-lag-unmanaged.conf`: prevents NM from managing the three LAG interfaces |
| DPU | `bf3-lag-reconcile.service`: waits for the driver/OVS and restores LAG state at boot |
| Host | `/etc/netplan/90-bf3-tmfifo-host.yaml`: management address `192.168.100.1/30` |
| Host | `rshim.service`: enabled to start automatically |

The installation script validates Netplan offline and **does not run** `netplan apply` **or restart OVS** at this step; the active datapath is preserved.

**Expected results:**

```text
SUCCESS: Persistence STAGED on DPU+Host.
PASS: persistence staged checks.
```

It is normal for DPU `bf3-lag-reconcile.service` not to have run before the reboot.

## Step 8 — Reboot and Validate Persistence

**Purpose:** Verify that the configuration is automatically restored after rebooting both ARM and the Host.

**From the HGX Host through KVM, reboot DPU ARM first:**

```bash
ssh -tt ubuntu@192.168.100.2 'sudo reboot'
```

Once the DPU is back up, verify that the recovery service completed successfully:

```bash
ssh -tt ubuntu@192.168.100.2 \
  'sudo systemctl status bf3-lag-reconcile.service --no-pager'
```

**Expected results:** `bf3-lag-reconcile.service` succeeds, and the log contains:

```text
[BF3-LAG] SUCCESS: bond0 has 2 LACP forwarding members; OVS datapath attached
```

Next, **reboot the HGX Host from KVM**:

```bash
reboot
```

Once the Host is back up:

```bash
sudo -i
cd /root/bf3-lag
bash bf3-lag.sh verify-boot
```

**Expected results:**

```text
Result=success
ExecMainStatus=0
ActiveState=active
PASS: persistence boot checks.
```

These results confirm that the two-member DPU LAG, OVS datapath, Host VLANs, RShim/TMFIFO, and recovery service are present after the reboots. If the boot log reports that OVSDB is not ready during the first few seconds but the service finishes with `success`, this is a temporary connection error while OVS initializes, not a final validation failure.
