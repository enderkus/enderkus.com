---
title: "Zabbix at scale: why ClickHouse history storage matters"
---

At a previous company I helped run a Zabbix deployment monitoring around 7,000 hosts, a place where monitoring was not a side project, it was the platform every engineering team relied on to know whether anything was on fire. We ran it on Kubernetes specifically so the monitoring layer itself could self-heal, because if Zabbix went down, every other team went blind at exactly the moment they'd need visibility most. Downtime there wasn't an inconvenience, it was the one failure mode we designed everything else around avoiding.

Host count was never the hard part of running that platform. The hard part, the thing that actually kept me up, was the data: roughly 2.5 TB a month of new history rows, and a housekeeper process that simply could not keep up with deleting them.

## The real cost of falling behind

Here's what that looked like in practice. We had a retention window configured, a perfectly reasonable one on paper. But disk usage kept climbing past what that window should have implied, month after month, because the housekeeper's deletes never fully caught up with the inserts. That's a genuinely uncomfortable position to be in on a monitoring system: the one database you cannot afford to run out of disk on, slowly filling up anyway, with no setting you can tune to make the cleanup process go faster than the data coming in.

The only lever that actually worked was blunt: periodically drop and recreate the history tables to force an immediate reclaim. We ended up doing that roughly every six months. In practice that meant scheduling a maintenance window on a production system every team depended on for alerting, and accepting that we were throwing away months of historical data we might have wanted for capacity planning, just to buy back disk space the housekeeper should have been reclaiming on its own. It worked, but it was an operational tax we paid on a schedule, not a fix, and it was a quiet admission that the housekeeper simply isn't a viable retention mechanism once you're ingesting at that volume.

Zabbix 8.0 adds ClickHouse as an alternative history storage backend, and it speaks directly to this problem: not host count, the retention mechanism itself, plus a query speed benefit on top.

## Where the bottleneck really is

Zabbix's history tables (numeric, text, log values, one row per item per poll) live in your SQL database, Postgres or MySQL, alongside all your configuration data. At a handful of hosts this is a non-issue. At thousands of hosts polled every 30 to 60 seconds, the history tables become by far the largest and busiest thing in that database, and they grow relentlessly.

The housekeeper's job is to delete rows older than your configured retention window. That's a straightforward idea that gets expensive fast in a row-oriented relational database:

- Deleting is a row-by-row (or batched-row) `DELETE`, which has to walk and update indexes, generate WAL/binlog, and in Postgres leaves dead tuples that only get reclaimed by `VACUUM`; in MySQL, InnoDB fragmentation accumulates similarly.
- That deletion work competes for the same disk I/O and locks that the constant stream of inserts needs.
- If your ingest rate is high enough, the housekeeper's delete throughput simply loses the race against new data coming in. It doesn't get a little behind, it gets permanently behind, and the gap only widens as data volume grows.

## Why ClickHouse actually fixes this, not just works around it

ClickHouse is a columnar database, and the compression alone helps: time-series values compress very well when stored column-by-column, so the same 2.5 TB/month of raw history can occupy a fraction of the space it would in a row store. But compression isn't the part that would have solved our actual problem. Retention is.

ClickHouse's `MergeTree` tables (what Zabbix's ClickHouse schema uses) are partitioned by time, and expiry is handled by a `TTL` clause tied to that partitioning. When data ages out, ClickHouse doesn't scan for expired rows and delete them one by one. It drops entire partition files once every row in them has passed the TTL. Dropping a partition is close to a filesystem operation: it doesn't matter whether that partition holds a thousand rows or a hundred million, the cost is roughly the same, and it doesn't compete with the write path the way a `DELETE` does.

That's the actual fix for the retention side. Not "ClickHouse can hold more data," but "ClickHouse's retention mechanism doesn't degrade as ingest volume grows," which is precisely the property our relational housekeeper never had. No twice-a-year maintenance window, no gambling on disk space, no throwing away history just to buy back room for more history.

There's a second benefit worth calling out on its own: query speed. Zabbix's graphs and dashboards mostly ask exactly the kind of question ClickHouse is built to answer fast, filter by item, filter by a time range, aggregate. A columnar, vectorized engine scanning that pattern over billions of rows should noticeably outperform the same query against a row-oriented table carrying the same volume, especially once that table has grown large enough that its indexes stop being cheap either.

Worth noting: Zabbix's own housekeeper explicitly does not manage ClickHouse data at all (this is documented, not a bug). Retention there is entirely ClickHouse's `TTL`/partition-drop mechanism, doing the one job it was actually built for. The trade-off is that trends are still calculated and stored only in the SQL database, and ClickHouse isn't supported as a proxy-side history backend, only on the server, both fine limitations given trend data is a much smaller, pre-aggregated dataset compared to raw history.

I'll be upfront that I haven't run this specific setup in production myself. Zabbix 8 isn't even LTS yet, this is a brand new, not-yet-battle-tested feature, and the 7,000-host story above predates it entirely, that problem got solved the old, blunt way, with a recurring table rebuild. But having lived through exactly the failure mode this history provider targets, I think it's one of the more significant additions in this release. It doesn't just add a new storage option, it replaces the specific mechanism, row-by-row deletion under sustained high-volume writes, that was the actual root cause.

## Trying it hands-on

If you want to see the mechanics without standing up a 7,000-host environment, I put together a small Docker Compose reference that wires up Zabbix 8's ClickHouse history provider end to end, schema included, so you can watch history land in ClickHouse and inspect it directly.

**Repo:** [github.com/enderkus/zabbix8-clickhouse](https://github.com/enderkus/zabbix8-clickhouse)

It's a learning setup, not a production deployment guide, but it's a fast way to see the ClickHouse side of a Zabbix install before deciding whether it's worth migrating a real one.
