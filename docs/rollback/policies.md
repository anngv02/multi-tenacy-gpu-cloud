# Anti-Spoofing and Rate-Limit Rollback

[Overview](../../README.md) | [Choose a recovery scenario](README.md)

<a id="case-f"></a>

## Case F — Anti-spoofing blocks valid traffic

This report's six rules use cookie `0xb3a52026`. Confirm that the selected rules belong to this test and that the expected baseline forwarding rules still exist. On DPU ARM:

```bash
ovs-ofctl -O OpenFlow13 dump-flows ovsbr1 'cookie=0xb3a52026/-1'
ovs-ofctl -O OpenFlow13 dump-flows ovsbr1
ovs-ofctl -O OpenFlow13 --bundle del-flows ovsbr1 'cookie=0xb3a52026/-1'
ovs-ofctl -O OpenFlow13 dump-flows ovsbr1 'cookie=0xb3a52026/-1'
```

**Validation:** No flows remain with this cookie; valid Host traffic works and DPU LAG remains intact. This temporarily removes the test's IPv4/ARP source validation. Keep other tenants' rules and the baseline forwarding policy. Do not delete all bridge flows or introduce an unconditional forwarding rule as a shortcut.

Correct the VLAN/IP/MAC bindings in `/root/bf3-antispoof/apply.sh`, reapply that script on ARM, and repeat the valid/spoofed packet checks before declaring protection restored. This anti-spoofing script is documented in the deployment report; it is not included in the LAG ZIP.

<a id="case-g"></a>

## Case G — TX throughput drops after rate limiting

**OVS ingress policing:** Restore the saved values on DPU ARM. The report's pre-test values were rate `0`, burst `0`; use the saved values if the local baseline differs.

```bash
cd /root/bf3-ratelimit
PREVIOUS_RATE=$(cat old_rate.txt) || exit 1
PREVIOUS_BURST=$(cat old_burst.txt) || exit 1
[[ "$PREVIOUS_RATE" =~ ^[0-9]+$ && "$PREVIOUS_BURST" =~ ^[0-9]+$ ]] || exit 1
ovs-vsctl set Interface pf0hpf \
  ingress_policing_rate="$PREVIOUS_RATE" \
  ingress_policing_burst="$PREVIOUS_BURST"
ovs-vsctl get Interface pf0hpf ingress_policing_rate
ovs-vsctl get Interface pf0hpf ingress_policing_burst
tc -s filter show dev pf0hpf ingress
```

**NVIDIA PF0 `max_tx_rate`:** Review `/root/bf3-ratelimit/pf0-before.txt`. If it confirms the report's original `MaxTxRate=0`, remove the test cap on DPU ARM:

```bash
cat /root/bf3-ratelimit/pf0-before.txt
echo 0 > /sys/class/net/p0/smart_nic/pf/max_tx_rate
cat /sys/class/net/p0/smart_nic/pf/config
```

If the saved rate was nonzero, restore that rate in Mb/s instead of `0`. These are independent controls: removing OVS policing does not remove a PF0 cap, and removing the PF0 cap does not remove OVS policing. Restore both only when both were changed by the test.

**Validation:** Read back the intended values, verify anti-spoofing and both LAG members, and repeat the Host benchmark in a test window. Compare with the baseline under comparable load; the reported 341.26 Gb/s average is a reference, not a guaranteed result. `run32.sh` and its result files are separate report artifacts, not bundled in the LAG ZIP.

<a id="case-h"></a>

## Case H — RX meter rollback or standalone TC action cleanup

The report does not record the RX meter ID or flow cookie. Identify them from the actual configuration and pre-test records; do not reuse the anti-spoofing cookie or guess an ID.

On DPU ARM:

```bash
ovs-ofctl -O OpenFlow13 dump-flows ovsbr1
ovs-ofctl -O OpenFlow13 dump-meters ovsbr1
ovs-ofctl -O OpenFlow13 meter-stats ovsbr1
```

1. Identify every rule referencing the test meter (`meter:<id>`), including any shared use.
2. If the test modified existing forwarding rules, restore those specific rules' pre-test actions first. A `dump-flows` diagnostic listing is not an `add-flows` restore file. Use the original rule definitions and actual table/match values.
3. If the test only added rules with a dedicated, nonzero cookie, delete only that cookie after confirming baseline forwarding will handle the traffic. The following prompts intentionally require the actual values:

```bash
# DPU ARM; only for added test rules with a dedicated cookie
read -r -p 'Dedicated RX test cookie (hex, excluding the anti-spoofing cookie): ' RX_COOKIE
[[ "$RX_COOKIE" =~ ^0x[0-9a-fA-F]{1,16}$ ]] || exit 1
[[ ! "$RX_COOKIE" =~ ^0x0+$ ]] || exit 1
[[ ! "${RX_COOKIE,,}" =~ ^0x0*b3a52026$ ]] || exit 1
ovs-ofctl -O OpenFlow13 dump-flows ovsbr1 "cookie=$RX_COOKIE/-1"
# After reviewing the selected rules:
ovs-ofctl -O OpenFlow13 --bundle del-flows ovsbr1 "cookie=$RX_COOKIE/-1"
```

4. After confirming that no remaining rule references the test meter and no other workload needs it, delete that meter alone:

```bash
read -r -p 'Unused RX test meter ID (decimal): ' RX_METER_ID
[[ "$RX_METER_ID" =~ ^[1-9][0-9]*$ ]] || exit 1
ovs-ofctl -O OpenFlow13 del-meters ovsbr1 "meter=$RX_METER_ID"
ovs-ofctl -O OpenFlow13 dump-meters ovsbr1
ovs-ofctl -O OpenFlow13 dump-flows ovsbr1
```

5. For the standalone TC action from Step 12, inspect `tc -s actions ls action police`. Delete index `62000` only if it is the test action and still has `bind 0`; if absent, skip deletion:

```bash
tc -s actions ls action police
# Only after confirming index 62000 belongs to this test and has bind 0:
tc actions delete action police index 62000
tc -s actions ls action police
```

**Validation:** RX forwarding works, the selected meter/action is absent, and unrelated policies remain. Repeat RX throughput measurement during a test window. Avoid global meter deletion or deleting the entire ingress qdisc. Targeted cookie/meter operations follow the [Open vSwitch ovs-ofctl manual](https://www.openvswitch.org/support/dist-docs/ovs-ofctl.8.html).
