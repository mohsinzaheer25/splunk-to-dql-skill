# SPL functions to DQL

The "Watch out" column lists cases where a function with the same name, or the obvious match, returns a different value.

## eval functions

| SPL | DQL | Watch out |
|---|---|---|
| `if(c, a, b)` | `if(c, a, else: b)` | `else:` is a named parameter. Without it, the result is null when `c` is false. |
| `case(c1, v1, c2, v2, true(), d)` | `if(c1, v1, else: if(c2, v2, else: d))` | DQL has no `case`. Nest `if`. |
| `coalesce(a, b)` | `coalesce(a, b)` | |
| `nullif(a, b)` | `if(a != b, a)` | Returns null when they are equal. |
| `isnull(x)` / `isnotnull(x)` | `isNull(x)` / `isNotNull(x)` | |
| `typeof(x)` | `type(x)` | Returns DQL type names (`long`, `string`, ...), not SPL's (`Number`, `String`). |
| `tonumber(s)` | `toDouble(s)` or `toLong(s)` | Returns null if the string isn't a number. |
| `tonumber(s, 16)` | `hexStringToNumber(s)` | |
| `tostring(x)` | `toString(x)` | |
| `a . b` (concatenation) | `concat(a, b)` | |
| `len(s)` | `stringLength(s)` | SPL counts code points, DQL counts UTF-16 units. They differ for emoji. |
| `lower(s)` / `upper(s)` | `lower(s)` / `upper(s)` | |
| `trim(s)` | `trim(s)` | DQL trims whitespace only. SPL `trim(s, chars)` has no direct equivalent. |
| `substr(s, start, len)` | `substring(s, from: start - 1, to: start - 1 + len)` | SPL is 1-based with a length. DQL is 0-based with an exclusive end. |
| `replace(s, regex, r)` | `replaceString(s, "literal", r)` or `replacePattern(s, "<DPL>", r)` | No regex and no capture-group backreferences. |
| `match(s, regex)` | `matchesPattern(s, "<DPL>")` | SPL matches anywhere in the string. DPL matches from the start, so put `LD` in front if you need that. |
| `like(s, "a%")` | `like(s, "a%")` | Same `%` and `_` wildcards. |
| `searchmatch("x")` | `matchesPhrase(content, "x")` | |
| `split(s, ",")` | `splitString(s, ",")` | |
| `urldecode(s)` | `decodeUrl(s)` | |
| `cidrmatch("10.0.0.0/8", ip)` | `ipIn(ip, "10.0.0.0/8")` | Arguments are in the opposite order. |
| `round(x, 2)` | `round(x, decimals: 2)` | |
| `floor`, `ceil`, `abs`, `sqrt`, `exp` | same names | |
| `pow(x, y)` | `power(x, y)` | |
| `log(x)` | `log10(x)` | SPL `log` is base 10. DQL `log` is the natural log. |
| `ln(x)` | `log(x)` | |
| `now()` | `now()` | |
| `relative_time(now(), "-1d@d")` | `now() - 1d @ d` | Use `@` for alignment. |
| `strftime(_time, "%Y-%m-%d")` | `formatTimestamp(timestamp, format: "yyyy-MM-dd")` | Different format letters (Java style). |
| `strptime(s, fmt)` | `toTimestamp(s)` for ISO strings, else `parse s, "TIMESTAMP('yyyy-MM-dd HH:mm:ss'):ts"` | |
| `mvcount(m)` | `arraySize(m)` | |
| `mvindex(m, i)` | `m[i]` or `arrayElement(m, i)` | Both 0-based. Negative indexes count from the end. |
| `mvjoin(m, ",")` | `arrayToString(m, delimiter: ",")` | |
| `mvdedup(m)` | `arrayDistinct(m)` | |
| `mvsort(m)` | `arraySort(m)` | |
| `mvappend(a, b)` | `arrayConcat(a, b)` | Skips non-array arguments. Wrap scalars with `array(x)`. |
| `mvfind`, `mvmap`, `mvfilter` | Iterative expressions (`m[]`) or `expand` + `filter` | No direct equivalent. |

## stats, chart, and timechart functions

| SPL | DQL | Watch out |
|---|---|---|
| `count` | `count()` | |
| `count(f)` | `countIf(isNotNull(f))` | DQL `count()` takes no argument. |
| `count(eval(c))` | `countIf(c)` | |
| `sum`, `avg`, `min`, `max` | same names | |
| `range(x)` | `max(x) - min(x)` | |
| `dc(f)` / `distinct_count(f)` | `countDistinctExact(f)` | `countDistinct` is approximate. The exact version fails above 1M distinct values. |
| `estdc(f)` | `countDistinctApprox(f)` | |
| `values(f)` | `arraySort(collectDistinct(f))` | SPL sorts the values. DQL doesn't guarantee an order. |
| `list(f)` | `collectArray(f)` | SPL keeps the first 100 values in event order. DQL keeps up to 131,072 values in no guaranteed order. |
| `first(f)` / `last(f)` | `sort timestamp desc`, then `takeFirst(f)` / `takeLast(f)` | Without the `sort`, the result can change on every run. |
| `earliest(f)` / `latest(f)` | `sort timestamp asc`, then `takeFirst(f)` / `takeLast(f)` | |
| `earliest(_time)` / `latest(_time)` | `min(timestamp)` / `max(timestamp)` | |
| `median(x)` | `median(x)` | Both are approximations and can differ slightly. |
| `perc95(x)` / `p95(x)` | `percentile(x, 95)` | DQL gives an estimate. |
| `exactperc95(x)` | `percentile(x, 95)` | DQL has no exact percentile, so say the result is approximate. |
| `stdevp(x)` / `varp(x)` | `stddev(x)` / `variance(x)` | Both are population statistics. |
| `stdev(x)` / `var(x)` | `stddev(x) * sqrt(toDouble(n) / (n - 1))`, where `n = count()` / `variance(x) * n / (n - 1)` | SPL uses sample statistics. Compute `n` in the same `summarize`, then correct with `fieldsAdd`. |
| `sumsq(x)` | `sum(x * x)` | |
| `mode(f)` | `summarize c = count(), by: {f}` then `sort c desc \| limit 1` | No `mode()` aggregation. With a by-clause this needs `dedup` per group. |
| `rate(x)`, `per_second(x)` | `timeseries` with `rate:`, or `makeTimeseries` + array math | Depends on the data. |

## Sources

- [DQL functions](https://docs.dynatrace.com/docs/platform/grail/dynatrace-query-language/functions)
- [Splunk evaluation functions](https://help.splunk.com/en/splunk-enterprise/search/spl-search-reference/10.0/evaluation-functions/comparison-and-conditional-functions)
- [Splunk statistical functions](https://help.splunk.com/en/splunk-enterprise/search/spl-search-reference/10.0/statistical-and-charting-functions/aggregate-functions)
