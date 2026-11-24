# Changelog

Notable changes to this skill. Versions follow [Semantic Versioning](https://semver.org/). The version is also set in the `metadata.version` field of `SKILL.md`.

## Unreleased

- README: added install steps for Cursor.
- Added `references/visualization.md`: how to get side-by-side bars per time bucket in a Dashboard (Categorical bar chart, Vertical layout, Group mode grouped). Tested in a Dashboard.
- Added OpenPipeline support for logs: `references/openpipeline.md` (when to use it, setup, processors, limits), `references/splunk-ingest-to-openpipeline.md` (`props.conf` and `transforms.conf` mapping), and `references/openpipeline-examples.md` (6 processor examples). Checked against the Dynatrace docs. Not yet run in an environment.
- Added `references/environment-discovery.md`: read-only lookup queries for agents connected to Dynatrace (for example through the Dynatrace MCP server), with cost and safety rules.
- `SKILL.md`: the agent now checks for a Dynatrace connection, decides between a query and OpenPipeline, and returns a "use today / recommended setup / after setup" answer when OpenPipeline helps.

## 0.1.0 - 2026-11-23

First public version.

- `SKILL.md` with the conversion workflow and the checks that change results (precedence, case, nulls, `dedup`, hidden SPL limits, joins, statistics).
- Command and function mappings, checked against the Dynatrace DQL and Splunk SPL docs.
- 11 worked examples. Ten include a `data record(...)` test query with an expected result. These have not all been run in a Dynatrace environment yet.
- Notes on DQL syntax, performance, validation, and limitations.
