# Veto — SQL Safety & Cost Oracle (MCP)

> **Veto is a deterministic MCP server that vets Postgres SQL for safety and cost *before* an AI coding agent runs it.** It returns an `ok` / `warn` / `block` verdict on every statement — no LLM in the core, and it never connects to your database.

**Website:** https://vetosql.com · **MCP endpoint:** `https://vetosql.com/mcp` (remote, streamable-http)

AI coding agents (Claude Code, Cursor, …) write and execute SQL. Occasionally they write `DELETE FROM payments` with no `WHERE`, or a `DROP TABLE` during a migration. More prompting doesn't fix a probabilistic system — a deterministic gate does. Veto is that gate: given the same statement, it returns the same verdict, every time, with stable finding ids you can audit and gate CI on.

---

## What it catches

| Verdict | Meaning | What falls here |
|---|---|---|
| `block` | Destructive / data loss | Unscoped `DELETE`/`UPDATE`, `TRUNCATE`, dropping data-bearing objects — including destructive statements hidden inside CTEs |
| `warn` | Risky but recoverable | Lock-heavy schema changes, expensive scans, common anti-patterns like `SELECT *` |
| `ok` | Safe to run | Routine, reversible migrations |

Every finding carries a stable dotted id (e.g. `destructive.delete_without_where`) so your pipeline can branch on it. The exact rule set lives server-side and evolves over time.

## Why deterministic

- **Reproducible** — same input, same verdict. Testable, so trustable.
- **Auditable** — a *named rule* fired, not "the model felt it was risky."
- **No drift** — can't be talked out of a `block` by a clever prompt; doesn't get worse on a bad day.
- **Never touches your DB** — cost is measured with a real `EXPLAIN` on a throwaway scratch Postgres inside a transaction that is **always rolled back**. Your production database is never connected.

## Tools

### `analyze_sql`
Returns a deterministic safety + cost verdict for Postgres SQL / migrations.

| Input | Type | Notes |
|---|---|---|
| `sql` | string | The SQL / migration to analyze — one or more statements (required) |
| `schema` | string? | Optional `CREATE TABLE/INDEX` DDL — enables `EXPLAIN`-based cost analysis on scratch Postgres |
| `rowCountHints` | object? | Optional map of table name → estimated row count, for realistic cost estimates |

Returns `{ verdict, findings[], plan?, meta }` where `verdict` ∈ `ok | warn | block`.

### `set_policies` *(Pro)*
Stores custom org policies keyed to your Pro key; `analyze_sql` then enforces them on top of the built-in rules. Policies are **declarative data — validated and never executed** (max 50, replaces the previously stored set).

Each policy: `table` (exact name or glob, e.g. `payments`, `audit_*`, `*`), `operations` (any of `select`/`insert`/`update`/`delete`/`truncate`/`drop`/`alter`), `action` (`block`/`warn`), optional `message`.

```json
{
  "policies": [
    { "table": "payments", "operations": ["delete", "truncate"], "action": "block",
      "message": "Never delete from payments — use the refund flow." }
  ]
}
```

Sending the full set **updates** it; sending an empty array **clears** it.

### `get_policies` *(Pro)*
Returns the custom org policy set currently stored for your key — the same rules `analyze_sql` enforces on top of the built-ins. Read-only; returns an empty list if none are set.

---

## Setup

Veto is a **remote** MCP server — no install, no source needed. Point your client at the endpoint.

### Claude Code — `.mcp.json`
```json
{
  "mcpServers": {
    "veto": {
      "type": "http",
      "url": "https://vetosql.com/mcp"
    }
  }
}
```

### Cursor — `~/.cursor/mcp.json`
```json
{
  "mcpServers": {
    "veto": {
      "url": "https://vetosql.com/mcp"
    }
  }
}
```

The free tier needs no key (60 req/min). **Pro:** add your `VETO-…` key as a bearer token (keep the word `Bearer` and the space):
```json
"headers": { "Authorization": "Bearer VETO-…" }
```

---

## Pricing

| Tier | Price | Limits | Extras |
|---|---|---|---|
| **Free** | €0 | 60 req/min | Full deterministic verdict — all destructive, locking & cost rules |
| **Pro** | €9.90 / mo | 1200 req/min | Custom org policies (`set_policies`), maintainer support |

Subscribe at [vetosql.com](https://vetosql.com).

---

## FAQ

**What databases does Veto support?**
PostgreSQL. Works with any Postgres host (Supabase, Neon, RDS, self-hosted) and any migration tool, because Veto analyzes the SQL text — it doesn't connect to your database.

**Is it safe? Can it see or modify my data?**
No. Veto never connects to your production database. Cost analysis runs inside a transaction that is always rolled back, against a separate scratch Postgres — no data is read or written.

**How is Veto different from a linter like Squawk or sqlfluff?**
Those are CI/style tools. Veto is a real-time *runtime gate* an AI agent calls over MCP, returning an `ok`/`warn`/`block` verdict on the exact statement it's about to execute — plus cost estimation and custom org policies.

**Why not just give the agent a read-only or restricted DB role?**
Roles are coarse and easy to misconfigure, and they don't catch a costly sequential scan or a full-table `UPDATE` inside a write-allowed role. Veto adds a statement-level verdict on top of whatever roles you use.

**Does it use an LLM?**
No. The core is deterministic static analysis + `EXPLAIN`. The calling agent narrates the structured verdict; the verdict itself never comes from a model.

---

## Links

- **Website:** [vetosql.com](https://vetosql.com)
- **Blog:** [An AI agent wiped a production database. The fix isn't a better prompt.](https://vetosql.com/blog/ai-agent-deleted-production-database/)
- **MCP endpoint:** `https://vetosql.com/mcp`
- **Official MCP registry:** `com.vetosql/veto`
- **Glama:** [glama.ai/mcp/connectors/com.vetosql/veto](https://glama.ai/mcp/connectors/com.vetosql/veto)

---

Built by [Metawear](https://www.metawear.cz). The hosted service runs at [vetosql.com](https://vetosql.com); this repository is the public documentation and MCP registry manifest for the Veto server.
