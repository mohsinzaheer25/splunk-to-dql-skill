# OpenPipeline for logs

OpenPipeline processes logs at ingest, before they are stored in Grail. Use it to extract fields once instead of running `parse` in every query, to mask data, to drop or skip storing logs, to create metrics, and to choose the bucket each log is stored in.

This file covers logs only. All UI paths start at **Settings > Process and contextualize > OpenPipeline > Logs**.

## When to suggest OpenPipeline

Most of the decision can be read from the SPL and the request. Ask the user only for what you can't tell.

| Signal | Suggest |
|---|---|
| A one-off or occasional search | Query-time `parse`. A pipeline isn't worth setting up. |
| `rex`, `spath`, `kv`, or `extract` in a dashboard panel, alert, or saved search that runs often | Extract the fields in a DQL processor. The query then reads the stored fields. |
| The same extraction appears in several of the user's searches | Extract once in a pipeline, so every query reuses the fields. |
| An alert or panel that only counts matching lines (`stats count`, `timechart count`) | A counter metric in the Metric extraction stage. Query it with `timeseries`. |
| A numeric value charted over time (`timechart avg(duration)`) | A value metric, or a histogram metric for percentiles. |
| `rex mode=sed` or `SEDCMD` that hides personal data | A DQL processor with `replacePattern` or `replaceString`. Masking in a query doesn't change what's stored. |
| Splunk drops events with nullQueue, or most events are noise | Drop record (nothing kept), or No storage assignment (metrics still extracted, record not stored). |
| Different retention or access per index | A custom bucket and the Bucket assignment stage. |

Questions worth asking when the SPL doesn't answer them:

1. Is this an ad hoc search, a dashboard panel, or an alert? How often does it run?
2. Roughly how many of these logs arrive per day?
3. Can you edit OpenPipeline settings, or does another team own them?

When you recommend OpenPipeline, give the user:

1. **Use today:** the DQL with `parse`, so they have a working query now.
2. **Recommended setup:** the pipeline, processors, and route, with a sample record to test with **Run sample data**.
3. **After setup:** the shorter query that uses the extracted fields, without `parse`.

## How data flows

1. **Ingest source.** Logs arrive from OneAgent, the log ingest API (`/api/v2/logs/ingest`), OTLP (`/api/v2/otlp/v1/logs`), Amazon Data Firehose, or an extension. The source is stored in `dt.openpipeline.source`.
2. **Routing.** Dynamic routes are checked from top to bottom. The first route whose matching condition is true sends the record to its pipeline, and later routes are skipped. Records that match no route go to the default route.
3. **Pipeline.** The record runs through the stages below, in a fixed order.
4. **Storage.** The record is stored in the bucket the pipeline assigned, otherwise in `default_logs`.

Stages for logs, in execution order:

| Stage | What it does | Processors run |
|---|---|---|
| Processing | Parse, add, remove, rename, mask, or drop | All that match |
| Smartscape node / edge | Add topology | All that match |
| Permission | Set `dt.security_context` | First match only |
| Product allocation / Cost allocation | Set `dt.cost.product` / `dt.cost.costcenter` | First match only |
| Bucket assignment | Pick the bucket, or don't store | First match only |
| Metric extraction | Counter, value, and histogram metrics | All that match |
| Davis | Create Davis events for alerting | All that match |
| Data extraction | Create business events or SDLC events | All that match |

Within a stage, processors run from top to bottom, and each one sees the output of the one before it.

## Processors in the Processing stage

| Processor | Use it for |
|---|---|
| **DQL** | Parsing and anything computed. Limited DQL, see below. |
| **Add fields** | Static values only, for example `team = payments`. |
| **Remove fields** / **Rename fields** | Simple field changes without writing DQL. |
| **Drop record** | Discard the record. Nothing later runs, nothing is stored. |
| **Inline lookup** | Map a string field through a key-value table, for example status code to text. Up to 100 KB of table data per pipeline. |
| **Technology bundle** | Built-in parsers for common formats: Syslog, Nginx, Apache HTTP, IIS, Java, Python, Kafka, AWS services, and others. |
| **GeoIP lookup** | Add location fields from an IP address. Early Access only. |

