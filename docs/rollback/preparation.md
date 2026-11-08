# Preparation Recovery

[Overview](../../README.md) | [Choose a recovery scenario](README.md)

<a id="case-a"></a>

## Case A — Preparation or staging fails before cutover

1. Stop before running `cutover`. `prepare` validates files offline; `stage` changes TMFIFO management and copies scripts, but neither changes the Ethernet uplink/VLAN datapath.
2. From Host KVM, inspect the failure and confirm the original Host bond and addresses remain:

```bash
cd /root/bf3-lag
bash bf3-lag.sh inspect
cat /proc/net/bonding/bond0
ip -4 addr show
ip -4 route
ls -l /root/bf3-lag/state/orig-netplan/
```

3. Resolve the reported dependency, firmware prerequisite, Netplan filename, or TMFIFO issue. The scripts expect `/etc/netplan/01-cloud-net.yaml`; check the actual Host filename before preparing a deployment.
4. Keep any existing `state/orig-netplan/` backup. `prepare` deliberately refuses to overwrite it. Review a partial preparation before retrying; do not delete the backup to force a fresh run. If preparation completed and only `stage` failed, fix management access and rerun `bash bf3-lag.sh stage`.

**Validation:** The Host retains its original configuration and its previous connectivity level. No runtime rollback is needed if cutover never began.
