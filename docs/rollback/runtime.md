# Runtime Rollback and Management Recovery

[Overview](../../README.md) | [Choose a recovery scenario](README.md)

<a id="case-b"></a>

## Case B — Cutover or runtime validation fails before persistence

**Applies only when DPU persistence has not been installed or activated.** Cutover already attempts DPU-first automatic rollback after a disruptive failure. Inspect that result before starting a manual attempt; do not run two rollback controllers concurrently.

From Host KVM:

```bash
cd /root/bf3-lag
test -s state/orig-netplan/01-cloud-net.yaml || { echo 'STOP: original Host backup missing'; exit 1; }
bash bf3-lag.sh rollback-runtime
```

The controller calls ARM `11_rollback_dpu.sh` first, then Host `21_rollback_host.sh`. If ARM rollback fails, the controller stops before restoring the Host. Use [Case C](runtime.md#case-c) for an unreachable DPU or [Cases D and E](persistence.md) for a persistence guard; do not skip the failed DPU step.

After both steps succeed, inspect from Host KVM:

```bash
cat /proc/net/bonding/bond0
ip -d link show bond0.1302
ip -d link show bond0.3048
ip -4 addr show
ip -4 route
ping -I bond0.1302 -c 3 171.232.252.254
ssh -tt ubuntu@192.168.100.2 'sudo ovs-vsctl show'
```

**Validation:** Host VLANs use Host `bond0` again with the saved addresses/routes; ARM shows `ovsbr1:p0+pf0hpf` and `ovsbr2:p1+pf1hpf`. Check public connectivity and a known private peer separately. `verify-runtime` checks the deployed DPU LAG topology, so it is not a post-rollback acceptance test. If the restored DPU-mode baseline still cannot forward both links, use [Case I](nic-mode.md#case-i) when the intended target is the original NIC-mode architecture.

<a id="case-c"></a>

## Case C — TMFIFO or ARM SSH is unavailable

Restore the management path from Host KVM:

```bash
systemctl restart rshim
journalctl -u rshim -b -n 50 --no-pager
ls -l /dev/rshim0/misc /dev/rshim0/console
ip link set tmfifo_net0 up
ip addr replace 192.168.100.1/30 dev tmfifo_net0
ip -4 route get 192.168.100.2
ssh ubuntu@192.168.100.2 'hostname; ip -br addr'
```

The route must use `tmfifo_net0`. If ARM SSH works again, resume the applicable recovery case. If SSH still fails, open `/dev/rshim0/console` with the site's serial-console tool and inspect ARM locally. A missing RShim device requires resolving driver/device attachment before this console is usable.

For a **pre-persistence runtime rollback only**, run on the ARM console:

```bash
sudo bash /home/ubuntu/bf3-lag/11_rollback_dpu.sh
sudo ovs-vsctl show
```

Only after DPU rollback succeeds, return to Host KVM:

```bash
cd /root/bf3-lag
bash 21_rollback_host.sh
```

**Validation:** Use the [Case B](runtime.md#case-b) checks. If the ARM script reports persistence/generated networkd configuration, follow [Cases D and E](persistence.md) instead of forcing Host rollback.
