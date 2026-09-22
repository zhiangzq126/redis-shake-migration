**English** | [简体中文](readme-zh.md)

# RedisShake Migration

An Agent skill for planning and managing RedisShake migration tasks.
It turns Excel tables, text descriptions, or guided answers into `shake.toml`,
then guides deployment, start/stop, monitoring, and troubleshooting on Linux.
This standalone repository contains skill instructions, references, and helper scripts—not the RedisShake binary.

## Scope and limits

- Supports Redis-to-Redis migration, RDB import, and AOF replay.
- Covers full migration and incremental synchronization, subject to source permissions and reader capabilities.
- Supports execution directly on the Linux migration host or remotely over SSH.
- Does not handle MongoDB/MySQL/Elasticsearch migration, Redis memory-dump backup creation, or cluster topology changes.
- Remote operations require user-provided SSH access; browser-only Agents are not supported.
- Does not validate post-migration consistency or guarantee zero downtime or lossless cutover.
  Plan separate verification, such as `redis-full-check`, and DBA review before production use.

## Get started with an Agent

```bash
git clone https://github.com/zhiangzq126/redis-shake-migration.git
cd redis-shake-migration
```

1. Open this repository in an Agent platform with Bash/Shell tools, such as Qoder, Claude Code, or Cursor.
2. Ask the Agent to read [SKILL.md](SKILL.md) and the relevant references before acting.
   For native skill discovery, follow your platform's skill-loading instructions;
   the skill identifier is `alibabacloud-migration-dbm-redis-shake-migration`.
3. State whether you need configuration only, deployment, task control, monitoring, or troubleshooting.
   Stages can run independently; an existing configuration does not require restarting information collection.
4. Review the proposed configuration and execution plan before authorizing any changes.

Example requests (use placeholders, not real credentials, in shared prompts):

> Read SKILL.md. Help me plan a one-time migration from managed Redis to a target Redis.
> List missing information and draft a masked configuration only. Do not connect, write files, or start anything.

> Read SKILL.md. I have a shake.toml on a Linux migration host.
> Explain the deployment checks and ask me to confirm the host, SSH access, paths, and operation before proceeding.

Cloning or reading the skill does not install RedisShake or authorize execution.
This repository is the skill root; `/opt/redis-shake` is a separate default runtime directory.

## Prerequisites: local versus remote

| Location | Requirements |
| --- | --- |
| Agent machine | Bash 4+, `tar`, and `sha256sum`; OpenSSH client tools (`ssh`, `scp`, `ssh-copy-id`) for remote workflows. |
| Linux migration host | Linux x86_64, glibc >= 2.17, Bash 4+, an executable RedisShake binary, and access to the required input files or Redis endpoints. |
| Remote access | User-specified host, SSH port, username, authentication method, and permissions for the runtime directory. |
| Optional production mode | systemd on the migration host; sudo authorization for service installation and management. |
| Optional checks | `redis-cli` where sampling is performed; Python >= 3.11 with `tomllib` where TOML validation is performed. |
| Optional credential templating | `envsubst` where a credential template is rendered; the resulting TOML still needs protection. |

For local execution, the Agent's shell is already on the Linux migration host;
that machine must satisfy both sets of requirements. A Windows clone is not a Windows migration runtime.
For remote execution, the Agent machine needs SSH reachability, while the migration host needs Redis reachability.
Obtain the RedisShake binary manually from its official release source, verify its published SHA256,
and deploy it before starting a task. This repository supplies no installer or binary.

## Choose and review the configuration

Provide source/target addresses, authentication requirements, desired migration mode,
cluster/TLS settings, DB/key filters, conflict policy, rate limits, and any target-clearing request.
For file readers, provide the absolute input path on the migration host.
Review the templates against the RedisShake version actually deployed.

| Reader | Intended use and key requirement |
| --- | --- |
| `sync_reader` | Replication-capable Redis; requires PSYNC/REPLCONF permissions. Supports full and incremental stages. |
| `scan_reader` | Sources without replication access, including the managed-cloud cases covered by this skill; requires SCAN/DUMP/TTL/TYPE. |
| `scan_reader` with `ksn=true` | Notification-based incremental listening; source keyspace notifications must be enabled. Events may be missed; this is not equivalent to replication. |
| `rdb_reader` | Import an existing RDB file; full import only. |
| `aof_reader` | Replay an existing AOF file. |

Use [reader modes](references/reader-modes.md) for selection details and
[configuration templates](references/templates.md) for TOML sections and placeholders.
Do not treat template overwrite policies or throughput settings as production-safe defaults.

## Deployment and safety

- Default runtime layout: binary `/opt/redis-shake/redis-shake`, configuration `/opt/redis-shake/shake.toml`, logs/PID under `/opt/redis-shake/data/`.
- For remote work, confirm the destination before transferring the configuration or scripts. See [SSH guidance](references/remote-ssh.md).
- Before SSH/Redis connections, uploads, file writes, permission changes, sudo, or process control, explain the exact target, purpose, paths, and privilege level and obtain explicit confirmation.
  Follow the skill's `YES` confirmation flow; target-clearing requires two explicit confirmations because it is irreversible.
- Use least privilege. Authorize only user-specified endpoints; distinguish intended transfers to the migration host from disclosure to third parties.
- Treat `shake.toml` as a secret-bearing file: use mode `600`, mask displayed passwords, and keep credentials out of shell arguments.
  Never commit real configurations, private keys, or sensitive logs, or include them in public issues/screenshots. Arrange Git exclusions before generating real configuration files.
- Protect SSH private keys with mode `600` or `400`; use approved credential channels rather than shared prompts.
- The confirmation hook is a keyword-based helper, not a complete security boundary.
  This checkout contains no platform hook registration; verify integration separately and retain explicit Agent/user confirmation.
- Review scripts before use. Their default runtime directory is the parent of the script directory, not inherently `/opt/redis-shake`; confirm deployment paths first.
- `stop.sh` falls back to `pkill -f "redis-shake shake.toml"` when its PID file is missing; this can affect other matching tasks. Confirm process scope, especially on shared hosts.
- Check logs, progress, and data samples after running. Catch-up metrics and a successful process exit are not proof of data consistency.

## Repository navigation

| File | Purpose |
| --- | --- |
| [SKILL.md](SKILL.md) | Agent workflow, permissions, authorization, deployment, monitoring, and troubleshooting. |
| [references/reader-modes.md](references/reader-modes.md) | Reader decision tree, requirements, and limitations. |
| [references/templates.md](references/templates.md) | Reader/writer, filtering, and advanced configuration templates. |
| [references/remote-ssh.md](references/remote-ssh.md) | Remote access, transfers, systemd, and metrics forwarding guidance. |
| [scripts/start.sh](scripts/start.sh) | Check binary/configuration and existing PID, then launch the task. |
| [scripts/status.sh](scripts/status.sh) | Report PID-based process status, uptime, and recent logs. |
| [scripts/stop.sh](scripts/stop.sh) | Stop via PID, with the process-matching fallback described above. |
| [scripts/confirm-write.sh](scripts/confirm-write.sh) | PreToolUse-style confirmation helper for matching command patterns. |

## Origin

Derived from the corresponding [RedisShake migration skill in aliyun/alibabacloud-aiops-skills](https://github.com/aliyun/alibabacloud-aiops-skills/tree/master/skills/playbooks/wadaps/alibabacloud-migration-dbm-redis-shake-migration).
This attribution does not establish a new license for this standalone repository; review applicable upstream terms before reuse or redistribution.
