# Final Results

[Overview](../README.md) | [Rollback](rollback/README.md) | [Previous: TX/RX Rate Limiting](rate-limiting.md)

| Item | Achieved State |
| --------------------------------------------- | --------------------------------------------------------------------------------------------- |
| BlueField Mode | **DPU Mode**, with the eSwitch managed by ECPF |
| DPU management | RShim/TMFIFO and ARM SSH operational; restored after Host reboot |
| DPU LAG                                       | **802.3ad, Hash Mode, p0+p1, 2×200GbE**                                                       |
| LACP | Both members in the same aggregator; collecting/distributing |
| Forwarding                                    | `Host PF0 ↔ pf0hpf ↔ ovsbr1 ↔ DPU bond0 ↔ ToR`                                                |
| Public/private VLANs | Both VLANs, IP addresses, and gateway preserved; Host parent is PF0 |
| OVS | Forwarding with hardware offload for eligible flows |
| IPv4/ARP anti-spoofing | 6 outbound rules installed; spoofed IPv4/ARP packets dropped |
| Anti-spoofing hardware offload | IPv4 ALLOW/DROP have `in_hw` evidence; ARP DROP used software TC in testing |
| Anti-spoofing after Host reboot | All 6 rules remained; DPU Boot ID and OVS PID unchanged |
| Persistence | **Both ARM and HGX rebooted;** `verify-boot` **PASS** |
| 2×200G LAG bandwidth | Nominal aggregate link capacity of 400 Gb/s in each direction; peak TX P32 **360.43 Gb/s**, RX P32 **362 Gb/s** |
| Repeated TX baseline after rollback | **341.26 Gb/s** (3 runs, P32) |
| TX 100 Gb/s through OVS ingress policing | Failed: TC **31.28 Gb/s**, iperf3 **29.32 Gb/s** |
| TX 100 Gb/s through NVIDIA Host PF0 `max_tx_rate` | **PASS: 93.72 Gb/s** average over 3 runs; PF0 `MaxTxRate=100000 Mb/s` |
| RX 10 Gb/s through OpenFlow Meter | **PASS: 9.64 Gb/s** |
| RX 100 Gb/s through OpenFlow Meter | **Failed: 30.16 Gb/s**, TC displayed `31280Mbit` |

**Conclusion:** DPU-managed 2×200GbE LAG and LAG persistence are complete; IPv4/ARP anti-spoofing has been deployed and tested; peak TX of **360.43 Gb/s** and RX of **362 Gb/s** were measured with 32 TCP streams. **TX rate limiting at 100 Gb/s passed** through Host PF0 (`93.72 Gb/s` average), while **RX 100 Gb/s failed** through OpenFlow Meter because TC actually displayed only `31.28 Gb/s`. RX metering at 10 Gb/s worked (`9.64 Gb/s`).
