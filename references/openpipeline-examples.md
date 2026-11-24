# OpenPipeline examples

Each example starts from a Splunk setup and shows the matching OpenPipeline configuration for logs. All values are made up.

How to test:
- **Processing stage processors** (DQL, Add/Remove/Rename fields, Drop record): paste the sample record into the processor's sample data field and select **Run sample data**. Compare the output with the expected result.
- **Other stages** (Bucket assignment, Metric extraction): the editor may not preview these. Check them after logs arrive, with the query under each example.

The expected results were worked out from the Dynatrace docs and haven't all been run yet. If a preview differs, trust the preview and please [report it](../CONTRIBUTING.md).

---

## 1. Extract key=value fields

**Splunk**
```ini
# props.conf
[payments_app]
EXTRACT-order = order=(?<order_id>\S+) amount=(?<amount>[\d.]+) status=(?<status>\w+)
```

**Processor:** Processing stage > **DQL**

- Matching condition:
  ```dql
  matchesValue(log.source, "payments-api") and matchesPhrase(content, "order")
  ```
- Definition:
  ```dql
  parse content, "LD 'order=' NSPACE:order.id ' amount=' DOUBLE:order.amount ' status=' WORD:order.status"
  ```

**Sample record**
```json
{
  "content": "2026-01-10T10:00:00Z INFO order=o-42 amount=19.99 status=paid",
  "log.source": "payments-api"
}
```

**Expected:** three new fields, `order.id` = `o-42`, `order.amount` = `19.99` (number), `order.status` = `paid`. `content` is unchanged.

**Notes**
- `LD` skips the start of the line, because a DPL pattern matches from the beginning. The literals `' amount='` and `' status='` include the space, so the pattern only matches when the keys appear in this order.
- If any part doesn't match, all three fields are null for that record. For logs where keys appear in any order, use one `parse` per key in the same processor, chained with `|`.

**Query after setup**
```dql
fetch logs, from: -1h
| filter log.source == "payments-api" and isNotNull(order.id)
| summarize revenue = sum(order.amount), by: {order.status}
```

---

## 2. JSON inside `content`

**Splunk**
```ini
[orders_json]
KV_MODE = json
```

First check whether this is needed. OneAgent, the log ingest API, and OTLP store JSON attributes as fields already. Only use this processor when the JSON is still a string inside `content`.

**Processor:** Processing stage > **DQL**

- Matching condition:
  ```dql
  matchesValue(log.source, "orders-api") and matchesValue(content, "{*")
  ```
- Definition:
  ```dql
  parse content, "JSON:parsedJson"
  | fieldsAdd user.id = parsedJson[user][id], order.total = parsedJson[total]
  | fieldsRemove parsedJson
  ```

**Sample record**
```json
{
  "content": "{\"event\":\"checkout\",\"total\":42.5,\"user\":{\"id\":\"u-7\",\"tier\":\"gold\"}}",
  "log.source": "orders-api"
}
```

**Expected:** `user.id` = `u-7`, `order.total` = `42.5`. No `parsedJson` field remains.

**Notes**
- Picking the fields you need keeps the record small. To bring every key to the top level, use `parse content, "JSON:j" | fieldsFlatten j` instead (fields get the prefix `j.`), but check the attribute limit (500 per log).
- Dynatrace's best practice for large JSON is a JSON matcher that names only the fields you need, for example `JSON{STRING:event}(flat=true)`, because auto-discovery scans the whole structure.

---

## 3. Fix the timestamp

**Splunk**
```ini
[batch_jobs]
TIME_FORMAT = %d/%m/%Y %H:%M:%S
```

For OneAgent logs, set the format in OneAgent's timestamp rules first (**Settings > Collect and capture > Log monitoring > Timestamp/splitting rules**), because the timestamp also controls how lines are split. Use this processor for logs from other sources whose timestamp is only in the text.

**Processor:** Processing stage > **DQL**

- Matching condition:
  ```dql
  matchesValue(log.source, "batch-jobs")
  ```
- Definition:
  ```dql
  parse content, "TIMESTAMP('dd/MM/yyyy HH:mm:ss'):timestamp SPACE UPPER:loglevel"
  ```

