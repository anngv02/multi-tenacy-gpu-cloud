# Recovery and Rollback by Scenario

[Overview](../../README.md)

Choose the case that matches the current state. Run Host recovery commands as root from **HGX KVM** and DPU commands as root through **TMFIFO or the ARM console**. Keep backups and console access available throughout recovery.

`rollback-runtime` restores the **pre-cutover DPU-mode topology**: two ARM OVS bridges and the original Host Netplan bond. It does not revert firmware, disable DPU Mode, remove anti-spoofing rules, or reset rate limits. In this deployment, the pre-cutover DPU-mode topology did not aggregate both Host links successfully; restoring it is not proof of full 2×200GbE recovery. Returning to the original NIC-mode architecture is a separate procedure in [Case I](nic-mode.md#case-i).

| Situation | Recovery path | Expected outcome |
| --- | --- | --- |
| `prepare` or `stage` fails before cutover | [Case A](preparation.md#case-a) | Keep the existing Ethernet datapath; resolve prerequisites |
| Cutover fails, public connectivity is lost, or runtime validation fails before persistence | [Case B](runtime.md#case-b) | Restore DPU ports first, then Host Netplan |
| TMFIFO/ARM SSH is unavailable during rollback | [Case C](runtime.md#case-c) | Recover management access or use the ARM console before Host rollback |
| Persistence is incomplete or must be removed before any reboot/apply/reconcile | [Case D](persistence.md#case-d) | Complete the missing installation phase, or remove staged files and roll back runtime |
| DPU LAG does not recover after reboot | [Case E](persistence.md#case-e) | Repair the persistent LAG; do not bypass the runtime rollback guard |
| Valid traffic is blocked after anti-spoofing changes | [Case F](policies.md#case-f) | Remove only this report's anti-spoofing rules |
| TX throughput drops after policing or PF0 rate limiting | [Case G](policies.md#case-g) | Restore the relevant previous rate settings |
| RX metering fails or a standalone TC test action remains | [Case H](policies.md#case-h) | Restore affected forwarding rules, remove the identified test meter/action |
| The deployment must return to NIC Mode | [Case I](nic-mode.md#case-i) | Restore Host bonding and perform a separate firmware mode transition |

Before changing the DPU datapath, capture its current state for diagnosis. These command outputs are diagnostic records, not files to replay directly with `add-flows`:

```bash
# DPU ARM, as root
RECOVERY_DIR=$(mktemp -d /root/bf3-recovery.XXXXXX)
ovs-vsctl show > "$RECOVERY_DIR/ovs.txt"
ovs-ofctl -O OpenFlow13 dump-flows ovsbr1 > "$RECOVERY_DIR/flows.txt"
ovs-ofctl -O OpenFlow13 dump-meters ovsbr1 > "$RECOVERY_DIR/meters.txt"
ip -d link show > "$RECOVERY_DIR/links.txt"
ip -4 addr show > "$RECOVERY_DIR/addresses.txt"
printf 'Recovery evidence: %s\n' "$RECOVERY_DIR"
```