### DQL in a DQL processor

A pipeline DQL processor is not a full query. It works on one record at a time.

- **Commands allowed:** `parse`, `jsonExtract`, `fields`, `fieldsAdd`, `fieldsKeep`, `fieldsRemove`, `fieldsRename`, `fieldsFlatten`.
- **Not allowed:** `filter`, `summarize`, `dedup`, `sort`, `limit`, `expand`, `lookup`, `join`, and anything else that works across records. Put filtering in the matching condition instead.
- **Functions:** most string, conversion, conditional, time, array, network, and math functions. See [DQL functions in OpenPipeline](https://docs.dynatrace.com/docs/platform/openpipeline/reference/dql/openpipeline-dql-functions).
- **Maximum length:** 8,192 characters.

A query converted from SPL can't be pasted into a pipeline as-is. Keep only the `parse` and `fieldsAdd` parts, and move the filters into the matching condition.

### Matching conditions

Every processor and route has a matching condition. It's a DQL expression that returns true or false, for example:

```dql
matchesValue(log.source, "/var/log/payments/*") and matchesPhrase(content, "order")
```

- `matchesValue` and `matchesPhrase` are case-insensitive. `==` is case-sensitive.
- `true` matches every record.
- Maximum length is 4,096 characters.

## Set up a pipeline in the UI

1. **Find the records.** In a Notebook, write a query that returns exactly the logs you want to process. The filter becomes your matching condition, for example `matchesValue(log.source, "payments-api")`. Copy one `content` value to use as sample data.
2. **Create a bucket (optional).** Do this only when the logs need different retention or access. Go to **Settings > Storage management > Bucket storage management > Bucket**, set a name and retention in days, and choose table type `logs`.
3. **Create the pipeline.** Under **Pipelines**, select **Pipeline** and give it a name.
4. **Add processors.** For each stage you need, select **Processor**, choose the type, and fill in the name, matching condition, and definition.
5. **Test each processor.** Paste a sample record, as JSON with at least `content`, into the sample data field and select **Run sample data**. Check that the output fields have the right names and types.
6. **Save** the pipeline.
7. **Route data to it.** Under **Dynamic routing**, select **Dynamic route** and set the name, matching condition, and target pipeline. Move the route above any broader route, because the first match wins. Save.
8. **Verify** (next section).

## Verify it works

New logs are processed. Logs that were already stored aren't changed.

Check that records reach the pipeline and have the new fields:

```dql
fetch logs, from: -15m
| filter <same condition as the route>
| fields timestamp, dt.openpipeline.pipelines, <new fields>
| limit 20
```

Check for records that were dropped or not stored, and why:

```dql
timeseries records = sum(dt.sfm.openpipeline.not_stored.records), by: {pipeline_id, reason}
```

The `reason` values include `intentionally_dropped` (Drop record), `not_persisted` (No storage assignment), `not_valid` (missing `timestamp` or `content`), and `buffer_overflow` (processing memory ran out, usually from parsing that's too heavy). The **OpenPipeline usage overview** dashboard under **Dashboards > Ready-made** shows the same data.

## Best practices

From the OpenPipeline and log management docs:

- **Narrow matching conditions.** A specific condition on each processor speeds up processing and lowers the risk that a DQL processor is disabled for being too complex.
- **Parse each field once.** Parse `content` in one processor and reuse the extracted fields, instead of parsing it again in later processors.
- **Specific DPL patterns.** Start with a literal or typed matcher (`TIMESTAMP`, `LONG`, `IPADDR`, `DQS`). Follow `LD` or `DATA` straight away with a literal. Avoid optional markers (`?`), lookarounds, and large alternative groups where you can.
- **Small DQL processors.** Split complex logic into several processors. Use Add, Remove, and Rename fields processors for simple changes.
- **Temporary fields.** Name intermediate values `dt.temp.<name>`. They're available to later processors in the same pipeline and are removed before storage.
- **Explicit routes.** Route as much data as possible to custom pipelines with explicit conditions, and keep the default route for data you haven't assigned yet.
- **Drop vs don't store.** Use Drop record when nothing is needed. Use No storage assignment when you still want metrics or Davis events from the record.
- **Bucket size.** Dynatrace suggests around 2–3 TB of retained data per day per frequently queried bucket. Don't create a bucket per application when volume is low.

## Limits that come up often

| Limit | Value |
|---|---|
| Pipelines, routes, ingest sources per configuration scope | 100 each |
| Processors per pipeline | 1,000 |
| DQL processor definition | 8,192 characters |
| Matching condition | 4,096 characters |
| Log `content` | 10 MB (512 kB in the classic pipeline) |
| Attributes per log / attribute value | 500 / 32 kB |
| Record size after processing | 16 MB |
| Log age at ingest | 24 or 72 hours, depending on the environment |
| Pipelines that can extract data from one record | 5 |

A log record without `timestamp` and `content` after processing fails validation and is dropped.

Fields like `dt.entity.service`, `dt.entity.kubernetes_cluster`, and `dt.source_entity` are added after the Processing stage. They can't be used in routes or in the Processing stage, only in later stages. The fields `dt.ingest.*`, `dt.openpipeline.*`, `dt.retain.*`, and `dt.system.*` are read-only.

Full list: [OpenPipeline limits](https://docs.dynatrace.com/docs/platform/openpipeline/reference/limits) and [Log Management default limits](https://docs.dynatrace.com/docs/analyze-explore-automate/logs/lma-limits).

## What OpenPipeline doesn't do

- **Line splitting and multiline events.** For OneAgent logs, this is set in OneAgent's timestamp and splitting rules (**Settings > Collect and capture > Log monitoring**). Log shippers and the ingest API send records that are already split.
- **Cross-record logic.** OpenPipeline has no aggregation, deduplication, or joins. Those stay in queries.
- **Changing stored data.** Pipelines apply only to newly ingested logs.

## Environments still on the classic pipeline

If **Pipelines** shows a **Classic pipeline**, some logs are still processed by the classic rules. Before moving them to OpenPipeline:

- Field names become case-sensitive in OpenPipeline. Logs from the ingest API were lowercased by the classic pipeline. Queries and alerts that depend on that casing break.
- The classic pipeline set `event.type == "LOG"` on every record. OpenPipeline doesn't, so matching conditions that use it stop matching.
- Dynatrace provides an API that translates classic rules into an OpenPipeline pipeline. Treat its output as a starting point and check every processor.

See [Upgrade from classic pipeline to OpenPipeline](https://docs.dynatrace.com/docs/platform/upgrade/upgrade-your-data-pipeline/migration-classic-pipeline).

## Permissions

- **Pipelines:** users need `settings:objects:read` and `settings:objects:write` on `builtin:openpipeline.logs.pipelines`. The creator owns a new pipeline, and only the owner and admins can see it until it's shared (**Share** in the pipeline menu).
- **Routes:** routes have no owner. Anyone with write access to `builtin:openpipeline.logs.routing` can change every log route, so grant it carefully.
- **Buckets:** creating buckets needs the storage bucket permissions. Users also need read access to a bucket to query it.

## Automation

Everything in the UI is stored as Settings objects. To manage pipelines as code, use the Settings API with the schemas `builtin:openpipeline.logs.pipelines` and `builtin:openpipeline.logs.routing`. The pipeline editor shows the JSON for an existing pipeline, so it can be copied.

The OpenPipeline API can check configuration before you save it (needs `openpipeline:configurations:read`):

- `POST /platform/openpipeline/v1/dqlProcessor/verify`: check a DQL processor definition
- `POST /platform/openpipeline/v1/matcher/verify`: check a matching condition
- `POST /platform/openpipeline/v1/preview/processor`: run a processor against sample data

See the [OpenPipeline API](https://docs.dynatrace.com/docs/dynatrace-api/platform-api/platform-openpipeline-v1) reference.

## Related files

- [splunk-ingest-to-openpipeline.md](splunk-ingest-to-openpipeline.md): `props.conf` and `transforms.conf` settings mapped to OpenPipeline
- [openpipeline-examples.md](openpipeline-examples.md): processor examples you can test with Run sample data
