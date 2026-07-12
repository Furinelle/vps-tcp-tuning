# Changelog

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
