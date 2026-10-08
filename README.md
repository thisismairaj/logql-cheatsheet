<h1 align="center">logql-cheatsheet</h1>

<p align="center">
  A tight, scannable LogQL reference — one page instead of several sprawling docs pages.<br/>
  Pairs naturally with <a href="https://github.com/thisismairaj/logcli-shortcuts">logcli-shortcuts</a> if you're running these from the terminal.
</p>

---

## Why

Grafana's own LogQL docs are accurate and complete, but spread across several long pages. This
is the version you actually want open in a second tab while debugging: every operator, every
parser, every aggregation function, on one page, with a real example next to each one — plus a
"recipes" section of combined queries for actual troubleshooting, not just syntax in isolation.

Every example here is verified against Grafana's own docs, not reconstructed from memory.

---

## Table of contents

- [Stream selectors](#stream-selectors)
- [Line filters](#line-filters)
- [Parsers](#parsers)
- [Label filters](#label-filters)
- [Line format / label format](#line-format--label-format)
- [Drop / keep labels](#drop--keep-labels)
- [Metric queries](#metric-queries)
- [Recipes](#recipes)

---

## Stream selectors

Every LogQL query starts with a stream selector — which log streams to read, by label.

| Operator | Meaning |
|---|---|
| `=` | exactly equal |
| `!=` | not equal |
| `=~` | regex matches |
| `!~` | regex does not match |

```logql
{app="mysql", name="mysql-backup"}
{name =~ "mysql.+"}
{name !~ "mysql.+"}
{name !~ `mysql-\d+`}
```

**Gotcha:** unlike line filter regexes below, `=~`/`!~` here are **fully anchored** — the regex
must match the *entire* label value, not just part of it.

---

## Line filters

Filter log lines by content, after the stream selector, before any parsing.

| Operator | Meaning |
|---|---|
| `\|=` | line contains string |
| `!=` | line does not contain string |
| `\|~` | line matches regex |
| `!~` | line does not match regex |

```logql
{job="mysql"} |= "error"
{instance=~"kafka-[23]", name="kafka"} != "kafka.server:type=ReplicaManager"
{name="kafka"} |~ "tsdb-ops.*io:2003"
{name="cassandra"} |~ `error=\w+`
{job="mysql"} |= "error" != "timeout"          # chain filters — AND semantics
```

Case-insensitive: prefix the regex with `(?i)` — `|~ "(?i)error"`.

Strip ANSI color codes before filtering/parsing: `| decolorize`.

---

## Parsers

Extract labels from the line content. Each parser slots into the pipeline after `|`.

**JSON** — extracts every top-level property as a label, or pick specific fields:
```logql
| json
| json first_server="servers[0]", ua="request.headers[\"User-Agent\"]"
```

**logfmt** (`key=value key2=value2` lines) — all fields, specific fields, or strict/sparse modes:
```logql
| logfmt
| logfmt host, fwd_ip="fwd"
| logfmt --strict
| logfmt --keep-empty --strict host
```

**pattern** — the fastest to write for well-known line shapes (e.g. nginx access logs):
```logql
| pattern "<ip> - - <_> \"<method> <uri> <_>\" <status> <size> <_>"
```

**regexp** — needs at least one named capture group:
```logql
| regexp "(?P<method>\w+) (?P<path>[\w|/]+) \((?P<status>\d+?)\)"
```

**unpack** — for lines packed by Promtail's `pack` stage; also restores the original line into
the special `_entry` property:
```logql
| unpack
```

---

## Label filters

Filter on labels — either stream labels or ones a parser just extracted — with real typed
comparisons, not just string equality.

| Operator | Meaning |
|---|---|
| `=` or `==` | equality |
| `!=` | inequality |
| `>`, `>=` | greater than (/or equal) |
| `<`, `<=` | less than (/or equal) |

Value types: string (`"value"`), duration (`"300ms"`, `"1.5h"`, `"2h45m"`), number (`250`,
`89.923`), bytes (`"42MB"`, `"1.5KiB"`, `"20B"`).

```logql
| duration > 1m and bytes_consumed > 20MB
| duration >= 20ms or size == 20KB and method!~"2.."
| duration >= 20ms or (method="GET" and size <= 20KB)
```

Chain predicates with `and`, `or`, a comma, a pipe, or just whitespace.

---

## Line format / label format

**`line_format`** rewrites the displayed line from a Go template over extracted labels (math
functions like `div` work inside it):
```logql
{container="frontend"} | logfmt | line_format "{{.query}} {{.duration}}"
{...} | line_format "{{.ip}} {{.status}} {{div .duration 1000}}"
```

**`label_format`** renames or recomputes labels — `dst=src` renames, `dst="{{.a}} {{.b}}"`
recomputes from a template:
```logql
| label_format dst=src
| label_format dst="{{.status}} {{.query}}"
```

---

## Drop / keep labels

```logql
{job="varlogs"} | json | drop level, path, app=~"some-api.*"
{job="varlogs"} | json | keep level, method="GET"
```

---

## Metric queries

Turn log streams into numeric time series.

**Log range aggregations** — the function wraps a *range vector* (a selector with a duration in
`[ ]`):

| Function | Meaning |
|---|---|
| `rate(log-range)` | entries per second |
| `count_over_time(log-range)` | entry count in the range |
| `bytes_rate(log-range)` | bytes per second |
| `bytes_over_time(log-range)` | total bytes in the range |
| `absent_over_time(log-range)` | 1 if the range is empty, else nothing |

```logql
count_over_time({job="mysql"}[5m])
count_over_time({job="mysql"}[5m] offset 5m)   # offset goes INSIDE the [ ], not after
```

**Unwrap** — pull a numeric value out of a label instead of just counting lines:
```logql
| unwrap duration_seconds(duration)   # or: | unwrap bytes(size)
```
Then wrap with: `rate`, `rate_counter`, `sum_over_time`, `avg_over_time`, `max_over_time`,
`min_over_time`, `first_over_time`, `last_over_time`, `stdvar_over_time`, `stddev_over_time`,
`quantile_over_time(scalar, unwrapped-range)`.

**Aggregation operators** — aggregate a vector down, same shape as PromQL:

```
<aggr-op>([parameter,] <vector expression>) [without|by (<label list>)]
```

`sum`, `avg`, `min`, `max`, `stddev`, `stdvar`, `count`, `topk`, `bottomk`, `sort`, `sort_desc`.
`by (labels)` keeps only those labels; `without (labels)` drops just those and keeps the rest.

```logql
sum by (status) (count_over_time({job="nginx"}[5m]))
topk(5, sum by (path) (rate({job="nginx"} | json [5m])))
```

For very high-cardinality instant queries, `approx_topk(k, <vector expression>)` is a
probabilistic, faster alternative to `topk` (instant queries only; group with an inner `sum by`
first instead of a grouping clause on `approx_topk` itself).

---

## Recipes

Combined, real-troubleshooting versions of the above — the part a syntax reference alone doesn't
give you.

**All 5xx responses for a service, last hour:**
```logql
{namespace="backend-prod", service_name="api"} | json | status >= 500
```

**Requests slower than 1s, with the duration visible:**
```logql
{namespace="backend-prod", service_name="api"} | json | duration > 1s | line_format "{{.duration}} {{.path}}"
```

**Error rate per minute, grouped by error type:**
```logql
sum by (error_type) (rate({namespace="backend-prod"} | json | error_type != "" [1m]))
```

**Case-insensitive search for a term across a whole namespace:**
```logql
{namespace="backend-prod"} |~ "(?i)timeout"
```

**Parse an nginx-style access line without writing a regex:**
```logql
{job="nginx"} | pattern "<ip> - - <_> \"<method> <uri> <_>\" <status> <size> <_>"
```

**p99 latency from a duration field, last 5 minutes:**
```logql
quantile_over_time(0.99, {namespace="backend-prod"} | json | unwrap duration_seconds(duration) [5m])
```

---

## Running these

If you want these one-liners without opening the Grafana UI — tail/grep wrappers for
[`logcli`](https://grafana.com/docs/loki/latest/query/logcli/), schema-agnostic, installable in
one command — see [logcli-shortcuts](https://github.com/thisismairaj/logcli-shortcuts).

---

## Contributing

Corrections and additional recipes welcome — see [CONTRIBUTING.md](CONTRIBUTING.md).

---

## License

MIT
