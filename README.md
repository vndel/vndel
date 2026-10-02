<div align="center">

<img src="https://cdn.discordapp.com/avatars/398520151236149249/eae453267f7cca2cc6395d6b01e5a940.png?size=128" width="104" height="104" style="border-radius:50%" alt="" />

### Mohammed

**Systems & Game Server Developer** · Syria

Minecraft server architecture — Paper/Spigot internals, tick-budget engineering,
and proxy networks that stay up under load.

<a href="https://team.fivemcodehub.com">
  <img src="https://img.shields.io/badge/team.fivemcodehub.com-8b5cf6?style=for-the-badge&logoColor=white" height="28" alt="Portfolio" />
</a>
<a href="https://discord.com/users/398520151236149249">
  <img src="https://img.shields.io/badge/777m-5865F2?style=for-the-badge&logo=discord&logoColor=white" height="28" alt="Discord" />
</a>
<a href="mailto:777m@vndel.com">
  <img src="https://img.shields.io/badge/777m@vndel.com-D14836?style=for-the-badge&logo=gmail&logoColor=white" height="28" alt="Email" />
</a>

</div>

---

## Architecture philosophy

Every decision below traces back to one number: **a Minecraft tick has a 50ms
budget.** Exceed it and TPS drops; players feel it immediately.

#### Keep the main thread for game logic only

The main thread ticks the world. Anything else on it is stolen budget. SQL
queries, HTTP calls and file I/O belong on a worker, with results handed back
through an immutable snapshot.

In practice: an economy lookup inside a command handler costs 15-40ms per call
at scale. Held in memory with batched write-behind persistence, it costs zero.

#### Allocation is a latency problem, not a memory problem

The JVM reclaims garbage cheaply; it pauses to do it. A per-tick allocation in a
hot loop becomes a GC pause, and a GC pause longer than 50ms is a dropped tick.

So: reuse buffers, iterate without creating collections, and pin `Xms` to `Xmx`
so the collector is not re-tuning region sizes on a live server.

#### Adapt to load instead of capping it

Static limits punish players during quiet hours and still fail during peak.
Measure MSPT and throttle proportionally — when the server is healthy, a
well-written optimiser should do nothing at all.

#### Separate what players see from what the server simulates

`view-distance` and `simulation-distance` became separate settings in 1.18 for a
reason. Entity AI, redstone and block ticks scale with *simulation* distance;
perception scales with *view* distance. Dropping simulation from 10 to 6 cuts
simulated volume ~60% and is almost never noticed.

#### Scale networks on a broker, not on player connections

Plugin messaging tunnels through a player connection: it cannot reach an empty
backend and dies when the last player leaves. A Redis-backed bus lets any node
publish at any time — with the honest caveat that Pub/Sub is not durable, so
nothing that must survive a restart goes over it.

---

## Core repositories

**Plugins** — Paper 1.20+, Java 21

