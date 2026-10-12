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

...(original content continues unchanged)...