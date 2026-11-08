# LACP Failover on BlueField-3

[Overview](../README.md) | [Rollback](rollback/README.md) | [Previous: Persistence and Reboot Validation](persistence.md) | [Next: IPv4/ARP IP/MAC Anti-Spoofing](anti-spoofing.md)

This document incorporates the supplied [LACP failover test report](../bao_cao_lacp_failover_bluefield3.md). Measurements below are reported observations; no new hardware test was performed when preparing this guide.

## Objective and Configuration

Evaluate whether TCP and RDMA traffic can continue through LACP when a physical member fails. The supplied report covers disabling the switch port connected to `p0`; it does not document a separate `p1` failure test.

| Component | Test Configuration |
| --- | --- |
| Device | NVIDIA BlueField-3 DPU |
| Physical ports | `p0`, `p1`, each rated at 200 Gb/s |
| Bond | `bond0` |
| Bonding mode | IEEE 802.3ad (LACP) |
| Transmit hash policy | `layer3+4` |
| LACP rate | `fast` |
| `min_links` | `0` |
| OVS bridge | `ovsbr1`, containing `bond0` and `pf0hpf` |
| Test tools | `ping` for ICMP, `iperf3` for TCP, `ib_write_bw` for RDMA |

## Test Sequence and Link State

1. Observe the baseline with `p0`, `p1`, and `bond0` up while traffic is running.
2. Disable the switch port connected to `p0`. Observe DPU link state, LAG mapping, ICMP reachability, and TCP/RDMA throughput.
3. Re-enable the same switch port and observe traffic and member recovery.

| Phase | `p0` | `p1` | `bond0` | LAG Mapping | Evidence Status |
| --- | --- | --- | --- | --- | --- |
| Initial | UP | UP | UP | `1, 2` | Reported baseline |
| Switch port connected to `p0` disabled | DOWN | UP | UP | `2` | Reported observation; ICMP continued |
| Switch port re-enabled | UP | UP | UP | `1, 2` | Expected recovery; confirming logs were not supplied |

The source's throughput table lists a third phase with `p0 UP`, but its link-state section explicitly says recovery logs are unavailable. The third-phase measurements are retained below; they do not independently confirm that both LACP members rejoined the forwarding aggregator.

## Reported TCP and RDMA Measurements

All throughput values use Gb/s; decimal commas in the source have been converted to decimal points.

| Phase | Reported DPU State | TCP — `iperf3` | RDMA — `ib_write_bw` | Reported Connectivity |
| --- | --- | --- | --- | --- |
| 1. Initial | `p0 UP`, `p1 UP`, `bond0 UP` | 97.9 | 301.22 | TCP and RDMA transferring data |
| 2. Switch port connected to `p0` disabled | `p0 DOWN`, `p1 UP`, `bond0 UP` | 93.6 | 60.29 | TCP/RDMA throughput remained; ICMP continued |
| 3. Switch port connected to `p0` re-enabled | `p0 UP`, `p1 UP`, `bond0 UP` | 344 | 40.12 | TCP/RDMA transfer and ICMP reported as continuing |

The measurements vary substantially, especially RDMA throughput. They support the observation that traffic was still passing with `p0` down, but do not establish a stable failover throughput target.

The arithmetic sums of the TCP and RDMA figures are **399.12**, **153.89**, and **384.12 Gb/s** for phases 1–3. These are calculated sums, not independently measured aggregate bandwidth: the report does not establish that the tools measured concurrent traffic in the same direction over matching time intervals. In particular, the phase-2 figures do not demonstrate 200 Gb/s of application throughput.

The report does not provide test duration, stream counts, RDMA device/GID settings, traffic direction, rate-limit state, packet-loss counts, connection restart records, or interruption timing. Keep these results separate from the [TCP baseline benchmarks](benchmarks.md) and [rate-limit tests](rate-limiting.md); their test conditions have not been shown to match.

## Recovery and Follow-up Validation

For a repeat test, use an approved maintenance window with HGX KVM and ARM management access available. Disable only the identified switch port and restore that port after the test. Switch CLI commands depend on the switch platform and are not included in the supplied report.

Capture the following on DPU ARM before the interruption, while `p0` is down, and after the switch port is restored. These are suggested inspection commands, not additional recorded test results:

```bash
cat /proc/net/bonding/bond0
ip -d link show p0
ip -d link show p1
ip -d link show bond0
cat /sys/class/net/bond0/bonding/ad_num_ports
sudo ovs-vsctl show
```

After restoring the switch port, confirm both members are up, in the same aggregator, and collecting/distributing. Then run the existing validation on the HGX Host:

```bash
cd /root/bf3-lag
bash bf3-lag.sh verify-runtime
```

`verify-runtime` expects two forwarding members. A failure of that check during the intentional one-port outage is not by itself a failure of single-link traffic continuity; use it after both ports should have recovered.

If the restored port does not rejoin, inspect the switch LACP state, DPU bond state, and physical link first. For an already persistent deployment, follow [post-reboot/persistent LAG recovery](rollback/persistence.md#case-e) if repair is required. The [runtime rollback procedure](rollback/runtime.md#case-b) applies before persistence; it is not the routine cleanup for this failover test.

To complete the evidence, retain timestamped switch/DPU state and traffic logs through all three phases, record loss and interruption duration, verify whether the same TCP/RDMA sessions remained active, and repeat with `p1` as the failed member. These are follow-up checks, not completed results in the source report.

## Assessment

The supplied test reports continued ICMP reachability and nonzero TCP/RDMA throughput when `p0` went down while `p1` and `bond0` remained up. Recovery to a fully forwarding two-member LAG, zero-loss continuity, failover time, and equivalent behavior for a `p1` failure remain unverified by the supplied evidence.
