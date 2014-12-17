---
layout: documentation
title: System tables
active: docs
docs_active: system-tables
permalink: docs/system-tables/
---

Starting with version 1.16, RethinkDB maintains special *system tables* that contain configuration and status information about servers, databases, individual tables, and issues with the cluster. Querying system tables returns information about the status of the cluster and current objects (such as servers and tables) within the cluster. By inserting or deleting records and updating fields in these tables, the configuration of the objects they represent can be modified.

# Overview #

Access the system tables through the `rethinkdb` database. This database is automatically created on cluster startup, and system tables cannot be created, dropped, reconfigured, or renamed. These tables aren't real RethinkDB document stores the way user-created tables are, but rather "table-like" interfaces to the system allowing most ReQL commands to be used for control.

The metadata in the system tables applies to the RethinkDB cluster as a whole. Each server in a cluster maintains its own copy of the system tables. Whenever a system table on a server changes, the changes are synced across all the servers.

## The Tables ##

* `table_config` stores table configurations, including sharding and replication. By writing to `table_config`, you can create, delete, and reconfigure tables.
* `server_config` stores server names and tags. By writing to this table you can rename servers, assign them tags, and remove them from the cluster entirely.
* `db_config` stores database UUIDs and names. By writing to this table, databases can be created, deleted or modified.
* `cluster_config` stores the authentication key for the cluster.
* `table_status` is a read-only table which returns the  status and configuration of tables in the system.
* `server_status` is a read-only table that returns the status of the servers, including uptime, host name, and process ID.
* `issues` is a read-only table that returns statistics about cluster problems. For details, read the [System issues table][sit] documentation.
* `jobs` lists the jobs&mdash;queries, index creation, disk compaction, and other utility tasks&mdash;the cluster is spending time on, and also allows you to interrupt running queries.
* `stats` is a read-only table that returns statistics about the cluster.
* `logs` contains log files for the cluster.

[sit]: /docs/system-issues/

## Caveats ##

* While system tables support changefeeds, they do not support all of the chaining that real tables do. For instance, aggregation (`max` and `min`) and `limit` commands will not work with system tables.
* Some system tables are read-only. System tables which allow writing require specific document schema, described below.
* Write operations on system tables are non-atomic. Avoid writing to the same system table row from more than one client at the same time.
* The `durability` argument on writes is ignored for system tables.

With system tables only, the `table` command takes a new argument, `identifier_format`. Legal values are `name` and `uuid`. When it's set to `uuid`, references in system tables to databases or other tables will be UUIDs rather than database/table names. This is useful for writing scripts and administration tasks, as UUIDs remain consistent even if object names change. The default is `name`.

# Configuration tables #

## table_config ##

Sharding and replication can be controlled through the `table_config` table, along with the more advanced settings of write acknowledgements and durability. Tables can also be renamed by modifying their rows. A typical row in the `table_config` table will look like this:

```js
{
    id: "31c92680-f70c-4a4b-a49e-b238eb12c023",
    name: "tablename",
    db: "test",
    primary_key: "id",
    shards: [
        {primary_replica: "a", "replicas": ["a", "b"]},
        {primary_replica: "b", "replicas": ["a", "b"]}
        ],
    write_acks: "majority",
    durability: "hard"
}
```

* `id`: the UUID of the table. (Read-only.)
* `name`: the name of the table.
* `db`: the database the table is in, either a name or UUID depending on the value of `identifier_format`. It cannot be modified.
* `primary_key`: the name of the field used as the primary key of the table, set at table creation. (Read-only.)
* `shards`: a list of the table's shards. Each shard is an object with these fields:
	* `primary_replica`: the name or UUID of the server acting as the shard's primary. If `primary_replica` is `null` the table will be unavailable. (This may happen if the server acting as the shard's primary is deleted.)
	* `replicas`: a list of servers, including the director, storing replicas of the shard.
* `write_acks`: the write acknowledgement settings for the table. When set to `majority` (the default), writes will be acknowledged when a majority of replicas have acknowledged their writes; when set to `single` writes will be acknowledged when a single replica acknowledges it. This may also be set to a list of requirements; see below.
* `durability`: `soft` or `hard` (the default). In `hard` durability mode, writes are committed to disk before acknowledgements are sent; in `soft` mode, writes are acknowledged immediately upon receipt. The `soft` mode is faster but slightly less resilient to failure.

