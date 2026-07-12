# TCP-Optimize Review Notes

Source reviewed: [`Madhatter2099/TCP-Optimize`](https://github.com/Madhatter2099/TCP-Optimize), `tcp.sh` at commit [`e43b4ba`](https://github.com/Madhatter2099/TCP-Optimize/blob/e43b4ba0a6ac0c4474bdd3b0ba7b5d7c902627cb/tcp.sh), reviewed 2026-07-12.

Use these notes when a user asks to copy, compare, audit, or run that script. They are a static review of the cited commit; inspect the current upstream revision again before acting.

## Ideas Worth Reusing

- Separate features into visible modules: IPv4 address preference, BBR/fq, broader kernel tuning, RPS/RFS, update, and rollback.
- Show effective live state instead of reporting only that a file was written.
- Check BBR availability with `modprobe` plus `tcp_available_congestion_control`.
- Load `nf_conntrack` before reading or applying its sysctls.
- Inspect RX queues and expose RPS/RFS as a separate decision rather than burying it in sysctl.
- Make cleanup explicit and keep an inventory of created files, symlinks, and firewall rules.

These are workflow patterns. Their concrete values still require host-role and path evidence.

## Do Not Adopt as Universal Defaults

| Upstream behavior | Skill treatment |
| --- | --- |
| TCP socket ceilings equal 5% of RAM, capped at 256 MiB | Estimate BDP and concurrency first; use memory as a safety bound. |
| `udp_mem = 65536 131072 262144` described like ordinary buffer bytes | `udp_mem` uses pages and is calculated at boot by default; inspect real UDP pressure before overriding. |
| `somaxconn=65535`, `netdev_max_backlog=65535`, global `nofile=1048576` | Require listen/drop/softnet/service-limit evidence. |
| Enable IPv4 forwarding and IPv6 forwarding in the general profile | Enable only for an actual router/relay role. |
| Add global iptables TCPMSS clamp whenever iptables exists | Require forwarding/tunnel PMTU evidence and use the host's persistent firewall owner. |
| Enable `tcp_mtu_probing=1` for every host | Treat as a candidate for observed PMTUD black holes; it is TCP-specific. |
| Resize conntrack and its buckets only from total RAM | Inspect NAT/firewall use, count/max ratio, insert failures, drops, and boot persistence. |
| Apply RPS to every interface and all CPUs | Check RSS/RX queues, IRQ affinity, per-CPU softirq load, NUMA/locality, and valid bitmap width. |
| Infer BBRv3 from kernel version `>= 6.12` | Verify kernel provenance or patch/package documentation. The sysctl value alone identifies `bbr`, not its generation. |
| Roll back to `cubic + fq_codel` | Restore the captured pre-change values and files instead of guessing defaults. |

## Static Implementation Findings

### Live qdisc versus default qdisc

Writing `net.core.default_qdisc=fq` does not prove the already-created external interface now has `fq`. Verify with:

```bash
dev=$(ip -o route get 1.1.1.1 | awk '{for(i=1;i<=NF;i++) if($i=="dev") print $(i+1)}' | head -1)
sysctl -n net.core.default_qdisc
tc -s qdisc show dev "$dev"
```

If an explicit live replacement is recommended, provide a persistent systemd/network-manager/networkd mechanism and rollback for that interface.

### IPv4 preference semantics

`/etc/gai.conf` controls address sorting used by libc `getaddrinfo`. It does not make a DNS server return only A records. Compare both families to the real service before recommending a preference:

```bash
getent ahosts <hostname>
ping -4 -c 5 <hostname>
ping -6 -c 5 <hostname>
curl -4 -o /dev/null -sS -w 'v4 connect=%{time_connect} total=%{time_total}\n' https://<hostname>/
curl -6 -o /dev/null -sS -w 'v6 connect=%{time_connect} total=%{time_total}\n' https://<hostname>/
```

### RPS/RFS correctness

The reviewed script computes the CPU mask with shell arithmetic and applies it broadly. That approach becomes fragile for large CPU counts, may include unsuitable interfaces, and reports an interface configured even if no queue file was successfully changed.

Collect this first:

```bash
nproc
find /sys/class/net/<dev>/queues -maxdepth 1 -type d -printf '%f\n'
grep -Ei '<dev>|virtio' /proc/interrupts
grep -E 'NET_RX|NET_TX' /proc/softirqs
cat /proc/net/softnet_stat
```

Kernel guidance says RPS can be redundant when RSS already maps a receive queue per CPU. For RFS, configure the global table and per-queue tables consistently; on a single RX queue, the per-queue value would normally match the global value.

### Configuration ownership and rollback

Before using any external tuning script, capture:

```bash
RUN_ID=$(date -u +%Y%m%dT%H%M%SZ)
backup=/root/network-tuning-$RUN_ID/pre-script
mkdir -p "$backup"
cp -a /etc/sysctl.conf /etc/sysctl.d /etc/security/limits.d /etc/gai.conf "$backup"/ 2>/dev/null || true
sysctl -a > "$backup/sysctl-a.txt" 2>/dev/null || true
tc -s qdisc show > "$backup/tc-qdisc.txt"
iptables-save > "$backup/iptables-save.txt" 2>/dev/null || true
nft list ruleset > "$backup/nft-ruleset.txt" 2>/dev/null || true
```

Also record the exact script commit and SHA-256. Prefer a downloaded, inspected file over a moving `curl | bash` target.

## Evidence Gate for Each Extra Module

| Module | Minimum evidence before recommending apply |
| --- | --- |
| IPv4 preference | Real service IPv6 is broken or consistently worse; no IPv6-only dependency. |
| Queue/backlog expansion | Listen overflow, softnet drop/squeeze, or queue pressure during the critical workload. |
| Conntrack expansion | Host uses conntrack and count/insert/drop evidence approaches the effective limit. |
| RPS/RFS | Multi-vCPU, insufficient RX queues, concentrated NET_RX/IRQ load, and improved controlled retest. |
| MSS clamp | Forwarded/tunneled TCP path has a demonstrated PMTU/MSS problem. |
| Global/per-service limits | Actual daemon or accept path approaches the current limit. |

## Primary References

- [Linux kernel IP sysctl documentation](https://docs.kernel.org/networking/ip-sysctl.html)
- [Linux kernel networking scaling documentation](https://docs.kernel.org/networking/scaling.html)
- [Google BBR repository and v3 branch](https://github.com/google/bbr)