**Sample record**
```json
{
  "content": "10/01/2026 10:15:30 WARN disk usage 91%",
  "log.source": "batch-jobs",
  "timestamp": "2026-01-10T10:20:00Z"
}
```

**Expected:** `timestamp` becomes 10 January 2026, 10:15:30, and `loglevel` = `WARN`.

**Notes**
- The format uses Java-style letters (`dd`, `MM`, `yyyy`), not Splunk's `%d/%m/%Y`.
- Check the time zone of the parsed value in the preview. If the logs are written in local time, the result can be off by the zone offset.
- If the pattern doesn't match, the original `timestamp` is kept. In OpenPipeline, `parse` keeps existing fields when parsing fails (`preserveFieldsOnFailure` is on). Test with lines that don't match to make sure.

---

## 4. Mask data

**Splunk**
```ini
[web_access]
SEDCMD-mask_ip = s/\d+\.\d+\.\d+\.\d+/x.x.x.x/g
```

**Processor:** Processing stage > **DQL**

- Matching condition:
  ```dql
  matchesValue(log.source, "web-access")
  ```
- Definition:
  ```dql
  fieldsAdd content = replacePattern(content, "IPADDR", "x.x.x.x")
  ```

**Sample record**
```json
{
  "content": "GET /cart from 192.0.2.10 via 198.51.100.7 status=200",
  "log.source": "web-access"
}
```

**Expected:** `content` = `GET /cart from x.x.x.x via x.x.x.x status=200`.

**Notes**
- Masking in OpenPipeline happens after the data reaches Dynatrace. If the data must never leave the host, use OneAgent sensitive data masking instead.
- To keep part of the value, such as the network, parse it into a field and use `ipMask(ip, 24)`, which sets the last octet to 0.

---

## 5. Drop noise, but keep a count

**Splunk**
```ini
# props.conf
[web_access]
TRANSFORMS-null = drop_healthchecks

# transforms.conf
[drop_healthchecks]
REGEX = GET /healthz
DEST_KEY = queue
FORMAT = nullQueue
```

**Option A: drop completely.** Processing stage > **Drop record**

- Matching condition:
  ```dql
  matchesValue(log.source, "web-access") and contains(content, "GET /healthz")
  ```

Expected in Run sample data: a matching sample record is dropped, and a non-matching one passes through unchanged.

**Option B: don't store, but count.** Use this when someone still wants to see how many health checks there were.

1. Bucket assignment stage > **No storage assignment**, with the same matching condition.
2. Metric extraction stage > **Counter metric**:
   - Matching condition: the same condition
   - Metric key: `log.web.healthchecks`
   - Dimension: `log.source`

The record goes through every stage, adds 1 to the counter, and isn't stored.

**Query after setup**
```dql
timeseries checks = sum(log.web.healthchecks), by: {log.source}, interval: 5m
```

Check that the records really are gone:

```dql
timeseries dropped = sum(dt.sfm.openpipeline.not_stored.records), by: {pipeline_id, reason}
```

Option A shows `reason` = `intentionally_dropped`. Option B shows `not_persisted`.

---

## 6. Route to a bucket instead of an index

**Splunk**
```ini
# transforms.conf
[route_audit]
REGEX = AUDIT
DEST_KEY = _MetaData:Index
FORMAT = audit
```

The Splunk index `audit` had longer retention and its own access rules. In Dynatrace, a bucket does both.

1. Create the bucket: **Settings > Storage management > Bucket storage management > Bucket**, name `audit_logs`, retention for example 1095 days (3 years), table type `logs`.
2. Bucket assignment stage > **Bucket assignment**:
   - Matching condition:
     ```dql
     matchesValue(log.source, "auth-service") and matchesPhrase(content, "AUDIT")
     ```
   - Storage: `audit_logs`
3. Give the right users read access to `audit_logs` in Account Management.

**Notes**
- Bucket assignment uses the first matching processor. Put the narrowest condition at the top of the stage.
- Splunk's `REGEX = AUDIT` is case-sensitive. `matchesPhrase` isn't. Use `contains(content, "AUDIT")` if the case matters.

**Query after setup**
```dql
fetch logs, bucket: {"audit_logs"}, from: -1h
| summarize count = count(), by: {dt.system.bucket}
```

Querying only this bucket also makes audit queries cheaper, because Grail scans less data.
