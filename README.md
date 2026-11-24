# splunk-to-dql

An agent skill for converting Splunk SPL searches into Dynatrace Query Language (DQL).

You give your coding agent an SPL query. It returns the DQL, lists every place where the result can differ from Splunk, and gives you a small test query so you can check the logic before running it on real data. When a field extraction runs often enough that it belongs at ingest, it also tells you how to set it up in OpenPipeline.

I built this while preparing for a Splunk to Dynatrace migration. Most conversions don't fail because of syntax. They fail on things like `OR` precedence, case sensitivity, null groups in `stats`, and which record `dedup` keeps. A literal translation of those runs fine and returns different numbers. The skill is mostly a list of those traps.

> **Status:** Every DQL function, command, and OpenPipeline setting used here was checked against the Dynatrace documentation. The test queries in `references/examples.md` and the processor examples in `references/openpipeline-examples.md` have not all been run in a Dynatrace environment yet. Until they are, treat the expected results as worked out by hand. If you run one and get something different, please [open an issue](../../issues/new/choose).

## What's in the repo

```
SKILL.md                       instructions the agent loads
references/
  command-mapping.md           SPL commands to DQL
  function-mapping.md          eval and stats functions to DQL
  examples.md                  11 worked conversions with test queries
  dql-patterns.md              DQL syntax reminders and performance
  validation.md                how to compare results with Splunk
  limitations.md               what has no direct DQL equivalent
  visualization.md             Dashboard chart setup (side-by-side bars)
  openpipeline.md              OpenPipeline for logs: when to use it, setup, limits
  splunk-ingest-to-openpipeline.md  props.conf and transforms.conf to OpenPipeline
  openpipeline-examples.md     6 processor examples to test with Run sample data
  environment-discovery.md     queries the agent runs when connected to Dynatrace
```

The agent reads `SKILL.md` when a task matches and opens the reference files only when it needs them.

## Install

The skill follows the [Agent Skills](https://agentskills.io/specification) format. The folder must be named `splunk-to-dql` to match the `name` in `SKILL.md`.

**Claude Code**

```sh
git clone https://github.com/mohsinzaheer25/splunk-to-dql-skill.git ~/.claude/skills/splunk-to-dql
```

For a single project, clone into `.claude/skills/splunk-to-dql` inside that project instead.

**Codex**

```sh
git clone https://github.com/mohsinzaheer25/splunk-to-dql-skill.git ~/.agents/skills/splunk-to-dql
```

For a single repository, use `.agents/skills/splunk-to-dql` at the repository root.

**Pi**

```sh
git clone https://github.com/mohsinzaheer25/splunk-to-dql-skill.git ~/.pi/agent/skills/splunk-to-dql
```

Pi also reads `~/.agents/skills/`. Run `/reload` in an open session, or call the skill directly with `/skill:splunk-to-dql`.

**Cursor**

```sh
git clone https://github.com/mohsinzaheer25/splunk-to-dql-skill.git ~/.cursor/skills/splunk-to-dql
```

For a single project, clone into `.cursor/skills/splunk-to-dql` inside that project instead. Cursor also reads `.agents/skills/` and `.claude/skills/`, so a copy you installed for another tool works too.

To check that Cursor found it, open **Customize** in the sidebar and go to **Skills**. Cursor applies the skill when a request matches. To call it directly, type `/splunk-to-dql` in Agent chat.

Skills in `~/.cursor/skills/` only run on your machine. To use the skill with Cursor Cloud Agents, turn on **Sync Skills for Cloud Agents**, or install it in the project folder.

**Other tools**

If your tool supports Agent Skills, put the folder where it looks for skills. If it doesn't, you can paste `SKILL.md` into the tool's custom instructions and attach the files in `references/`.

To update, run `git pull` inside the skill folder.

## Using it

Ask in plain language and paste the SPL. The more you say about your data, the fewer placeholders you get back.

```
Convert this Splunk search to DQL:

index=web sourcetype=access_combined status>=400
| stats count by status
| sort -count
```

```
Convert this to DQL. In our Dynatrace setup the web logs have
log.source == "nginx" and status is already a numeric field.

index=web sourcetype=access_combined
| eval is_error = if(status >= 500, 1, 0)
| stats sum(is_error) as errors, count as total by host
| eval error_rate = round(errors / total * 100, 2)
| where error_rate > 5
| sort -error_rate
```

```
This alert uses transaction. Convert it, tell me what can't be done
the same way in DQL, and how I should check the numbers against Splunk.

index=web sourcetype=access_combined
| transaction session_id maxspan=30m
| where duration > 600
| stats count as long_sessions
```

What comes back:

1. the DQL, with `<placeholders>` for anything specific to your environment
2. the places where the result can differ from Splunk, and why
3. the assumptions you need to confirm
4. how to check it, usually including a `data record(...)` query you can run without any log data
5. when it helps, an OpenPipeline setup: the processors and route, plus the shorter query to use once the fields are extracted at ingest

You can also ask about ingest config directly, for example "Here's our props.conf for this sourcetype, how do I do this in OpenPipeline?"

## Optional: connect Dynatrace

The skill works without access to your environment. It reads what it can from the SPL, asks a few questions, and gives you short queries to run when a fact matters.

If your agent can run DQL against your environment, for example through the [Dynatrace MCP server](https://docs.dynatrace.com/docs/dynatrace-intelligence/dynatrace-mcp), the skill looks things up itself: field names, where a Splunk index ended up, log volume, which pipeline handles the logs. It also runs the converted query and the test queries, so it can report results instead of expected results.

When connected, the skill:

- only reads. It never changes settings, pipelines, or buckets, and gives you the steps to make changes yourself.
- keeps queries small (short timeframes, a `limit` on log queries) and asks before scanning more than a day of logs. Grail queries are billed by data scanned.
- shows each query it ran.

Use a token with read permissions only. The Dynatrace MCP docs list the scopes, mainly `mcp-gateway:servers:invoke`, `mcp-gateway:servers:read`, and `storage:logs:read`, `storage:metrics:read`, `storage:buckets:read` for the data you want it to see. See [environment-discovery.md](references/environment-discovery.md) for the exact queries the skill uses.

## Before you trust a converted query

- Expand macros, eventtypes, and tags in Splunk first. The agent can't know what they contain.
- Run the SPL and the DQL on the same fixed time window and compare counts. [validation.md](references/validation.md) lists what to compare and in what order.
- Pay attention to anything the agent marks as "not 1:1". Those are real differences, not style choices.

## Contributing

Corrections and new examples are welcome. Every change needs a docs link and a test query, and all data must be made up. See [CONTRIBUTING.md](CONTRIBUTING.md).

## License and trademarks

MIT, see [LICENSE](LICENSE).

This is an independent project. It is not affiliated with, endorsed by, or supported by Splunk, Cisco, or Dynatrace. Splunk, SPL, Dynatrace, Grail, and DQL are trademarks of their respective owners. The names are used only to describe what the skill converts between.

The converted queries are suggestions. Check them against your own data before you use them in alerts or dashboards.

## References

- [DQL reference](https://docs.dynatrace.com/docs/platform/grail/dynatrace-query-language)
- [DQL best practices](https://docs.dynatrace.com/docs/platform/grail/dynatrace-query-language/dql-best-practices)
- [Dynatrace Pattern Language](https://docs.dynatrace.com/docs/platform/grail/dynatrace-pattern-language)
- [Splunk SPL Search Reference](https://help.splunk.com/en/splunk-enterprise/search/spl-search-reference/10.6/introduction/welcome-to-the-search-reference)
