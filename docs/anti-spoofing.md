# IPv4/ARP IP/MAC Anti-Spoofing

[Overview](../README.md) | [Rollback](rollback/README.md) | [Previous: LACP Failover](lacp-failover.md) | [Next: Throughput Benchmarks](benchmarks.md)

Run Host commands from HGX KVM and DPU commands on BlueField ARM. See the [execution conventions](firmware-and-management.md) before making changes.

## Step 9 — Configure and Test IP/MAC Anti-Spoofing on BlueField-3

**Purpose:** Restrict source IP/MAC addresses per VLAN in the `Host → Network` direction on OVS `ovsbr1`, without changing DPU LAG. The test configuration uses public VLAN `1302` (`171.232.252.202`) and private VLAN `3048` (`10.29.252.239`), both with MAC `58:a2:e1:d6:b2:c8`.

**On DPU ARM — create and run** `/root/bf3-antispoof/apply.sh`:

```bash
sudo -i
mkdir -p /root/bf3-antispoof
cat > /root/bf3-antispoof/apply.sh <<'SH'
#!/usr/bin/env bash
set -euo pipefail
BR=ovsbr1
COOKIE=0xb3a52026
HP=$(ovs-vsctl --bare get Interface pf0hpf ofport)
UP=$(ovs-vsctl --bare get Interface bond0 ofport)
[[ "$HP" =~ ^[0-9]+$ ]] && (( HP > 0 )) || exit 1
[[ "$UP" =~ ^[0-9]+$ ]] && (( UP > 0 )) || exit 1

cat > /root/bf3-antispoof/flows.of <<EOF
cookie=$COOKIE,priority=310,in_port=$HP,dl_vlan=1302,dl_src=58:a2:e1:d6:b2:c8,ip,nw_src=171.232.252.202,actions=NORMAL
cookie=$COOKIE,priority=310,in_port=$HP,dl_vlan=1302,dl_src=58:a2:e1:d6:b2:c8,arp,arp_spa=171.232.252.202,arp_sha=58:a2:e1:d6:b2:c8,actions=NORMAL
cookie=$COOKIE,priority=310,in_port=$HP,dl_vlan=3048,dl_src=58:a2:e1:d6:b2:c8,ip,nw_src=10.29.252.239,actions=NORMAL
cookie=$COOKIE,priority=310,in_port=$HP,dl_vlan=3048,dl_src=58:a2:e1:d6:b2:c8,arp,arp_spa=10.29.252.239,arp_sha=58:a2:e1:d6:b2:c8,actions=NORMAL
cookie=$COOKIE,priority=300,in_port=$HP,ip,actions=drop
cookie=$COOKIE,priority=300,in_port=$HP,arp,actions=drop
EOF
ovs-ofctl -O OpenFlow13 --bundle add-flows "$BR" /root/bf3-antispoof/flows.of
ovs-ofctl -O OpenFlow13 dump-flows "$BR" "cookie=$COOKIE/-1"
SH
chmod +x /root/bf3-antispoof/apply.sh
bash /root/bf3-antispoof/apply.sh
```

**Expected results:** **6 OpenFlow rules** are installed (4 rules with `priority=310,actions=NORMAL` for valid IP/ARP bindings, and 2 rules with `priority=300,actions=drop` for all other IPv4/ARP traffic from `pf0hpf`). Uplink `bond0` and LAG `p0+p1` remain unchanged.

**On the HGX Host — create** `/root/bf3-antispoof/spoof_test.py` **to send test packets:**

