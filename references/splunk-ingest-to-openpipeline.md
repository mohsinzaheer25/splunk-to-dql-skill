# Splunk ingest config to OpenPipeline

How the settings in Splunk `props.conf` and `transforms.conf` map to Dynatrace. Use this when the user shares these files, or asks where their Splunk field extractions, masking, filtering, or index routing should go.

Splunk applies these settings at two different times, and the timing matters for where they go in Dynatrace:

- **Index time** (`TRANSFORMS`, `SEDCMD`, `LINE_BREAKER`, `TIME_*`, nullQueue routing): changes what's stored. In Dynatrace this is OneAgent config or OpenPipeline.
- **Search time** (`EXTRACT`, `REPORT`, `KV_MODE`, `FIELDALIAS`, `EVAL`, `LOOKUP`): runs on every search and doesn't change stored data. In Dynatrace it can stay in queries (`parse`, `fieldsAdd`), or move to OpenPipeline when the fields are used often. See "Query or OpenPipeline" in `SKILL.md`.

## props.conf

| Setting | Dynatrace | Notes |
|---|---|---|
| `[sourcetype]`, `[source::...]`, `[host::...]` stanza | Matching condition on the route or processor, for example `matchesValue(log.source, "/var/log/app/*")` | Grail has no sourcetype. Find which attribute identifies these logs first ([environment-discovery](environment-discovery.md)). |
| `EXTRACT-<class> = <regex>` | DQL processor: `parse content, "<DPL pattern>"` | Rewrite the regex in DPL. One processor can export several fields. |
| `EXTRACT-<class> = <regex> in <field>` | DQL processor: `parse <field>, "<DPL pattern>"` | |
| `REPORT-<class> = <transform>` | DQL processor, using the `REGEX` and `FORMAT` from the transform | See the transforms.conf table. |
| `KV_MODE = auto` | DQL processor with a `KVP{...}` pattern, or `parse` for the keys you need | Parse only the keys you need, so field count stays small (limit: 500 attributes per log). |
| `KV_MODE = json` | Usually nothing. OneAgent, the log ingest API, and OTLP already store JSON log attributes as fields. | If JSON stays inside `content`: DQL processor `parse content, "JSON:j" \| fieldsFlatten j` or `jsonExtract content`. |
| `KV_MODE = xml` | No XML matcher in DPL | Parse the elements you need with literals, or send JSON instead. |
| `INDEXED_EXTRACTIONS = json` | Same as `KV_MODE = json` | |
| `INDEXED_EXTRACTIONS = csv` / `tsv` / `psv` | DQL processor with a pattern per column, for example `LD:a ',' LD:b ',' LD:c` | Header detection isn't automatic. List the columns. |
| `FIELDALIAS-<class> = a AS b` | Rename fields processor, or `fieldsAdd b = a` | `AS` keeps the original field in Splunk. Rename removes it. |
| `EVAL-<field> = <expr>` | DQL processor: `fieldsAdd <field> = <expr>` | Translate functions with [function-mapping](function-mapping.md). |
| `LOOKUP-<class>` (automatic lookup) | Inline lookup processor for small static tables (100 KB per pipeline), or `lookup` in queries | DQL processors can't call `lookup`. |
| `SEDCMD-<class> = s/<regex>/<replacement>/g` | DQL processor: `fieldsAdd content = replacePattern(content, "<DPL>", "<replacement>")` | No capture-group backreferences. For masking, OneAgent sensitive data masking keeps the data from ever leaving the host. |
| `TIME_FORMAT`, `TIME_PREFIX`, `MAX_TIMESTAMP_LOOKAHEAD`, `TZ` | OneAgent: **Settings > Collect and capture > Log monitoring > Timestamp/splitting rules** | For logs from shippers or the API, send the timestamp as a field. A DQL processor can fix a wrong timestamp after ingest: `parse content, "TIMESTAMP('<format>'):timestamp"`. |
| `LINE_BREAKER`, `SHOULD_LINEMERGE`, `BREAK_ONLY_BEFORE` | OneAgent timestamp/splitting rules (entry boundary), or the log shipper's multiline settings | OpenPipeline receives records that are already split. It can't merge or split them. |
| `TRUNCATE` | No direct setting | Content up to 10 MB is accepted. Drop or trim in a DQL processor only if needed. |
| `TRANSFORMS-<class> = <transform>` | Depends on the transform: see the next table | |

## transforms.conf

| Transform | Dynatrace | Notes |
|---|---|---|
| `REGEX` + `FORMAT = name::$1` (index-time field) | DQL processor with `parse` | |
| `DEST_KEY = queue`, `FORMAT = nullQueue` (drop events) | **Drop record** processor, with the transform's `REGEX` as a matching condition, for example `contains(content, "[sshd]")` | Use **No storage assignment** instead if you still want metrics from these records. |
| Keep some events, drop the rest (setnull + setparsing) | Drop record with `not (<keep condition>)` | One processor does what two transforms do in Splunk. |
| `DEST_KEY = _MetaData:Index` (route to an index) | **Bucket assignment** processor with a custom bucket | The bucket sets retention and access, like an index. First matching processor wins. |
| `DEST_KEY = MetaData:Sourcetype` / `MetaData:Host` | DQL processor: `fieldsAdd log.source = "..."` or another attribute | Pick the attribute your queries filter on. |
| `DEST_KEY = _raw` (rewrite the event) | DQL processor: `fieldsAdd content = ...` | |
| `CLONE_SOURCETYPE` (store a copy) | No direct equivalent | To keep a modified copy, extract a business event (Data extraction stage), or forward the processed records to cloud storage. |
| `DEST_KEY = _TCP_ROUTING` / `_SYSLOG_ROUTING` (send elsewhere) | OpenPipeline forwarding to AWS S3, Azure Blob Storage, or Google Cloud Storage | Only object storage targets, not TCP or syslog. |
| `WRITE_META = true` | DQL processor with `parse` | The field is stored with the record. |
| `[lookup]` stanza with `filename = ...` | Upload the CSV as Grail lookup data and use `lookup [ load "/lookups/<name>" ]` in queries, or an Inline lookup processor for small tables | |

## Not covered by OpenPipeline

| Splunk | Where it goes instead |
|---|---|
| Forwarder-side filtering, `inputs.conf` blacklist | OneAgent log ingest rules, or the log shipper's filter |
| Summary indexing (`collect`, scheduled searches writing to an index) | Metric extraction for counts and values. Scheduled queries in Workflows for the rest. |
| Search-time `transaction` or `eventstats` in saved searches | Stays in queries. See [limitations](limitations.md). |
| Data model acceleration (`tstats`) | Metric extraction for the numbers you need often |

## Order of work

1. Get the expanded config for one sourcetype at a time, from both `props.conf` and every `transforms.conf` stanza it references.
2. Find which Dynatrace attribute identifies those logs. This becomes the route's matching condition.
3. Sort the settings into OneAgent (timestamps, line splitting, masking at the source), OpenPipeline (extractions, drops, routing to buckets), and queries (search-time logic you leave as is).
4. Build one pipeline per sourcetype, or per team. Test each processor with **Run sample data**.
5. Add the route above any broader route, then check the result with the queries in [openpipeline](openpipeline.md#verify-it-works).

Examples: [openpipeline-examples](openpipeline-examples.md).
