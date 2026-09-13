---
title: "Zabbix at scale: why ClickHouse history storage matters"
---

A few years ago I helped run a Zabbix deployment monitoring around 7,000 hosts, on Kubernetes, because monitoring was critical infrastructure and downtime wasn't an option. Host count was never the hard part. The hard part was the data: roughly 2.5 TB a month of new history rows, and a housekeeper process that simply could not keep up with deleting them.

Zabbix 8.0 adds ClickHouse as an alternative history storage backend, and it directly targets that exact problem. Here's why it matters and what actually breaks at scale in the traditional setup.

## Where the bottleneck really is

Zabbix's history tables (numeric, text, log values, one row per item per poll) live in your SQL database, Postgres or MySQL, alongside all your configuration data. At a handful of hosts this is a non-issue. At thousands of hosts polled every 30 to 60 seconds, the history tables become by far the largest and busiest thing in that database, and they grow relentlessly.

The housekeeper's job is to delete rows older than your configured retention window. That's a straightforward idea that gets expensive fast in a row-oriented relational database:

- Deleting is a row-by-row (or batched-row) `DELETE`, which has to walk and update indexes, generate WAL/binlog, and in Postgres leaves dead tuples that only get reclaimed by `VACUUM`; in MySQL, InnoDB fragmentation accumulates similarly.
- That deletion work competes for the same disk I/O and locks that the constant stream of inserts needs.
- If your ingest rate is high enough, the housekeeper's delete throughput simply loses the race against new data coming in. It doesn't get a little behind, it gets permanently behind, and the gap only widens as data volume grows.

The practical symptom is exactly what we ran into: retention set to a reasonable window on paper, but disk usage that kept climbing regardless, because deletion could never fully catch up. The only real lever we had was to periodically drop and recreate the history tables to force an immediate reclaim, something we ended up doing about every six months. It worked, but it's a maintenance-window operation, not a fix. It's an admission that the housekeeper isn't a viable retention mechanism at that volume.

## Why ClickHouse actually fixes this, not just works around it

ClickHouse is a columnar database, and the compression alone helps: time-series values compress very well when stored column-by-column, so the same 2.5 TB/month of raw history can occupy a fraction of the space it would in a row store. But compression isn't the part that solves the housekeeper problem. Retention is.

ClickHouse's `MergeTree` tables (what Zabbix's ClickHouse schema uses) are partitioned by time, and expiry is handled by a `TTL` clause tied to that partitioning. When data ages out, ClickHouse doesn't scan for expired rows and delete them one by one. It drops entire partition files once every row in them has passed the TTL. Dropping a partition is close to a filesystem operation: it doesn't matter whether that partition holds a thousand rows or a hundred million, the cost is roughly the same, and it doesn't compete with the write path the way a `DELETE` does.

That's the actual fix. It's not "ClickHouse can hold more data," it's "ClickHouse's retention mechanism doesn't degrade as ingest volume grows," which is precisely the property a relational housekeeper doesn't have at scale.

Worth noting: Zabbix's own housekeeper explicitly does not manage ClickHouse data at all (this is documented, not a bug). Retention there is entirely ClickHouse's `TTL`/partition-drop mechanism, doing the one job it was actually built for. The trade-off is that trends are still calculated and stored only in the SQL database, and ClickHouse isn't supported as a proxy-side history backend, only on the server, both fine limitations given trend data is a much smaller, pre-aggregated dataset compared to raw history.

## Trying it hands-on

If you want to see the mechanics without standing up a 7,000-host environment, I put together a small Docker Compose reference that wires up Zabbix 8's ClickHouse history provider end to end, schema included, so you can watch history land in ClickHouse and inspect it directly.

**Repo:** [github.com/enderkus/zabbix8-clickhouse](https://github.com/enderkus/zabbix8-clickhouse)

It's a learning setup, not a production deployment guide, but it's a fast way to see the ClickHouse side of a Zabbix install before deciding whether it's worth migrating a real one.
