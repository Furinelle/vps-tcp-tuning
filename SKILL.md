---
name: vps-tcp-tuning
description: Use when the user asks to perform TCP tuning, 进行TCP调优, tcp调优, VPS网络调优, or to audit, plan, recommend, apply, or verify Linux VPS networking for relay, landing, exit, proxy, or web hosts with SSH access, especially BBR, fq, sysctl, qdisc, MTU/PMTU, iperf3, TBF, HTB, qos-agent, retransmission, throughput, or latency decisions.
---

# VPS TCP Tuning

## Overview

Use evidence, not cargo-cult values. Treat `MTU 1440`, `TBF 1000Mbit`, huge TCP buffers, BBR, fq, HTB, and qos-agent as candidates that must be justified by host role, traffic direction, path tests, and rollback safety. When the user says "进行 TCP 调优", "tcp调优", "VPS 网络调优", or equivalent wording, treat that as an automatic invocation of this skill.

Before live tuning or a concrete plan, read `references/blog-method.md` for the detailed checklist and command patterns distilled from the source article. When the user mentions a one-click tuning script, IPv4 priority, conntrack sizing, RPS/RFS, UDP buffers, MSS clamping, or `Madhatter2099/TCP-Optimize`, also read `references/tcp-optimize-review.md`.

## Mandatory Active Questioning Gate

When this skill is invoked, actively ask the user for missing intent before touching remote hosts. Ask concise grouped questions; do not assume the business path from hostname alone.

Ask for any missing fields:

- `target_ssh`: SSH alias or SSH command for each target.
- `machine_role`: relay, landing, mixed, exit, web, or test peer.
- `traffic_path`: e.g. `user -> relay -> landing -> internet`.
- `critical_direction`: which direction maps to the user's real experience.
- `proxy_software` and `proxy_protocols`: sing-box, xray, realm, gost, Hysteria2, TUIC, WireGuard, nginx/caddy, nftables/iptables, etc.
- `service_ports`: proxy/web/relay/iperf ports.
- `advertised_bandwidth`: practical or advertised up/down/port speed.
- `test_peers`: label, host/IP, iperf3 port, ICMP allowed, SSH access, role.
- `peer_lifecycle`: which peers are long-term/renewing versus temporary/soon-to-expire; only durable peers should drive persistent tuning decisions unless the user explicitly says otherwise.
- `permission_boundary`: inspect only, test only, plan only, apply allowed, reboot allowed, MTU changes allowed, shaping/qos-agent allowed.

Defaults when the user is terse: inspect + test + recommendation only; no persistent changes, no reboot, no MTU change, no HTB/TBF/qos-agent, no cleanup of backups/logs. Never ask for private keys, tokens, or provider credentials.

## Recommendation Before Application Gate

Always produce a recommended configuration before applying persistent changes. The user decides whether to apply it. Even if inspection and testing are allowed, do not write persistent sysctl/qdisc/MTU/qos-agent/service changes until the user explicitly approves the recommended config or says to apply it.

A recommendation must include:

- Evidence summary: role, critical path, PMTU, iperf/counter deltas, bottleneck interpretation.
- Exact candidate config: proposed `/etc/sysctl.d/*.conf` content and any qdisc/systemd/MTU/qos-agent commands or units.
- Non-changes: candidate knobs rejected because evidence is insufficient.
- Risk and interruption notes: whether restart/reboot/service reload is needed.
- Verification plan: exact read-back checks and retests after apply.
- Rollback plan: backup location and restore commands.

After presenting the recommendation, stop and ask the user for approval unless the current user message already contains an explicit approval such as "应用这个推荐配置", "按推荐应用", or "直接应用".

## Workflow