If `write_acks` is set to a list of requirements, they should take the form of:

```js
[
	{replicas: ['a', 'b'], acks: 'soft'},
	{replicas: ['c', 'd'], acks: 'hard'},
	...
]
```

If you `delete` a row from `table_config` the table will be deleted. If you `insert` a row, you must include the `name` and `db` fields; the rest will be automatically generated or set to default. You must *not* include the `id` field--the system will auto-generate a UUID.

If you `replace` a row in `table_config`, you must include all the fields. It's usually easier to `update` specific fields.

Native ReQL commands like `reconfigure` also control sharding and replication, and if you're not using server tags you can change sharding/replication settings in the web UI. Read [Sharding and replication][shrep] for more details.

[shrep]: /docs/sharding-and-replication/

## server_config ##

This table stores the names of servers along with their *tags.* Server tags organize servers into logical groups: servers could be tagged by usage (database, application, etc.), or by data center location ("us_west," "us_east," "london," and so on). For more about server tags, read [Sharding and replication][shrep].

Every server that has ever been part of the cluster--and has not been permanently removed--will have a row in this table in the following format.

```js
{
    id: "de8b75d1-3184-48f0-b1ef-99a9c04e2be5",
    name: "servername",
    tags: ["default"]
}
```

* `id`: the UUID of the server. (Read-only.)
* `name`: the server's name.
* `tags`: a list of unordered tags associated with the server.

If tags aren't specified when a server starts, the server is automatically assigned the `default` tag. Documents cannot be inserted into `server_config`. A new document gets created when a server connects to the cluster.

Documents *can* be deleted from this table--doing so permanently removes the server from the cluster. When a server is permanently removed from a cluster, the following occurs:

* all references to it are deleted from `table_config`;
* if it was a replica for a shard, that entry is removed from the shard's `replicas` list;
* if it was the primary replica for a shard, the `primary_replica` field in that shard's entry becomes `null`;
* if it was in a list of replicas in the `write_acks` field, it's removed from that list.

Permanently deleting a server is *irreversible:* a deleted server cannot rejoin the cluster, even by restarting the RethinkDB process on that server. You can make a server "new" again by entirely deleting that server's RethinkDB data directory.

## db_config ##

One document exists in `db_config` for each database in the cluster, with only two fields in the document.

```js
{
    id: "de8b75d1-3184-48f0-b1ef-99a9c04e2be5",
    name: "dbname"
}
```

* `id`: the UUID of the database. (Read-only.)
* `name`: the name of the database.

Documents can be inserted to create new databases, deleted to remove databases, and modified to rename databases. (Renaming databases is the only task that requires querying the `db_config` table; the other two tasks have native ReQL commands, [dbCreate][dbc] and [dbDrop][dbd].) As with tables, if you `insert` a database, don't include the `id` field: the system will auto-generate the UUID.

[dbc]: /api/javascript/db_create
[dbd]: /api/javascript/db_drop

## cluster_config ##

The `cluster_config` table is even simpler than `db_config`; only one thing can be changed, the [authentication key][auth]. This table always has exactly one row. Documents cannot be inserted into or deleted from this table.

[auth]: /docs/security/

```js
{
    id: "auth",
    auth_key: null
}
```

* `id`: the primary key, always `auth`.
* `auth_key`: the authentication key, or `null` if no key is set.

Updating the `auth_key` field is the only way to set or change the cluster authentication key. Read [Securing your cluster][auth] to learn about the key and why you might want to set it.

The `auth_key` field is unusual in that it is a *write-only* field. If you try to read its value, you will get `null` or `{hidden: true}` but will not see the actual key.

# Status tables #

All the status tables are read-only. Some of the information in status tables is also returned in config tables (such as object names and UUIDs).

## table_status ##

This table stores information about table availability. There is one document per table (not counting system tables).