```bash
sudo -i
apt-get install -y python3-scapy
mkdir -p /root/bf3-antispoof
cat > /root/bf3-antispoof/spoof_test.py <<'PYTEST'
#!/usr/bin/env python3
import os, sys
from scapy.all import Ether, Dot1Q, IP, UDP, ARP, Raw, sendp

IFACE = 'enp39s0f0np0'  # PF0, add VLAN tags using Dot1Q
MAC = '58:a2:e1:d6:b2:c8'
FAKE_MAC = '02:11:22:33:44:55'
GOOD_IP = '171.232.252.202'
FAKE_IP = '198.51.100.99'  # Address reserved for testing
GW_IP = '171.232.252.254'
GW_MAC = os.environ.get('GW_MAC')

cases = {
    'legit': (1302, MAC, GOOD_IP),
    'ip':    (1302, MAC, FAKE_IP),
    'mac':   (1302, FAKE_MAC, GOOD_IP),
    'vlan':  (3048, MAC, GOOD_IP),
}
if len(sys.argv) != 2 or sys.argv[1] not in (*cases, 'arp-ip'):
    sys.exit('Usage: spoof_test.py {legit|ip|mac|vlan|arp-ip}')
mode = sys.argv[1]
if mode == 'arp-ip':
    pkt = (Ether(src=MAC, dst='ff:ff:ff:ff:ff:ff') / Dot1Q(vlan=1302)
           / ARP(op=1, hwsrc=MAC, psrc=FAKE_IP, pdst=GW_IP))
else:
    if not GW_MAC:
        sys.exit('Missing GW_MAC environment variable')
    vid, smac, sip = cases[mode]
    pkt = (Ether(src=smac, dst=GW_MAC) / Dot1Q(vlan=vid)
           / IP(src=sip, dst=GW_IP) / UDP(sport=43000, dport=59999)
           / Raw(load=('BF3-ANTISPOOF-' + mode).encode()))
sendp(pkt, iface=IFACE, count=5, inter=0.2, verbose=False)
print('Sent test:', mode)
PYTEST
chmod +x /root/bf3-antispoof/spoof_test.py
```

**Test on the HGX Host:**

```bash
ping -I bond0.1302 -c 3 171.232.252.254
export GW_MAC="$(ip -4 neigh show 171.232.252.254 dev bond0.1302 | awk '{for(i=1;i<=NF;i++) if($i=="lladdr"){print $(i+1); exit}}')"
test -n "$GW_MAC" || { echo 'Could not obtain the gateway MAC address'; exit 1; }
cd /root/bf3-antispoof
/usr/bin/python3 spoof_test.py legit    # Valid IPv4
/usr/bin/python3 spoof_test.py ip       # Spoofed source IP
/usr/bin/python3 spoof_test.py mac      # Spoofed source MAC
/usr/bin/python3 spoof_test.py vlan     # Incorrect VLAN/IP binding
/usr/bin/python3 spoof_test.py arp-ip   # Spoofed ARP sender IP
```

**Check counters and hardware offload on DPU ARM (capture before and after each test):**

```bash
ovs-ofctl -O OpenFlow13 dump-flows ovsbr1 'cookie=0xb3a52026/-1'
ovs-appctl bridge/dump-flows --offload-stats ovsbr1
tc -s filter show dev pf0hpf ingress
```

**Expected results:** Valid packets increment IPv4 ALLOW; the `ip`, `mac`, and `vlan` tests increment **IPv4 DROP**; the `arp-ip` test increments **ARP DROP**. TC reports `in_hw`/`Sent hardware` for IPv4 forwarding and IPv4 DROP. ARP DROP blocks packets but shows `not_in_hw` (software TC) in this test. Counters do not need to have fixed values.

**Check after rebooting the HGX Host:**

```bash
# On the HGX Host (through KVM):
reboot
# Once the Host is back up, run on the Host:
cd /root/bf3-lag && bash bf3-lag.sh verify-boot
# On DPU ARM:
cat /proc/sys/kernel/random/boot_id
ps -o pid,lstart,cmd -p "$(pidof ovs-vswitchd)"
ovs-ofctl -O OpenFlow13 dump-flows ovsbr1 'cookie=0xb3a52026/-1'
```

**Observed results:** Rebooting the HGX Host **did not restart BlueField ARM or the OVS process** in this test (Boot ID and OVS PID remained unchanged); DPU LAG still passed validation, and **all 6 anti-spoofing rules remained**, without manual reapplication. This result **does not prove anti-spoofing persistence after a DPU ARM/OVS reboot**, because no service has been installed to restore the rules automatically.
