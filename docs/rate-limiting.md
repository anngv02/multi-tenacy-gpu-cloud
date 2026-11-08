# TX/RX Rate Limiting

[Overview](../README.md) | [Rollback](rollback/README.md) | [Previous: Throughput Benchmarks](benchmarks.md) | [Next: Final Results](results.md)

Run Host commands from HGX KVM and DPU commands on BlueField ARM. See the [execution conventions](firmware-and-management.md) before making changes.

## Step 11 — Test TX Rate Limiting at 100 Gb/s on BlueField-3

**Purpose:** Limit traffic in the **HGX202 Host → Ethernet Fabric** direction to 100 Gb/s, comparing OVS ingress policing with NVIDIA's direct Host PF rate limit. Both public and private VLANs pass through Host PF0, so a PF0 cap is not a separate policy for each IP/VLAN.

**11.1. Test OVS ingress policing on DPU ARM (failed):**

```bash
# Back up the values before changing them: rate=0, burst=0
mkdir -p /root/bf3-ratelimit
cd /root/bf3-ratelimit
ovs-vsctl get Interface pf0hpf ingress_policing_rate > old_rate.txt
ovs-vsctl get Interface pf0hpf ingress_policing_burst > old_burst.txt

# 100 Gb/s; OVS uses kbit/s for rate and kbit for burst
ovs-vsctl set Interface pf0hpf \
  ingress_policing_rate=100000000 \
  ingress_policing_burst=1000000

ovs-vsctl get Interface pf0hpf ingress_policing_rate
tc -s filter show dev pf0hpf ingress
tc -s action ls action police
```

**Observed results:** OVSDB accepted `100000000` kbit/s, and TC `matchall` showed `in_hw`, but the action reported only `rate 31280Mbit`. Three `-P32` benchmark runs achieved **29.32 / 29.32 / 29.31 Gb/s** (average **29.32 Gb/s**): **the TX 100 Gb/s target was not met**.

**Roll back OVS policing on DPU ARM:**

```bash
ovs-vsctl set Interface pf0hpf \
  ingress_policing_rate=0 ingress_policing_burst=0
ovs-vsctl get Interface pf0hpf ingress_policing_rate
tc -s filter show dev pf0hpf ingress
```

After rollback, `matchall police` disappeared; the IPv4 forwarding flower filter remained `in_hw`. Three baseline measurements averaged **341.26 Gb/s** ([Step 10: benchmarks](benchmarks.md)).

**11.2. Limit TX using NVIDIA Host PF** `max_tx_rate` **(passed):**

Inspection on DPU ARM showed that `p0/smart_nic/pf/config` had MAC `58:a2:e1:d6:b2:c8` and `MaxTxRate=0`, matching Host PF0/`pf0hpf`; `p1` corresponded to MAC `58:a2:e1:d6:b2:c9`. Both expose `max_tx_rate` through sysfs, in **Mb/s**.

```bash
# On DPU ARM, back up and configure Host PF0 only
mkdir -p /root/bf3-ratelimit
cat /sys/class/net/p0/smart_nic/pf/config \
  > /root/bf3-ratelimit/pf0-before.txt

echo 100000 > /sys/class/net/p0/smart_nic/pf/max_tx_rate
cat /sys/class/net/p0/smart_nic/pf/config

# On HGX202, repeat TX measurements
cd /root/bf3-ratelimit
bash run32.sh limit100G_pf
```

**Observed results:** `MaxTxRate: 100000`, `MinTxRate: 0`, `State: Up`; OVS policing remained `0`.

| Run 1 | Run 2 | Run 3 | Average | Conclusion |
| ---------- | ---------- | ---------- | -------------- | ----------------------------- |
| 93.73 Gb/s | 93.71 Gb/s | 93.73 Gb/s | **93.72 Gb/s** | **PASS: TX 100 Gb/s limit** |

After testing, **all 6 anti-spoofing rules remained**, and DPU LAG retained **2 members UP at 200 Gb/s/member**. PF0 TX still had `MaxTxRate=100000` at the last inspection; automatic restoration of the rate limit after a DPU reboot has not been validated.

**Roll back PF0 when needed:** `echo 0 > /sys/class/net/p0/smart_nic/pf/max_tx_rate` (run only when intentionally removing the limit).

## Step 12 — Identify the TC 32-bit Limitation and Check 64-bit Support

**Purpose:** Distinguish the kernel/`tc` ability to represent rates from OVS policing programming errors, and confirm configuration cleanup after RX meter testing.

**On DPU ARM — inspect and create a standalone TC action without attaching a filter:**

```bash
uname -r
tc -V
grep -n 'TCA_POLICE_RATE64' /usr/include/linux/pkt_cls.h

# Standalone action to test rate representation only, decimal index 62000
tc actions add action police \
  rate 100gbit burst 12500000 \
  conform-exceed drop/pipe index 62000

tc -s actions ls action police

# Remove only action 62000 after checking
tc actions delete action police index 62000
tc -s actions ls action police
```

**Observed results:** ARM kernel `5.15.0-1032-bluefield`; iproute2 `5.15.0`; the header contains `TCA_POLICE_RATE64`. TC read back action `0xf230` (index 62000) with `rate 100Gbit`, `ref 1 bind 0`, although the CLI emitted a syntax message during the operation; a subsequent creation attempt reported `File exists`. Action 62000 was deleted, and the final list showed `total acts 0`.

**Cause analysis:** `100 Gb/s ÷ 8 = 12,500,000,000 bytes/s`; if stored as an unsigned 32-bit value, `12,500,000,000 mod 2³² = 3,910,065,408 bytes/s ≈ 31.28 Gb/s`, matching the observed TC policer. **Technical conclusion:** The kernel/`tc` can represent 100 Gb/s; the issue is likely in the rate conversion path from **NVIDIA OVS 2.17.8 → TC**. Successful offload of a standalone 100 Gb/s hardware policer has not been demonstrated.