```js
{
    id: "31c92680-f70c-4a4b-a49e-b238eb12c023",
    name: "tablename",
    db: "test",
    status: {
        ready_for_outdated_reads: true,
        ready_for_reads: true,
        ready_for_writes: true,
        all_replicas_ready: true
    },
    shards: [
        {
            primary_replica: "a",
            replicas: [{server: "a", state: "ready"}, {server: "b", state: "ready"}]
        },
        {
            primary_replica: "b",
            replicas: [{server: "a", state: "ready"}, {server: "b", state: "ready"}]
        }]
}

```

* `id`: the UUID of the table.
* `name`: the table's name.
* `db`: the database the table is in, either a name or UUID depending on the value of `identifier_format` (see "caveats" in the overview at the top of this document).
* `status`: the subfields in this field indicate whether all shards of the table are ready to accept the given type of query: `outdated_reads`, `reads` and `writes`. The `all_replicas_ready` field indicates whether all backfills have finished.
* `shards`: one entry for each shard in `table_config`. Each shard's object has the following fields:
	* `primary_replica`: the name of the shard's primary or `null` if one is not set.
	* `replicas`: a list of all servers acting as a replica for that shard. This may include servers which are no longer configured as replicas but are still storing data. The `state` field may be one of the following:
		* `missing`: the server is not connected to the cluster.
		* `backfilling_data`: the server is receiving data from another server.
		* `offloading_data`: the server is waiting for all other servers to report `ready` so it can erase its data.
		* `erasing_data`: the server is erasing its data.
		* `looking_for_primary`: the server is waiting for its primary replica to be available.
		* `ready`: the server is ready to serve queries.
		* `transitioning`: the server is between one of the above states. A transitioning state should typically only last a fraction of a second.

## server_status ##

This table returns information about the status and availability of servers within a RethinkDB cluster. There is one document per server that's ever been connected to the cluster and not permanently removed.

This is a typical document schema for a server connected to the host server--that is, the server the client's connecting to when they query the `server_status` table. If a server is *not* connected to the host server there will be differences noted below.

```js
{
    id: "de8b75d1-3184-48f0-b1ef-99a9c04e2be5",
    name: "servername",
    status: "available"
    connection: {
        time_connected: <ReQL time object>,
        time_disconnected: null
    },
    network: {
        hostname: "companion-cube",
        cluster_port: 29015,
        http_admin_port: 8080,
        reql_port: 28015,
        canonical_addresses: [
            { host: "127.0.0.1", port: 29015 },
            { host: "::1", port: 29015 }
            ]
    },
    process: {
        cache_size_mb: 1882.30078125,
        pid: 28580,
        time_started: <ReQL time object>,
        version: "rethinkdb 1.16.0-xxx (CLANG 3.4 (tags/RELEASE_34/final))"
    },
}
```

* `id`: the UUID of the server.
* `name`: the name of the server.
* `status`: always `available` if the server is connected.
* `connection`: two timestamp fields:
	* `time_connected`: the time the server connected to the host server. 
	* `time_disconnected`: always `null` if the server is connected.
* `network`: information about the network the server is on:
	* `hostname`: the host name as returned by `gethostname()`.
	* `*_port`: the RethinkDB ports on that server (from the server's own point of view).
	* `canonical_addresses`: a list of the canonical addresses and ports of the server. These may differ from `hostname` and `cluster_port` depending on your network configuration.
* `process`: information about the RethinkDB server process:
	* `cache_size_mb`: the cache size in megabytes. (This can be [configured on startup][startup].)
	* `pid`: the process ID.
	* `time_started`: the time the server process started.
	* `version`: the version string of the RethinkDB server.

[startup]: /docs/cluster-on-startup/

If the server is not connected to the host server:

* `status` becomes `unreachable`;
* `time_connected` becomes `null`;
* If the server was connected to the host server since the last time the host server restarted, `time_disconnected` becomes the time that the server was last seen. If not, it will be `null`;
* All the fields in `network` and `process` become `null`. (The field names are still present, but the values are `null`.)

# Other tables #

## stats ##

The `stats` table provides statistics about server read/write throughput, client connections, and memory usage. For details, read the [System stats table][sst] documentation.

[sst]: /docs/system-stats/

## jobs ##

## issues ##

This table shows problems that have been detected within the RethinkDB cluster. For details, read the [System issues table][sit] documentation.
