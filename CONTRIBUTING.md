# Contributing

Thanks for helping. The skill is only useful if its mappings are correct, so every change needs proof that it's right.

## Ways to help

- **Report a wrong conversion.** Use the [wrong conversion](../../issues/new?template=wrong-conversion.yml) issue form. Say what the agent produced and what was wrong with it.
- **Report a missing mapping.** Use the [missing mapping](../../issues/new?template=missing-mapping.yml) form for an SPL command or function the skill doesn't cover.
- **Run the existing test queries.** Run a `data record(...)` query from `references/examples.md` in a Dynatrace Notebook or the Playground, and report whether the result matches. This is the most useful thing anyone can do right now.
- **Open a pull request** with a fix or a new example.

## Made-up data only

This repo is public. Everything in an issue or pull request stays visible, even after it's edited or deleted.

Don't paste real queries, log lines, or results from your company or a client. Rewrite them first:

| Replace | With |
|---|---|
| hostnames, service names | `web-01`, `cart`, `auth` |
| IP addresses | documentation ranges: `192.0.2.x`, `198.51.100.x`, `203.0.113.x` |
| user names, emails | `alice`, `bob`, `user@example.com` |
| customer, account, order IDs | `c1`, `acct-123`, `order-42` |
| index, sourcetype, bucket names that reveal internal systems | `app`, `web`, `payments` |
| Dynatrace entity IDs, tenant URLs, API tokens | `HOST-0000000000000001`, `<tenant>`, never post tokens |

Keep the structure that matters for the bug, such as field names, value types, and the shape of the log line. Change everything that identifies a person, company, or system.

Issues and pull requests with real data will be closed and the content removed.

## What a change needs

**For a new or changed mapping** in `command-mapping.md` or `function-mapping.md`:

1. A link to the Dynatrace or Splunk docs page that supports it. Use the current docs (`docs.dynatrace.com`, `help.splunk.com`).
2. A short `data record(...)` query that shows the behavior, with the result you got.
3. A "Watch out" note if the SPL and DQL versions can return different results.

**For a new example** in `examples.md`, follow the existing format:

- **SPL**, **DQL** (with `<placeholders>` for environment-specific values), **Notes**, **Check**.
- The Check query must run without real data, using `data record(...)` only.
- Include the expected result as a table.
- Include at least one row that would catch a common mistake (a null field, a different case, a duplicate).

**In the pull request**, say whether you ran the Check queries in Dynatrace. "Checked against docs only" is fine. Just say so.

## Writing style

- Plain language and short sentences. Write the way you'd explain it to a colleague.
- Say exactly what differs and when. "`dedup` keeps a random record unless you pass `sort:`" is useful. "Be careful with dedup" isn't.
- Don't claim something works unless you've checked it.
- Keep `SKILL.md` short. It's loaded every time the skill runs, so detail belongs in `references/`.

## Pull requests

1. Fork the repo and create a branch.
2. Make one focused change per pull request.
3. Add a line under `Unreleased` in `CHANGELOG.md`.
4. Fill in the pull request template.

The maintainer reviews every change. Mapping changes are only merged when the source link supports them.

## License

By contributing, you agree that your contribution is licensed under the [MIT License](LICENSE).
