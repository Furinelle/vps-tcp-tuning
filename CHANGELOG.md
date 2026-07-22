# Changelog

## 2026-07-22

### Added

- Added a commit-reviewed reference for [`Eric86777/vps-tcp-tune`](https://github.com/Eric86777/vps-tcp-tune) (`net-tcp-tune.sh` v5.4.4) in `references/vps-tcp-tune-review.md`.
- Documented the bandwidth × service-region buffer candidate ladder (Asia / overseas), BDP cross-check (`Mbps × RTT_ms × 125`), live `fq` apply + boot persistence, sysctl conflict hygiene, and owned-file inventory for menu 3 artifacts.
- Extended active-questioning fields with `service_region` / RTT class and optional third-party-script / kernel-swap permission.
- Expanded skill triggers for `BBR调优`, XanMod, one-click `bbr` menus, Realm timeout fix, and menu `3`/`66` style requests.

### Changed

- Upgraded `SKILL.md` workflow: size buffers from evidence after peer tests; verify live qdisc after `default_qdisc=fq`; prefer `tcp_mtu_probing` over cargo-cult interface MTU 1440; keep recommendation-before-apply for all persistent changes.
- Extended `references/blog-method.md` candidate decisions for live fq persistence, initcwnd, endpoint extras, Realm/conntrack modules, and conflict-aware apply/read-back.
- Clarified that this skill must not silently wrap `curl | bash` or auto-run menu-66 chains (DNS purify, permanent IPv6 disable, Realm rewrite) without per-step evidence and approval.

### Notes

- Reusable ideas from Eric's script are absorbed as **candidates with evidence gates**, not as universal defaults.
- Do not infer BBRv3 from `uname -r` alone; sysctl name remains `bbr` across variants.

## 2026-07-12

### Added

- Added a commit-pinned review of `Madhatter2099/TCP-Optimize` covering reusable workflow ideas, parameter caveats, static implementation findings, and evidence gates.
- Added read-only collection guidance for dual-stack routing, conntrack pressure, RX queue topology, IRQ distribution, RPS/RFS, and per-CPU softirq state.

### Changed

- Extended tuning policy for IPv4 address selection, conntrack sizing, RPS/RFS, MSS clamping, service limits, live qdisc verification, and third-party script ownership.
- Clarified that `udp_mem` uses memory pages, RAM percentages are not a buffer-sizing formula, and BBRv3 must not be inferred from kernel version alone.
- Expanded the Chinese README with a concise comparison and link to the detailed review.

## 2026-07-09

### Added

- Added `peer_lifecycle` to the required tuning context so agents distinguish long-term/renewing peers from temporary or soon-to-expire VPSs.
- Added guidance to base persistent tuning decisions on durable peers that match the real traffic path.
- Added safe temporary `iperf3` server lifecycle guidance using `iperf3 -D --pidfile` instead of `pkill -f`.
- Added quoted-heredoc guidance for writing Markdown tuning profiles that contain backticks or rollback commands.
- Added README practical notes from the `dmit-lax` tuning run: HY2 layering, durable-peer priority, safe iperf cleanup, and evidence requirements before HTB/TBF/qos-agent.

### Changed

- Updated the workflow to explicitly choose peers before testing and to record durable peers in the final profile/report.
- Updated interpretation rules: weak or temporary peers should not downsize or reshape a long-term host when durable peers are clean.
- Clarified that high retransmission without local egress qdisc drop/backlog is not enough to justify local shaping.

### Fixed

- Documented a recurring pitfall where `pkill -f` can match the current SSH shell script and terminate the session before `iperf3` setup completes.
- Documented a shell heredoc pitfall where unquoted Markdown code fences can trigger command substitution while writing profile files.

## 2026-07-08

### Added

- Initial public skill release for evidence-based VPS TCP/network tuning.
- Added Chinese README covering purpose, source attribution, installation for Codex and Claude Code, active-questioning fields, recommendation-before-application gate, workflow, and operational cautions.
