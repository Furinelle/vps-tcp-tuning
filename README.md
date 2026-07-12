# vps-tcp-tuning

一个给 Codex / Claude Code 复用的 VPS TCP / 网络调优 Skill，用于让 agent 按证据检查、测试、推荐并在用户确认后应用 Linux VPS 网络配置。

本 Skill 基于 Lide / iBytebox 的文章整理而来：

- 原文：[让 AI 帮你调 VPS 网络：中转机和落地机 TCP 调优笔记](https://blog.ibytebox.com/posts/ai-agent-vps-tcp-tuning/)
- 作者：Lide / iBytebox
- 原文标注许可：CC BY-NC-SA 4.0

> 这个仓库不是“一键复制 sysctl 参数”的清单。它更像是一套给 AI 运维 agent 使用的工作流：先问清楚链路，再检查，再测试，再给出推荐配置，最后由用户决定是否应用。

## 这个 Skill 解决什么问题

很多 VPS 网络调优容易变成照抄参数，例如：

- 直接把 MTU 改成 `1440`
- 直接套 `TBF=1000Mbit`
- 直接把 TCP buffer 拉到 `256MB`
- 看到重传就马上限速
- 不区分中转机、落地机、建站机、出口机
- 不区分 TCP 协议和 HY2 / TUIC / QUIC 这类 UDP 协议

这个 Skill 强制 agent 按证据判断：

- 当前机器是什么角色：中转、落地、出口、建站、混合节点
- 真实业务链路怎么走
- 用户关键方向是哪一段
- 当前 sysctl / qdisc / MTU / 服务状态是什么
- PMTU 是否真的有问题
- iperf3 的重传是否和本机 qdisc drop / backlog 对得上
- 是否真的需要 BBR、fq、buffer、MTU、HTB/TBF 或 qos-agent
- 是否真的需要 IPv4 优先、conntrack 扩容、RPS/RFS、MSS Clamp 或全局文件句柄扩容

## 核心原则

- 用户说 `tcp调优`、`进行TCP调优`、`VPS网络调优` 时，应自动使用这个 Skill。
- agent 必须主动询问缺失信息，不能只看主机名猜业务链路。
- 默认只做检查、测试和推荐，不默认写持久配置。
- 所有持久配置变更前，必须先给出推荐配置。
- 是否应用推荐配置，由用户决定。
- 不把 `MTU 1440`、`TBF 1000Mbit`、`256MB buffer` 当万能答案。
- 测试 peer 必须逐个测试，不默认并发压测。
- 必须保留备份、验证步骤和回滚路径。

## 安装到 Codex

```bash
git clone https://github.com/Furinelle/vps-tcp-tuning.git ~/.agents/skills/vps-tcp-tuning
```

如果已经安装过，可以更新：

```bash
cd ~/.agents/skills/vps-tcp-tuning
git pull
```

## 安装到 Claude Code

```bash
git clone https://github.com/Furinelle/vps-tcp-tuning.git ~/.claude/skills/vps-tcp-tuning
```

如果已经安装过，可以更新：

```bash
cd ~/.claude/skills/vps-tcp-tuning
git pull
```

## 触发方式

显式调用：

```text
Use $vps-tcp-tuning to inspect and tune my relay or landing VPS networking safely.
```

中文直接说也应触发：

```text
帮我对这台 VPS 进行 TCP 调优。
```

```text
参考这个链路，帮我做 VPS 网络调优。
```

```text
检查一下这台中转机的 tcp调优 有没有问题。
```

## agent 会先问什么

运行这个 Skill 时，agent 应主动询问缺失的关键字段：

- `target_ssh`：目标机器的 SSH alias 或 SSH 命令
- `machine_role`：中转、落地、出口、建站、混合节点等
- `traffic_path`：例如 `用户 -> 中转 -> 落地 -> internet`
- `critical_direction`：用户真正关心的方向，例如下载、上传、视频秒开
- `proxy_software`：sing-box、xray、realm、gost、Hysteria2、TUIC、nginx、caddy、nftables 等
- `proxy_protocols`：SS2022、VLESS REALITY、HY2、TUIC、WireGuard、direct web 等
- `service_ports`：代理、Web、中转、iperf3 等相关端口
- `advertised_bandwidth`：商家标称带宽或端口速度
- `test_peers`：测试 peer 的标签、地址、iperf3 端口、是否允许 ping、是否能 SSH
- `peer_lifecycle`：哪些 peer 会长期续费/生产使用，哪些只是临时或即将弃用；持久参数应以长期 peer 为依据
- `permission_boundary`：只检查、允许测试、只给计划、允许应用、是否允许重启、是否允许改 MTU/限速/qos-agent

## 推荐配置再应用

这个 Skill 明确要求：**先给推荐配置，再由用户决定是否应用。**

推荐配置至少应包含：

- 证据摘要：角色、关键链路、PMTU、iperf/counter delta、瓶颈判断
- 精确候选配置：拟写入的 `/etc/sysctl.d/*.conf` 内容
- 可能的 qdisc / systemd / MTU / qos-agent 命令或 unit
- 不建议修改的项目和理由
- 风险说明：是否需要重启、是否会中断服务
- 验证计划：应用后如何读回和复测
- 回滚方案：备份路径和恢复命令

如果用户没有明确说“应用这个推荐配置”“按推荐应用”“直接应用”，agent 不应写入持久网络配置。

## 典型工作流

1. 主动询问缺失上下文。
2. 只读检查主机：OS、kernel、CPU、内存、接口、MTU、路由、socket、sysctl、qdisc、服务进程。
3. 读取已有 `/etc/sysctl.conf`、`/etc/sysctl.d/*.conf` 和 `*.profile.md`。
4. 识别长期 peer 和临时/即将弃用 peer；持久调优参数以长期 peer 为依据。
5. 逐个 peer 做 PMTU、ping、iperf3 P1/P4 正向/反向测试。
6. 记录测试窗口内 qdisc drop/backlog 和 TCP retransmission delta。
7. 按中转/落地/出口/Web 角色解释结果。
8. 给出推荐配置和不改项。
9. 等用户确认。
10. 应用前备份，应用后验证 SSH 和关键服务。
11. 写 profile 和最终报告。

## 实战经验更新

2026-07-09 对 `dmit-lax` 做 TCP/UDP 调优后，补充了几条更强约束：

- **长期 peer 优先**：如果一些 VPS 即将不续费，只把它们当作观察样本，不要让它们决定长期主机的持久参数。
- **HY2 判断要分层**：HY2 外层是 UDP/QUIC，不直接吃 TCP buffer；但 VPS 作为出口访问网站时，出口 TCP 仍受 BBR、TCP buffer、PMTU、qdisc 影响。
- **不要用 `pkill -f` 清 iperf3**：远端 SSH 脚本里包含同样命令文本时，`pkill -f` 可能杀掉当前 shell。应使用 `iperf3 -D --pidfile`，清理时只 kill pidfile 里的 `iperf3` PID。
- **profile heredoc 要 quoted**：写 Markdown profile 时如果包含反引号代码块，必须使用 `<<'EOF'` 这类 quoted heredoc，避免 shell 命令替换误执行回滚命令。
- **限速要有本机出口证据**：高重传但本机 egress qdisc drop/backlog 为 0 时，不应默认落盘 HTB/TBF/qos-agent。

## 对 TCP-Optimize 的参考

Skill 也审阅了 [`Madhatter2099/TCP-Optimize`](https://github.com/Madhatter2099/TCP-Optimize) 的模块化设计。吸收的部分包括：按功能拆分、展示 live 状态、先检查 BBR 支持、加载 conntrack 模块、独立评估 RPS/RFS、明确清理项。

其中的具体参数不会直接作为通用推荐：

- TCP buffer 不按“总内存 5%”直接定值，仍按 BDP、并发和内存压力计算。
- `udp_mem` 的单位是内存页，不按普通字节 buffer 解读。
- IPv4 优先只在真实 IPv6 路径较差时推荐；`gai.conf` 改的是地址选择顺序，不是 DNS 响应。
- RPS/RFS 先看 RX 队列、IRQ、每 CPU softirq 和 softnet drop；RSS 已充分分流时可能没有收益。
- conntrack、MSS Clamp、IP forwarding、百万级文件句柄只对有对应角色和压力证据的机器推荐。
- 不通过内核版本号直接判断 BBRv3，需核对内核包或补丁来源。
- 回滚恢复调优前的真实状态，不把 `cubic + fq_codel` 当所有发行版和主机的默认值。

完整审阅见 [`references/tcp-optimize-review.md`](references/tcp-optimize-review.md)。

## 仓库结构

```text
.
├── SKILL.md                    # Skill 主说明
├── references/
│   ├── blog-method.md           # 从原文整理出的详细方法和命令模式
│   └── tcp-optimize-review.md   # 对 TCP-Optimize 的证据化参考与边界
├── agents/
│   └── openai.yaml              # Codex UI metadata
├── README.md                    # 中文说明
└── LICENSE                      # 来源和许可说明
```

## 注意事项

- 不要把私钥、云厂商 token、代理密码直接发给 agent。
- 如果是生产节点，建议先只允许检查和测试。
- HY2 / TUIC / QUIC 不吃 Linux TCP buffer，但仍会受 MTU、qdisc、CPU 调度和出口 shaping 影响。
- qdisc drop/backlog 为 0 但 iperf3 高重传时，不要急着全局限速；更可能是路径、上游或对端问题。
- 一个弱 peer 的结果不能直接推导为全局配置。

## 许可与署名

本仓库内容是对原文方法的 Skill 化整理，并保留原文链接和作者信息。原文标注为 CC BY-NC-SA 4.0，请在使用和二次分发时遵守相应要求。