1. **Clarify and scope**: Use the questioning gate. State role, traffic path, and durable-peer assumptions back to the user before tests or changes.
2. **Inspect read-only first**: Collect OS/kernel, CPU/memory, interfaces/MTU/routes, socket summary, sysctl TCP state, qdisc/class/filter state, softnet/RPS/XPS if relevant, existing `/etc/sysctl.conf`, `/etc/sysctl.d/*.conf`, running proxy/web/systemd units, and any `*.profile.md` tuning profile.
3. **Choose peers deliberately**: Use all peers for broad observation only when useful, but let long-term/production peers drive persistent config; do not tune a lasting host around a soon-to-expire VPS.
4. **Test one peer at a time**: Ping if allowed; test PMTU before MTU/MSS decisions; run iperf3 P1/P4 forward and reverse for user-critical directions; record bitrate, retransmits, RTT/cwnd clues, and qdisc/TCP counter deltas during each window.
5. **Interpret by role**: For relays, map ingress/egress to user experience. For landing/exit/web hosts, separate TCP endpoint behavior from UDP/QUIC behavior. Do not generalize from one weak or temporary peer.
6. **Recommend exact config first**: Produce the recommendation bundle. Include exact candidate files/commands, rejected knobs, verification, and rollback. Stop for user approval unless the user already explicitly approved applying the recommendation.
7. **Apply only after approval**: Before persistent changes, back up `/etc/sysctl.conf` and `/etc/sysctl.d/`. Use a consolidated `/etc/sysctl.d/` file, avoid order conflicts, apply with `sysctl --system`, verify SSH/service health, rerun the important tests, and roll back if worse.
8. **Write profile and report**: Record role, durable peers, tests, chosen values, reasoning, caveats, backup path, and rollback commands in a small `/etc/sysctl.d/*.profile.md`. Final reports should prefer aliases and masked IPs.

## Tuning Policy

- Prefer BBR + fq only when the kernel exposes BBR and measurements/role support it.
- Do not infer BBRv3 from `uname -r` alone. Verify the installed kernel package/patch set or implementation provenance; the congestion-control name remains `bbr` across variants.
- Size TCP buffers from measured BDP, memory, concurrency, and role; do not increase buffers to hide path loss.
- Treat fixed percentages of total RAM as a ceiling candidate, not a sizing formula. Remember that `net.ipv4.udp_mem` is expressed in memory pages, while `udp_rmem_min` and `udp_wmem_min` are bytes.
- Keep MTU unchanged when PMTU and real application paths are clean.
- Use HTB/TBF only when local egress shaping is likely to reduce retransmits, queue drops, or backlog. Test caps as a ladder and keep fq below the shaping class.
- Consider qos-agent only for adaptive per-port/per-peer/per-source shaping with a clear target; never deploy it as a default tuning step.
- UDP/QUIC protocols such as HY2/TUIC do not consume Linux TCP buffers, but they still care about MTU, qdisc, CPU scheduling, and local egress shaping.
- Enable IPv4 address-selection preference only after comparing IPv4/IPv6 reachability and latency. `gai.conf` changes `getaddrinfo` address ordering, not authoritative DNS answers.
- Resize conntrack only when NAT/firewall use and `nf_conntrack_count`, table pressure, or insert/drop counters justify it. Load `nf_conntrack` before applying and verify the value after reboot.
- Enable RPS/RFS only when multi-vCPU receive processing is measurably concentrated or dropping and RSS/RX queues are insufficient. Scope it to selected physical interfaces, derive valid CPU bitmaps for the actual CPU count, and size per-queue RFS entries consistently with the global table.
- Apply `ip_forward`, MSS clamp, socket/file limits, and queue expansions only to roles that need them. Persist firewall rules with the host's existing firewall system and prefer per-service `LimitNOFILE` when a specific daemon is the constraint.
- Treat `net.core.default_qdisc` as the default for newly created qdiscs; always inspect the live root qdisc and add explicit interface persistence when the live qdisc must change.
- Before using a third-party one-click script, pin/download and inspect it, inventory every file/rule it owns, and take a real baseline backup. Never let its rollback replace pre-existing values with guessed distribution defaults.

## Common Mistakes

- Applying `MTU 1440`, `TBF 1000Mbit`, or `256MB` buffers because they appeared in a previous case.
- Applying a fixed RAM percentage as TCP/UDP buffer sizing without BDP, concurrency, unit, and memory-pressure checks.
- Assuming every Linux 6.12+ kernel supplies BBRv3 just because the module name is `tcp_bbr`.
- Enabling IPv4 preference, forwarding, conntrack expansion, MSS clamping, RPS/RFS, or million-entry file limits on every host regardless of role and measured pressure.
- Calling removal of owned files plus `cubic/fq_codel` a rollback without restoring the values and rules that existed before the run.
- Running multiple peers concurrently and losing attribution.
- Letting soon-to-expire or non-renewing VPS peers drive persistent tuning for a long-term host.
- Treating absolute retransmit counters as evidence without before/after deltas.
- Assuming qdisc-free retransmits are local queue problems.
- Using TCP iperf results as proof for all UDP/QUIC protocol behavior.
- Killing the current remote SSH shell with `pkill -f` because the iperf pattern appears in the shell script text; prefer iperf3 `--pidfile` and kill that PID only.
- Writing Markdown profile files with unquoted heredocs containing backticks; use quoted heredocs or avoid command-substitution characters.
- Leaving persistent config without a backup path and profile.
