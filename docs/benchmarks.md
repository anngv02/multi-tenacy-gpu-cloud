# Throughput Benchmarks

[Overview](../README.md) | [Rollback](rollback/README.md) | [Previous: IPv4/ARP IP/MAC Anti-Spoofing](anti-spoofing.md) | [Next: TX/RX Rate Limiting](rate-limiting.md)

Run Host commands from HGX KVM and DPU commands on BlueField ARM. See the [execution conventions](firmware-and-management.md) before making changes.

## Step 10 — Measure Baseline Throughput on the 2×200GbE DPU LAG

**Purpose:** Establish actual TX/RX throughput before applying rate limits and select a suitable number of TCP streams for comparison.

**Test environment:** HGX202 `171.232.252.202` (iperf3 client) ↔ peer `171.232.252.212` (iperf3 server), public VLAN `1302`; DPU `bond0` has **2×200GbE** with `layer3+4` hashing. Run benchmarks only during an approved test window, because they may consume most of the LAG capacity.

**On the peer — start the iperf3 server:**

```bash
iperf3 -s -B 171.232.252.212 -p 5201
```

**On HGX202 — test TX with multiple streams:**

```bash
# Vary P across 1, 4, 8, 16, 32, 48, 64, 96, 128
iperf3 -c 171.232.252.212 -B 171.232.252.202 \
  -p 5201 -P 32 -t 30 -O 5 --json \
  > /root/bf3-ratelimit/results/baseline_tx_P32.json

# RX: peer → HGX202
iperf3 -c 171.232.252.212 -B 171.232.252.202 \
  -p 5201 -P 32 -t 30 -O 5 -R
```

**TX results by stream count:**

| TCP streams | 1     | 4      | 8      | 16     | **32**     | 48     | 64     | 96     | 128    |
| ----------- | ----- | ------ | ------ | ------ | ---------- | ------ | ------ | ------ | ------ |
| TX (Gb/s)   | 33.47 | 129.41 | 172.85 | 276.44 | **360.43** | 346.22 | 314.46 | 297.95 | 275.25 |

**Baseline validation:** `P=32` achieved the highest TX throughput of **360.43 Gb/s** among the tested configurations; increasing to `P>32` did not improve throughput and produced many TCP retransmissions. RX with `-R -P32` measured **362 Gb/s** in a subsequent test.

**Repeat TX measurements 3 times after rolling back policing (on HGX202):**

```bash
cd /root/bf3-ratelimit
bash run32.sh after_rollback
```

| Run 1 | Run 2 | Run 3 | Average |
| ----------- | ----------- | ----------- | --------------- |
| 332.08 Gb/s | 353.86 Gb/s | 337.83 Gb/s | **341.26 Gb/s** |

The repeated measurements confirm that the baseline recovered after rollback, providing a reference for rate limiting comparisons; differences between runs indicate remaining variation due to TCP load/congestion.

Separate [LACP failover measurements](lacp-failover.md#reported-tcp-and-rdma-measurements) cover TCP and RDMA before, during, and after a switch-port outage. Their traffic parameters and rate-limit state were not recorded, so they are not directly comparable to this TCP baseline.
