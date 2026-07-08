# Evidence-Based VPS TCP Tuning Method

Source: Lide, “让 AI 帮你调 VPS 网络：中转机和落地机 TCP 调优笔记”, iBytebox, 2026-07-07, https://blog.ibytebox.com/posts/ai-agent-vps-tcp-tuning/. This file is a concise adaptation for agent reuse, not a verbatim copy.

## Input Contract

Collect or ask for:

| Field | Why it matters |
| --- | --- |
| `target_ssh` | Use aliases; never request private keys or secrets. |
| `machine_role` | Relay and landing nodes optimize different directions. |
| `traffic_path` | Needed to map measurements to real UX. |
| `critical_direction` | User download/upload may map to target egress/ingress differently. |
| `proxy_software` / `proxy_protocols` | TCP and UDP/QUIC respond to different knobs. |
| `service_ports` | Identify real services and testing ports. |
| `advertised_bandwidth` | Bound shaping ladder and BDP estimates. |
| `test_peers` | Peer label, IP/host, iperf3 port, ICMP, SSH, role. |
| `permission_boundary` | Inspect, test, recommend, apply, reboot, MTU, shaping, cleanup. Persistent apply still requires explicit approval of the recommendation. |

If these are missing, ask before remote work. If the user already provided some fields, ask only for the missing/high-risk ones. If the user says `进行TCP调优`, `tcp调优`, `VPS网络调优`, or similar, treat it as invoking this skill and start with the active questioning gate.

## Read-Only Inspection

Collect enough evidence to explain current state before changing it:

```bash
hostname; uname -r; cat /etc/os-release
lscpu | sed -n '1,20p'; free -h; swapon --show
ip -br addr show; ip route get 1.1.1.1; ss -s
sysctl net.ipv4.tcp_available_congestion_control \
  net.ipv4.tcp_congestion_control net.core.default_qdisc \
  net.core.rmem_max net.core.wmem_max net.ipv4.tcp_rmem \
  net.ipv4.tcp_wmem net.core.somaxconn \
  net.ipv4.tcp_max_syn_backlog net.core.netdev_max_backlog \
  net.ipv4.tcp_notsent_lowat net.ipv4.ip_forward \
  net.ipv6.conf.all.forwarding net.ipv4.tcp_fastopen \
  net.ipv4.tcp_ecn net.ipv4.tcp_syncookies net.ipv4.tcp_mtu_probing

tc -s qdisc show
dev=$(ip -o route get 1.1.1.1 | awk '{for(i=1;i<=NF;i++) if($i=="dev") print $(i+1)}' | head -1)
tc -s class show dev "$dev" 2>/dev/null || true
tc filter show dev "$dev" 2>/dev/null || true
ethtool -k "$dev" 2>/dev/null || true
cat /proc/net/softnet_stat 2>/dev/null | head
find /etc/sysctl.d -maxdepth 1 -type f -name '*.conf' -print -exec sed -n '1,160p' {} \;
systemctl --type=service --state=running --no-pager | grep -Ei 'sing-box|xray|realm|gost|nodepass|hysteria|tuic|nginx|caddy|apache|iperf3|qos-agent' || true
ls /etc/sysctl.d/*.profile.md 2>/dev/null | xargs -r sed -n '1,160p'
```

## PMTU and iperf3 Tests

Run peers sequentially. Snapshot qdisc/TCP counters before and after each test window.

PMTU ladder for IPv4:

```bash
tracepath -n <peer> || true
for s in 1472 1452 1432 1412 1392 1352 1332 1312 1292; do
  mtu=$((s+28))
  ping -M do -s "$s" -c 2 -W 1 <peer> >/dev/null 2>&1 \
    && echo "payload=$s mtu=$mtu OK" \
    || echo "payload=$s mtu=$mtu FAIL"
done
```

iperf3 pattern; adapt direction to user-critical path:

```bash
iperf3 -c <peer> -p <port> -t 12 -O 2 -P 1 -J
iperf3 -c <peer> -p <port> -t 12 -O 2 -P 4 -J
iperf3 -c <peer> -p <port> -t 12 -O 2 -P 1 -R -J
iperf3 -c <peer> -p <port> -t 12 -O 2 -P 4 -R -J
```

Record bitrate, retransmits, cwnd/RTT clues, startup behavior, single-flow vs multi-flow differences, qdisc drops/backlog deltas, and TCP retransmission counter deltas.

## Interpretation Rules

| Evidence | Likely meaning |
| --- | --- |
| Single-flow low, multi-flow high | BDP, per-flow path limits, loss recovery, or congestion-control behavior. |
| qdisc drop/backlog increases during test | Local egress queue/shaping may matter. |
| qdisc drop/backlog stays zero but retransmits high | Suspect path, upstream, or remote receiver before local buffer/shaping changes. |
| High retransmits with low cwnd | Loss/congestion is more likely than missing buffer. |
| One peer bad, others clean | Do not downsize global capacity from one weak peer. |
| HY2/TUIC/QUIC issue | Validate MTU/qdisc/CPU and app loss; TCP buffer may be irrelevant. |

Relay hosts: identify where traffic enters and leaves; a user download may correspond to relay egress toward landing. Kernel forwarding means TCP buffer/BBR might not affect forwarded TCP the same way userspace proxy termination does.

Landing hosts: if they terminate or re-originate TCP, BBR/fq/buffers/notsent/TFO/ECN can matter more directly. Separate web/proxy TCP behavior from UDP/QUIC behavior.

## Candidate Tuning Decisions

- BBR/fq: prefer when available and appropriate; use bbr3 only if exposed by kernel.
- Buffers: estimate from bandwidth-delay product, memory, role, and concurrency. Small 100M hosts often need conservative ceilings. 1G hosts may justify 64-128MB. Higher values need evidence.
- MTU: keep 1500 when clean. Consider 1450-1460 for mild tunnel/provider overhead, or 1400-1440 for nested encapsulation/consumer ISP/UDP paths, only with evidence.
- HTB/TBF: test practical stable uplink with 95/90/85/80/75% ladder. Choose the highest cap that lowers retransmits/drops without harming critical throughput. Keep fq as the child qdisc.
- qos-agent: reserve for adaptive per-peer/per-port/per-source control; do not deploy by default.

## Recommendation and Safe Apply Process

1. Explain the planned change and measurements that justify it.
2. Present exact recommended config before applying: proposed `/etc/sysctl.d/*.conf` content, any qdisc/systemd/MTU/qos-agent commands, rejected candidate knobs, risk/interruption notes, verification plan, and rollback plan.
3. Stop for user approval unless the current user message explicitly says to apply the recommendation.
4. After approval, back up `/etc/sysctl.conf` and `/etc/sysctl.d/`.
5. Write one consolidated sysctl.d file for active tuning; avoid order-dependent conflicts.
6. Apply with `sysctl --system`.
7. Confirm SSH and critical services still work.
8. Read back effective sysctl/qdisc values.
9. Rerun the most important tests.
10. Roll back or revise if worse.
11. Write `/etc/sysctl.d/*.profile.md` with role, tests, chosen values, reasoning, caveats, backup path, and rollback commands.

## Final Report Checklist

- Inspected hosts and roles.
- Tests run and user-critical results.
- Changes applied and explicit non-changes.
- Retransmission, qdisc, PMTU, and bottleneck interpretation.
- Backup/profile paths and rollback commands.
- Remaining uncertainty and next tests.
- Mask IPs unless the user asked for raw addresses.
