# BlueField-3 DPU-managed LAG on HGX H200

Deployment report and operating procedures for a single HGX H200 node with a BlueField-3 DPU and two 200GbE uplinks.

**System:** HGX H200.

The deployment moves LACP from the Host to the DPU, preserves public/private VLANs, adds reboot persistence and IPv4/ARP anti-spoofing, and evaluates throughput and rate limiting.

## Documentation

Read the deployment guides in order for a new installation. For an existing system, start with the relevant topic or the rollback guide.

| Document | Contents |
| --- | --- |
| [Initial environment](docs/environment.md) | Hardware, operating systems, firmware baseline, VLANs, and original architecture |
| [Firmware and management](docs/firmware-and-management.md) | Steps 1–3: DPU Mode, RShim/TMFIFO, Hash Mode, and command conventions |
| [LAG deployment](docs/lag-deployment.md) | Steps 4–6: backups, staging, cutover, and runtime validation |
| [Persistence](docs/persistence.md) | Steps 7–8: startup configuration and reboot validation |
| [Anti-spoofing](docs/anti-spoofing.md) | Step 9: IPv4/ARP rules, packet tests, and offload checks |
| [Benchmarks](docs/benchmarks.md) | Step 10: TCP stream scaling and baseline measurements |
| [Rate limiting](docs/rate-limiting.md) | Steps 11–12: OVS policing, PF0 TX limits, and TC rate representation |
| [Results](docs/results.md) | Measured outcomes and remaining limitations |
| [Recovery and rollback](docs/rollback/README.md) | Nine scenarios covering preparation, cutover, management, persistence, policies, and NIC Mode |

## Script Package

Download [bf3-lag.zip](./bf3-lag.zip) and extract it to `/root/bf3-lag` on the HGX Host. The archive includes its own overview, operating guide, rollback documents, and `SHA256SUMS`.

The anti-spoofing scripts (`/root/bf3-antispoof/apply.sh` on the DPU and `/root/bf3-antispoof/spoof_test.py` on the Host) are documented in the [anti-spoofing guide](docs/anti-spoofing.md). `/root/bf3-ratelimit/run32.sh` and `results/` are separate Host benchmark artifacts; they are not included in the LAG ZIP.

## Objectives

- Switch NVIDIA BlueField-3 from **NIC Mode** to **DPU Mode**, transferring embedded switch management to the DPU ARM OS.
- Move **2×200GbE** LACP from Linux `bond0` on the HGX Host to **DPU-managed LAG**, allowing the Host to use a single PF for both public and private VLANs.
- Preserve IP addresses, the default route, and management access during a controlled transition using **KVM + RShim/TMFIFO**.
- Automatically restore the DPU bond, OVS datapath, Host VLANs, and management channel after system reboots.
- Deploy IPv4/ARP IP/MAC anti-spoofing on the DPU as a foundation for further policy development independent of the Host/VM.
- Measure TCP throughput with multiple streams over the 2×200GbE DPU LAG; test **TX 100 Gb/s and RX 100 Gb/s** limits, compare OpenFlow/TC/hardware offload behavior, identify viable mechanisms, and document current limitations.

## Current Outcome

DPU-managed 2×200GbE LAG and reboot persistence were validated. Peak throughput reached **360.43 Gb/s TX** and **362 Gb/s RX** with 32 TCP streams. PF0 TX limiting at 100 Gb/s passed; RX metering at 100 Gb/s did not. See [results and limitations](docs/results.md) for the complete measurements.

Use HGX KVM and RShim/TMFIFO for disruptive work. Review the applicable [rollback scenario](docs/rollback/README.md) before cutover or policy changes.
