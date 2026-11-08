# Return to NIC Mode

[Overview](../../README.md) | [Choose a recovery scenario](README.md)

<a id="case-i"></a>

## Case I — Return the deployment to NIC Mode

Use this separate maintenance procedure when the target is the original Host-managed bond, including when DPU Mode itself must be abandoned. It disables ARM-based datapath policy enforcement. The runtime rollback command does not perform this mode change.

1. Keep independent HGX KVM/power control, stop workloads, and confirm that the saved original Host Netplan defines both PFs in `bond0` with both VLANs. If `prepare` was never run, use a known pre-change backup or confirm that the original Host configuration is still intact. Do not invent a replacement from the post-cutover state.
2. While ARM is still accessible, preserve its configuration/backups and disable this package's reconcile service if installed. Record any active persistence, anti-spoofing, and rate-limit configuration for review before a future return to DPU Mode:

```bash
# DPU ARM; only if this package's unit is installed
systemctl disable --now bf3-lag-reconcile.service
```

3. On Host KVM, restore the original Host Netplan with `bash /root/bf3-lag/21_rollback_host.sh` if a cutover occurred and the backup is valid. This interrupts Ethernet while ARM may still have the DPU LAG. Do not expect connectivity until NIC Mode is active. If the Host was never cut over, leave its original configuration in place.
4. Query firmware first and check the mode-switch procedure for the installed BlueField-3 BSP/firmware. The BlueField-3 transition uses `INTERNAL_CPU_OFFLOAD_ENGINE=1`; the following uses the `mstconfig` tool already used in this report:

```bash
# Host KVM
mstconfig -d 27:00.0 -e query
mstconfig -d 27:00.0 set INTERNAL_CPU_OFFLOAD_ENGINE=1
mstconfig -d 27:00.0 -e query
```

5. Apply the change using the platform's required power-cycle procedure. Once the Host is back, verify on Host KVM:

```bash
mstconfig -d 27:00.0 -e query | \
  grep -E 'INTERNAL_CPU_OFFLOAD_ENGINE|INTERNAL_CPU_ESWITCH_MANAGER'
cat /proc/net/bonding/bond0
ip -4 addr show
ip -4 route
ping -I bond0.1302 -c 3 171.232.252.254
```

**Validation:** Firmware reports NIC Mode (`INTERNAL_CPU_OFFLOAD_ENGINE=DISABLED(1)`) as the current state, Host bonding has both 200GbE members in the same LACP aggregator with collecting/distributing enabled, and public/private connectivity matches the original baseline. ARM SSH is not an acceptance check in BlueField-3 NIC Mode because ARM cores are inactive. See [NVIDIA BlueField BSP Modes of Operation](https://docs.nvidia.com/networking/display/bluefieldbsp4131/modes-of-operation.pdf) for mode transitions and platform restrictions.

This changes the operating mode; it does not restore every firmware field to its original value. The package's `state/mstconfig.before.txt` was captured after Hash Mode preparation, so it is not a pre-deployment firmware backup. Restore other firmware/ARM settings only from a genuine earlier baseline and the matching platform procedure. Review retained ARM configuration before enabling DPU Mode again.
