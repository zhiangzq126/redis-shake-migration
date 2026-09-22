[English](README.md) | **简体中文**

# RedisShake Migration

[![最近提交](https://img.shields.io/github/last-commit/zhiangzq126/redis-shake-migration/main)](https://github.com/zhiangzq126/redis-shake-migration/commits/main/)
[![GitHub 问题](https://img.shields.io/github/issues/zhiangzq126/redis-shake-migration)](https://github.com/zhiangzq126/redis-shake-migration/issues)

用于规划和管理 RedisShake 数据迁移任务的 Agent 技能。
可从 Excel 表格、文本描述或逐项问答中整理迁移信息，生成 `shake.toml`，
并指导 Linux 环境下的部署、启停、监控与故障排查。
这是独立的技能仓库，包含说明、参考资料和辅助脚本，不包含 RedisShake 二进制程序。

## 用途与边界

- 支持 Redis 到 Redis 的迁移、已有 RDB 文件导入和 AOF 文件回放。
- 覆盖全量迁移与增量同步，实际能力受源端权限和 Reader 模式限制。
- 可在 Linux 迁移主机上直接操作，也可通过 SSH 远程操作。
- 不适用于 MongoDB/MySQL/Elasticsearch 迁移、Redis 内存 dump 备份生成或集群拓扑改造。
- 远程操作必须由用户提供 SSH 访问条件；不支持仅能操作浏览器的 Agent。
- 不验证迁移后数据一致性，也不保证零停机或无损割接。
  应另行安排校验，例如使用 `redis-full-check`，生产执行前请 DBA 审核。

## 通过 Agent 上手

```bash
git clone https://github.com/zhiangzq126/redis-shake-migration.git
cd redis-shake-migration
```

1. 在具备 Bash/Shell 工具的 Agent 平台中打开本仓库，例如 Qoder、Claude Code 或 Cursor。
2. 请 Agent 先阅读 [SKILL.md](SKILL.md) 和当前任务所需的参考资料。
   若使用平台的原生技能发现机制，请遵循该平台的技能加载说明；
   技能标识为 `alibabacloud-migration-dbm-redis-shake-migration`。
3. 说明需要的是仅生成配置、部署、任务启停、监控还是排障。
   各阶段可独立使用；已有配置时不必重新开始信息收集。
4. 审核配置与执行计划，再授权具体操作。

自然语言请求示例（共享请求中使用占位符，不填写真实凭据）：

> 阅读 SKILL.md，协助规划一次从托管 Redis 到目标 Redis 的全量迁移。
> 先列出缺失信息并起草脱敏配置，不连接服务器、不写文件、不启动任务。

> 阅读 SKILL.md。我已在 Linux 迁移主机上准备好 shake.toml。
> 请说明部署前检查项，操作前向我确认主机、SSH 访问方式、路径和具体操作。

克隆仓库或阅读技能不等于安装 RedisShake，也不构成执行授权。
本仓库目录就是技能根目录；`/opt/redis-shake` 是另行准备的默认运行目录。

## 前置条件：本地与远端

| 所在位置 | 要求 |
| --- | --- |
| Agent 所在机器 | Bash 4+、`tar`、`sha256sum`；远程流程需要 OpenSSH 客户端工具（`ssh`、`scp`、`ssh-copy-id`）。 |
| Linux 迁移主机 | Linux x86_64、glibc >= 2.17、Bash 4+、可执行的 RedisShake 二进制，并能访问所需输入文件或 Redis 端点。 |
| 远程访问 | 用户指定的主机、SSH 端口、用户名、认证方式，以及运行目录所需权限。 |
| 可选生产模式 | 迁移主机具备 systemd；安装和管理服务需要 sudo 授权。 |
| 可选检查工具 | 在执行抽样检查的机器上准备 `redis-cli`；在校验 TOML 的机器上准备含 `tomllib` 的 Python >= 3.11。 |
| 可选凭据模板 | 在渲染凭据模板的机器上准备 `envsubst`；生成后的 TOML 仍需保护。 |

本地执行指 Agent 的 Shell 已位于 Linux 迁移主机上，
该机器须同时满足两侧条件。在 Windows 上克隆仓库不代表可以原生运行迁移任务。
远程执行时，Agent 机器须能通过 SSH 访问迁移主机，迁移主机须能连接相关 Redis。
请手动从 RedisShake 官方发布来源取得二进制，对照公布的 SHA256 校验完整性，
并在启动任务前部署完成。本仓库不提供安装器或二进制程序。

## 选择并审核配置

准备源端/目标端地址、认证要求、迁移模式、集群/TLS 设置、
DB/Key 过滤条件、冲突策略、限速要求，以及是否涉及清空目标库。
使用文件 Reader 时，提供输入文件在迁移主机上的绝对路径。
应结合实际部署的 RedisShake 版本审核模板。

| Reader | 用途与关键条件 |
| --- | --- |
| `sync_reader` | 具备复制能力的 Redis；需要 PSYNC/REPLCONF 权限，可处理全量与增量阶段。 |
| `scan_reader` | 无复制权限的源端，包括本技能涉及的托管云场景；需要 SCAN/DUMP/TTL/TYPE 权限。 |
| `scan_reader` 配合 `ksn=true` | 基于通知监听增量；须在源端启用键空间通知，可能遗漏事件，不等同于复制流同步。 |
| `rdb_reader` | 导入已有 RDB 文件，仅执行全量导入。 |
| `aof_reader` | 回放已有 AOF 文件。 |

模式选择详见 [Reader 说明](references/reader-modes.md)，
TOML 配置分区和占位符见 [配置模板](references/templates.md)。
不要把模板中的覆盖策略或吞吐参数直接视为生产安全默认值。

## 部署与安全原则

- 默认运行布局：二进制 `/opt/redis-shake/redis-shake`，配置 `/opt/redis-shake/shake.toml`，日志与 PID 位于 `/opt/redis-shake/data/`。
- 远程操作须先确认接收主机，再传输配置或脚本，详见 [SSH 参考](references/remote-ssh.md)。
- 在 SSH/Redis 连接、上传、文件写入、权限修改、sudo 或进程控制前，说明准确目标、目的、路径和权限级别，并取得明确确认。
  按技能要求使用 `YES` 确认流程；清空目标库不可逆，必须明确确认两次。
- 遵循最小权限，仅访问用户指定的端点；区分向迁移主机传输必要文件与向第三方披露信息。
- 将 `shake.toml` 视为含密文件：权限设为 `600`，展示时脱敏，不在 Shell 参数中传递密码。
  不提交真实配置、私钥或敏感日志，也不放入公开 Issue 或截图；生成真实配置前先安排 Git 忽略规则。
- SSH 私钥权限应为 `600` 或 `400`；通过认可的安全渠道提供凭据，不在共享请求中暴露。
- 确认 Hook 只是基于关键词的辅助检查，不是完整安全边界。
  当前副本不含平台 Hook 注册配置，需另行核实接入，不能省略 Agent 与用户的明确确认。
- 使用前审核脚本。脚本默认运行目录是脚本目录的上级，并非固定的 `/opt/redis-shake`；先确认部署路径。
- `stop.sh` 在 PID 文件缺失时会退回 `pkill -f "redis-shake shake.toml"`，可能影响其他匹配任务；共享主机上尤其需要确认进程范围。
- 运行后检查日志、进度和数据样本；追平指标或进程正常退出均不能证明数据一致。

## 配置与脚本导航

| 文件 | 内容 |
| --- | --- |
| [SKILL.md](SKILL.md) | Agent 工作流、权限、授权确认、部署、监控和排障。 |
| [references/reader-modes.md](references/reader-modes.md) | Reader 决策流程、权限要求和能力限制。 |
| [references/templates.md](references/templates.md) | Reader/Writer、过滤和高级配置模板。 |
| [references/remote-ssh.md](references/remote-ssh.md) | 远程访问、文件传输、systemd 与指标端口转发参考。 |
| [scripts/start.sh](scripts/start.sh) | 检查二进制、配置和已有 PID，然后启动任务。 |
| [scripts/status.sh](scripts/status.sh) | 根据 PID 查询进程状态、运行时长和最近日志。 |
| [scripts/stop.sh](scripts/stop.sh) | 根据 PID 停止任务，缺失时使用上述进程匹配兜底。 |
| [scripts/confirm-write.sh](scripts/confirm-write.sh) | 按命令模式触发确认的 PreToolUse 风格辅助 Hook。 |

## 来源

源自 [aliyun/alibabacloud-aiops-skills 中对应的 RedisShake 迁移技能](https://github.com/aliyun/alibabacloud-aiops-skills/tree/master/skills/playbooks/wadaps/alibabacloud-migration-dbm-redis-shake-migration)。
此来源说明不为独立仓库设定新的许可证；复用或再分发前请核实适用的上游条款。