| Repository | Architecture |
|---|---|
| [hyper-economy-bridge](https://github.com/vndel/hyper-economy-bridge) | In-memory ledger with batched write-behind flush. `long` minor units (never `double`), atomic `map.compute()` mutation, Redis for invalidation only. Zero main-thread I/O on reads. |
| [sentinel-guard-anticheat-heuristics](https://github.com/vndel/sentinel-guard-anticheat-heuristics) | Movement and combat telemetry with latency + MSPT compensation. Collects evidence and decays violations; deliberately does not punish. |
| [aether-dynamic-spawners](https://github.com/vndel/aether-dynamic-spawners) | Closed-loop spawner throttling on a rolling MSPT window. Loaded chunks only, bounded work per pass, vanilla timings restored on disable. |
| [nexus-proxy-sync](https://github.com/vndel/nexus-proxy-sync) | Redis Pub/Sub bus for Velocity backends. Self-message filtering, stale-message rejection, exponential reconnect backoff. |
| [void-drop-cleaner](https://github.com/vndel/void-drop-cleaner) | Merges ground stacks by full metadata before removing anything. 4,200 drops → 71 entities with zero items destroyed. |
| [celestial-combat-tag](https://github.com/vndel/celestial-combat-tag) | Combat logging treated as a design problem: a disconnect cannot be refused, so the penalty applies the outcome instead. |
| [chronos-daily-rewards](https://github.com/vndel/chronos-daily-rewards) | Day boundaries resolved via `ZoneId`, not `millis / 86_400_000`. DST-safe, offline UUID lookup. |

**Infrastructure & tooling**

| Repository | Architecture |
|---|---|
| [paper-purpur-optimized-stack](https://github.com/vndel/paper-purpur-optimized-stack) | Config set for 100+ players at 20 TPS. Only non-default keys, each with its reasoning stated inline. |
| [aikars-flags-tuner](https://github.com/vndel/aikars-flags-tuner) | Computes heap headroom and selects G1 vs generational ZGC by size. Warns at the compressed-oops boundary. Hardened systemd units with RCON-based graceful stop. |
| [spigot-metric-exporter](https://github.com/vndel/spigot-metric-exporter) | Prometheus exporter, ~20KB, no client library. Main thread samples into an immutable record; the HTTP thread never touches the Bukkit API. |
| [mc-backup-daemon](https://github.com/vndel/mc-backup-daemon) | `save-off` → flush → archive → `save-on`, with `save-on` in a `finally` block. Restore verified byte-exact by checksum. Zero dependencies. |
| [velocity-edge-network-template](https://github.com/vndel/velocity-edge-network-template) | Modern forwarding, Geyser/Floodgate bridging, and the nftables policy that keeps backend ports unreachable. |
| [luckperms-matrix-preset](https://github.com/vndel/luckperms-matrix-preset) | Donor and staff as orthogonal tracks, banded weights, per-server contextual nodes. |
| [prison-rankup-eco-preset](https://github.com/vndel/prison-rankup-eco-preset) | Cost curve matched to the income curve so time-per-rank stays constant A→Z. No compounding multipliers. |
| [skyblock-island-schematics-pack](https://github.com/vndel/skyblock-island-schematics-pack) | Grid spacing, deny-by-default WorldGuard policy, async sliced island generation. |
| [mc-portfolio-provisioner](https://github.com/vndel/mc-portfolio-provisioner) | The provisioning script behind these repositories. Public-only by construction, aborts rather than touch a private repo, token never written to `.git/config`. |

#### Verifying any of it

No CI badges here, and that is intentional: GitHub-hosted runners are
unavailable on this account — a workflow whose only job was `echo hello` still
failed to start. A permanently red ✗ would imply broken code, so each project
carries a `verify.sh` that builds, tests, and asserts instead.

```bash
git clone https://github.com/vndel/hyper-economy-bridge && cd hyper-economy-bridge
./verify.sh      # mvn verify, then assert the jar is Java 21 bytecode
```

`hyper-economy-bridge` 15 tests · `spigot-metric-exporter` 8 · `chronos-daily-rewards` 7
· `mc-backup-daemon` 15 plus a restore proven byte-exact by `diff -r`.

---

## Stack

<div align="center">

<img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/java/java-original.svg" height="36" alt="Java" />&nbsp;&nbsp;
<img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/kotlin/kotlin-original.svg" height="36" alt="Kotlin" />&nbsp;&nbsp;
<img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/gradle/gradle-original.svg" height="36" alt="Gradle" />&nbsp;&nbsp;
<img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/maven/maven-original.svg" height="36" alt="Maven" />&nbsp;&nbsp;
<img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/redis/redis-original.svg" height="36" alt="Redis" />&nbsp;&nbsp;
<img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/sqlite/sqlite-original.svg" height="36" alt="SQLite" />&nbsp;&nbsp;
<img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/mysql/mysql-original.svg" height="36" alt="MySQL" />&nbsp;&nbsp;
<img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/docker/docker-original.svg" height="36" alt="Docker" />&nbsp;&nbsp;
<img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/linux/linux-original.svg" height="36" alt="Linux" />&nbsp;&nbsp;
<img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/bash/bash-original.svg" height="36" alt="Bash" />&nbsp;&nbsp;
<img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/prometheus/prometheus-original.svg" height="36" alt="Prometheus" />&nbsp;&nbsp;
<img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/nodejs/nodejs-original.svg" height="36" alt="Node.js" />&nbsp;&nbsp;
<img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/typescript/typescript-original.svg" height="36" alt="TypeScript" />

</div>

| Layer | Tools |
|---|---|
| Server platforms | Paper · Purpur · Spigot · Velocity · Geyser/Floodgate |
| Languages | Java 17/21 · Kotlin · Lua · TypeScript · Bash · Python |
| Build | Maven · Gradle (Kotlin DSL) · maven-shade with relocation |
| Data | MySQL/MariaDB · SQLite (WAL) · Redis · HikariCP |
| Ops | Linux · systemd · Docker · nftables · Prometheus · RCON |
| JVM | G1GC · Generational ZGC · heap sizing · GC log analysis |

---

<div align="center">

<img src="https://github-readme-stats.vercel.app/api?username=vndel&show_icons=true&include_all_commits=true&count_private=true&theme=dracula&hide_border=true&bg_color=0a0a12&title_color=8b5cf6&icon_color=22d3ee&text_color=a9a9bd" height="160" alt="" />
<img src="https://github-readme-stats.vercel.app/api/top-langs?username=vndel&layout=compact&langs_count=6&theme=dracula&hide_border=true&bg_color=0a0a12&title_color=8b5cf6&text_color=a9a9bd" height="160" alt="" />

<sub>Part of the team at <a href="https://team.fivemcodehub.com">team.fivemcodehub.com</a> · Discord <code>777m</code> · <code>398520151236149249</code></sub>

</div>
