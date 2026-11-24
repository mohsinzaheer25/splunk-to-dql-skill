# Checking that the DQL matches the SPL

A query that runs is not proof that it's right. Compare results with Splunk on the same data before you replace an alert or a dashboard.

## 1. Test the logic with made-up rows

Before touching real data, run the DQL pipeline (everything after `fetch` and the source filters) on a few `data record(...)` rows. Include the cases that usually break:

- a record missing the group-by field
- values in different case (`ERROR`, `error`)
- a number stored as a string (`"500"`)
- duplicates with different timestamps (for `dedup`)
- an empty time bucket (for `timechart`)

Write down the result you expect before you run it. The [examples](examples.md) show this pattern.

## 2. Compare on real data

1. **Pick a fixed window.** Use an absolute timeframe on both sides, for example one hour from yesterday. In DQL use `fetch logs, timeframe: "2026-01-10T10:00:00Z/2026-01-10T11:00:00Z"`. Splunk shows times in the user's time zone, so convert to UTC before you set the window.
2. **Make sure the same data is in both.** If data was ingested into both products in parallel, check that the raw event count for the base search is close before comparing anything else. Different counts at this step mean a source or filter problem, not a translation problem.
3. **Turn off sampling.** Don't use `samplingRatio` or `scanLimitGBytes` in the comparison.
4. **Compare in this order:**
   1. raw event count after the base filters
   2. count after each `parse` (how many records got a value)
   3. number of groups after `summarize`
   4. totals: sums, counts, distinct counts
   5. the top N rows, including their order
   6. the time series shape, bucket by bucket
5. **Stop at the first difference.** Each step depends on the one before it, so fix that first.

## 3. Differences that are expected

These come from how the two products work. Write them down rather than trying to remove them:

- Approximate results: percentiles, `median`, `countDistinct` / `estdc`.
- Population vs sample `stdev` and `var`, unless you corrected the formula.
- First and last time buckets, when bucket alignment differs.
- Splunk limits that DQL doesn't have: 10 series plus OTHER in `timechart`, 10,000 rows from `sort`, 100 values from `list()`, 50,000 subsearch rows in `join`.
- Order of tied rows.

## 4. Common causes when numbers don't match

| Symptom | Usually caused by |
|---|---|
| DQL finds fewer events | case-sensitive `==` or `contains()`; `!=` dropping records without the field; a field that Splunk extracts at search time but Grail doesn't have |
| DQL finds more events | OR/AND precedence; missing source filter |
| An extra `null` group | SPL `stats` drops events without the by-field; add `filter isNotNull(...)` |
| Ratios are 0 | long / long integer division; use `toDouble()` |
| Different rows after `dedup` | missing `sort:`, or the dedup and filter run in a different order |
| Row count doubled after a join | `join` returns every match; SPL `join` takes one. Use `lookup` or dedup the right side |
| Timechart shows gaps instead of 0 | missing `default: 0` in `makeTimeseries` |
