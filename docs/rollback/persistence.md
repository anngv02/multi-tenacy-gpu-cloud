# Persistence Recovery and Rollback

[Overview](../../README.md) | [Choose a recovery scenario](README.md)

<a id="case-d"></a>

## Case D — Persistence installation is incomplete or staged changes must be cancelled

First determine which side completed installation:

```bash
# Host KVM
ls -l /root/bf3-lag-persistence-backup/
ls -l /etc/netplan/90-bf3-tmfifo-host.yaml
ssh -tt ubuntu@192.168.100.2 \
  'sudo ls -l /var/lib/bf3-lag-persistence/; sudo systemctl status bf3-lag-reconcile.service --no-pager'
```

**DPU installation completed, Host installation did not:** Fix the Host error. If Host installation did not already create its backup or target file, complete that phase and validate:

```bash
cd /root/bf3-lag
bash persistence/30_install_host_mgmt.sh
bash bf3-lag.sh verify-staged
```

If the Host phase already created a partial backup/configuration, inspect it first; the installer refuses to overwrite it. Likewise, an incomplete DPU installation needs inspection of its saved backup and installed files. Do not rerun the full installer over an existing DPU backup.

**Cancel staged persistence:** This path applies only before either side has rebooted, before live `netplan apply`, and before reconcile has run. Confirm that the files belong to this package. Remove DPU startup configuration from Host KVM:

```bash
ssh -tt ubuntu@192.168.100.2 \
  'sudo bash ~/bf3-lag/persistence/21_remove_dpu_persistence.sh'
```

Continue only if removal succeeds. The removal script expects a stoppable reconcile service; if installation failed before creating that unit, it can exit early. Inspect the partial installation on ARM instead of assuming removal completed.

If Host management persistence was also installed, remove it from Host KVM:

```bash
cd /root/bf3-lag
bash persistence/31_remove_host_mgmt.sh
```

Then run `bash bf3-lag.sh rollback-runtime` from Host KVM and use the [Case B](runtime.md#case-b) checks. If generated networkd configuration blocks rollback, move to [Case E](persistence.md#case-e).

**Validation:** The package's persistence YAML/service files are removed and runtime rollback has completed. Removing persistence alone leaves the active DPU bond unchanged; do not reboot between removing persistence and establishing a coherent recovery configuration. Preserve both backup directories.

<a id="case-e"></a>

## Case E — LAG fails after a reboot or persistence has already been applied

Recover TMFIFO using [Case C](runtime.md#case-c), then inspect the persistent LAG from Host KVM:

```bash
ssh -tt ubuntu@192.168.100.2 \
  'sudo systemctl status bf3-lag-reconcile.service --no-pager -l'
ssh -tt ubuntu@192.168.100.2 \
  'sudo journalctl -b -u bf3-lag-reconcile.service --no-pager'
ssh ubuntu@192.168.100.2 'cat /proc/net/bonding/bond0'
ssh -tt ubuntu@192.168.100.2 'sudo ovs-vsctl show'
```

Resolve the reported cause: missing interfaces/OVS, a missing persistence file, incorrect Hash Mode, wrong port ownership, or ToR LACP/VLAN/MTU configuration. When the expected persistent configuration is present and the cause has been corrected, restart reconcile and validate:

```bash
ssh -tt ubuntu@192.168.100.2 \
  'sudo systemctl restart bf3-lag-reconcile.service'
cd /root/bf3-lag
bash bf3-lag.sh verify-runtime
bash bf3-lag.sh verify-boot
```

`verify-boot` requires both Host and ARM to have rebooted since installation. If only one rebooted, that check can fail even when runtime networking has recovered; complete the other reboot during maintenance before claiming persistence validation.

**If full rollback is required:** The package does not automate removal of live networkd/NM bond ownership. Capture state on ARM, inspect `/etc/netplan/70-bf3-lag.yaml`, `/etc/NetworkManager/conf.d/98-bf3-lag-unmanaged.conf`, and generated files under `/run/systemd/network/`, and prepare a BSP-specific ownership transition using the saved configuration. Removing YAML alone is insufficient. Do not delete generated files indiscriminately or bypass the guard in `11_rollback_dpu.sh`. For a complete return to NIC Mode, use [Case I](nic-mode.md#case-i); for continued DPU Mode, complete the platform-specific ownership transition before [Case B](runtime.md#case-b).
