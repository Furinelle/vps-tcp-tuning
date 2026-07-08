# vps-tcp-tuning

A reusable Codex / Claude Code skill for evidence-based Linux VPS TCP/network tuning, adapted from Lide's iBytebox article:

- Source: https://blog.ibytebox.com/posts/ai-agent-vps-tcp-tuning/
- Author: Lide / iBytebox
- Source license noted by the article: CC BY-NC-SA 4.0

## What this skill enforces

- Automatically use this skill when the user asks for `tcp调优`, `进行TCP调优`, or VPS network tuning.
- Ask for missing context before touching remote hosts: SSH aliases, host roles, traffic path, critical direction, protocols, peers, and permission boundary.
- Inspect and test first; do not cargo-cult `MTU 1440`, `TBF 1000Mbit`, or `256MB` buffers.
- Produce a recommended configuration first. The user decides whether to apply it.
- Do not apply persistent sysctl/qdisc/MTU/qos-agent/service changes until the user explicitly approves the recommendation.
- Write rollback/profile evidence for any applied persistent changes.

## Install for Codex

```bash
git clone https://github.com/Furinelle/vps-tcp-tuning.git ~/.agents/skills/vps-tcp-tuning
```

## Install for Claude Code

```bash
git clone https://github.com/Furinelle/vps-tcp-tuning.git ~/.claude/skills/vps-tcp-tuning
```

## Invocation example

```text
Use $vps-tcp-tuning to inspect and tune my relay or landing VPS networking safely.
```

Or in Chinese:

```text
帮我对这台 VPS 进行 TCP 调优。
```

## Files

- `SKILL.md` — main skill instructions.
- `references/blog-method.md` — concise method checklist and command patterns distilled from the source article.
- `agents/openai.yaml` — Codex UI metadata.
