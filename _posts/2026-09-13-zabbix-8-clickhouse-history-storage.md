---
title: "Zabbix 8 and ClickHouse: wiring up the new history storage backend"
---

Zabbix 8.0 ships a feature I'd been waiting for: **ClickHouse as a history storage provider**. Configuration data (hosts, items, triggers, users) still lives in your regular SQL database, but the high-volume history rows, the numeric, text, and log values that pile up every few seconds per item, can now be written straight to ClickHouse instead.

I spent an evening setting this up in Docker to see how it behaves in practice, hit a genuinely interesting bug along the way, and turned the whole thing into a runnable demo repo. This post covers both.

## Why bother with ClickHouse for this

Zabbix has always stored history in PostgreSQL or MySQL. That works, but it's a row-oriented OLTP database being asked to do an OLAP job: ingest a constant stream of timestamped values and later answer range queries like "give me every value for this item between these two timestamps" across potentially billions of rows.

ClickHouse is a columnar database built for exactly that pattern:

- **Compression.** Each column, itemid, timestamp, value, is stored and compressed separately. Time-series values tend to be similar to their neighbors, which compresses very well. You can keep months of granular history in a fraction of the disk space a row-oriented table would need.
- **Fast range and aggregate scans.** Zabbix's own graphs and dashboards mostly issue exactly the query pattern ClickHouse is optimized for: filter by itemid, filter by a time window, aggregate. It doesn't need OLTP-style point-lookup indexes for that.
- **It takes load off Postgres.** Once you have enough hosts and low-enough polling intervals, history writes are usually the first thing that stresses a PostgreSQL-backed Zabbix install. Moving them to a purpose-built store lets your config database breathe and lets history scale independently.

The official docs are also refreshingly upfront about the trade-offs: the Zabbix housekeeper does not clean up ClickHouse data (retention is controlled by ClickHouse's own TTL clause instead), trends are still calculated and stored only in the SQL database, and ClickHouse isn't supported as a proxy-side history backend, only on the server. Worth knowing before you commit to it.

## The setup: Docker Compose, three agents, one bug

I built a stack with Postgres for config, a ClickHouse container for history, the Zabbix server and frontend, and three agents: one representing the built-in "Zabbix server" host (self-monitoring), and two throwaway "test-agent" containers to generate sample data.

I brought it up, and two of the three hosts came online immediately. The one that didn't was the one you'd least expect to have trouble: **"Zabbix server" itself**. The frontend just showed it as unavailable, no data coming in for its own health metrics, while both test agents reported fine.

The logs gave it away:

```
temporarily disabling Zabbix agent checks on host "Zabbix server": interface unavailable
```

Zabbix ships the built-in "Zabbix server" host with its agent interface hardcoded to `127.0.0.1:10050`. That's a sane default when the agent runs on the same machine as the server process, which is the traditional single-box install. But in this Docker Compose setup, the agent for that host is its own container, on its own IP, not sharing a network namespace with `zabbix-server`. So `zabbix-server` dutifully checked its own loopback interface, found nothing listening, and marked the host unreachable. The two test agents were fine because they'd been defined from the start with proper DNS-based interfaces pointing at their own containers, not `127.0.0.1`.

The fix is a one-line correction: point that interface at the agent container's DNS name on the compose network instead of `127.0.0.1`. Once I did that, "Zabbix server" came online within one polling cycle, right alongside the other two.

## Making it reproducible

A manual database patch isn't something I wanted to hand anyone else. So I turned the fix into a small init container that talks to the Zabbix API on first boot: it waits for the API to come up, logs in, checks whether the "Zabbix server" host interface still points at `127.0.0.1`, and repoints it at the agent container if so. It also registers the two test-agent hosts automatically, so a fresh checkout needs zero manual clicking in the UI. Both steps are idempotent, safe to run on every `docker compose up`.

I bundled all of it, Postgres, ClickHouse, the schema-creation script, the Zabbix stack, and this provisioning step, into one Docker Compose file. Clone it, run `docker compose up -d`, wait about a minute, and you have a fully monitored Zabbix 8 install writing history into ClickHouse, with a Tabix UI included for poking at the ClickHouse data directly.

**Repo:** [github.com/enderkus/zabbix8-clickhouse](https://github.com/enderkus/zabbix8-clickhouse)

The README covers the architecture, the exact ClickHouse schema (mirroring [Zabbix's official setup docs](https://www.zabbix.com/documentation/8.0/en/manual/appendix/install/clickhouse_setup)), and every configuration knob. It's a learning and demo setup, not a production hardening guide: default passwords, no TLS, single-node ClickHouse. Treat it as a starting point for understanding how the pieces fit together, then harden before you go anywhere near production.
