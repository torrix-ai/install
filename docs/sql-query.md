# SQL Query Interface

The SQL Query page lets you run read-only SQL queries directly against your Torrix database. Use it to answer questions that the built-in dashboards do not cover, build ad-hoc reports, or explore your run data in detail.

To open it, click **Query** in the left sidebar.

## Writing a query

Type any `SELECT` statement in the query editor. Press **Ctrl+Enter** (or **Cmd+Enter** on Mac) to run it, or click the **Run** button.

The schema reference panel on the left lists all available tables and their columns. Click any table name to expand it.

**Example queries:**

Total cost by model:
```sql
SELECT model, COUNT(*) AS runs, ROUND(SUM(cost_usd), 6) AS total_cost
FROM runs
GROUP BY model
ORDER BY total_cost DESC
```

Slowest runs in the last 7 days:
```sql
SELECT name, model, latency_ms, cost_usd
FROM runs
WHERE created_at >= datetime('now', '-7 days')
ORDER BY latency_ms DESC
LIMIT 20
```

Error rate by provider:
```sql
SELECT provider,
  COUNT(*) AS total,
  SUM(CASE WHEN status >= 400 THEN 1 ELSE 0 END) AS errors,
  ROUND(100.0 * SUM(CASE WHEN status >= 400 THEN 1 ELSE 0 END) / COUNT(*), 1) AS error_pct
FROM runs
GROUP BY provider
```

## Results

Results render as a table below the editor. The header row shows column names, and a badge shows the total row count and execution time.

Click **Export CSV** to download the result set as a CSV file.

If no rows are returned, the table shows "No rows returned."

## Security

The query interface only allows `SELECT` statements. The following are blocked and return an error:

- `DROP`, `DELETE`, `UPDATE`, `INSERT`, `ALTER`, `CREATE`, `ATTACH`, `DETACH`
- Stacked queries using `;`

Results are capped at 500 rows. To fetch more, add a `LIMIT` and `OFFSET` clause manually.

## Available tables

| Table | Description |
|---|---|
| `runs` | Every logged LLM request. Primary table for cost, latency, token, and model data. |
| `events` | Prompt, response, and thinking text attached to runs. |
| `projects` | Project namespaces. |
| `prompts` | Prompt Library entries. |
| `prompt_versions` | Versioned content for each prompt. |

To see all columns in a table, run:

```sql
PRAGMA table_info(runs);
```
