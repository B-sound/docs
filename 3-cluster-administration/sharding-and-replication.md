---
layout: documentation
title: Sharding and replication
active: docs
docs_active: sharding-and-replication
permalink: docs/sharding-and-replication/
---

<img alt="Sharding and Replication Illustration" class="api_command_illustration"
    src="/assets/images/docs/api_illustrations/shard-and-replicate.png" />

RethinkDB allows you to shard and replicate your cluster on a per-table basis. Settings can be controlled easily from the web administration console. In addition, ReQL commands for table configuration allow both scripting capability and more fine-grained control over replication, distributing replicas for individual tables across user-defined groups of servers using server tags.

{% infobox info %}

__Note__: Currently, RethinkDB implements range shards, but will eventually be
switching to hash shards. Follow [Github issue #364][gh364] to track progress.

[gh364]: https://github.com/rethinkdb/rethinkdb/issues/364

{% endinfobox %}

# Sharding and replication via the web console #

When using the web UI, simply specify the number of shards you want, and based on the data available RethinkDB will determine the best split points to maintain balanced shards. To shard your data:

- Go to the table view (_Tables_ &rarr; _table name_).
- Set the number of shards and replicas you would like.
- Click on the _Apply_ button.

![Shard with the web interface](/assets/images/docs/administration/shard.png)

# Sharding and replication via ReQL #

There are three primary commands for changing sharding and replication in ReQL. In addition, there are lower-level values that can be changed by manipulating [system tables](/docs/system-tables/).


* The [table_create](/api/python/table_create) (or [tableCreate](/api/javascript/table_create)) command can specify initial values for `shards` and `replicas`.
* The [reconfigure](/api/python/reconfigure) command can change the values for `shards` and `replicas` for an existing table.
* The [rebalance](/api/python/rebalance) command will rebalance table shards.

For more information about administration via ReQL, consult the API documentation for the individual commands as well as the [Administration tools][at] documentation.

[at]: /docs/administration-tools/

# Advanced configuration #

These tasks cannot be performed through the web interface.

## Server tags ##

All of the servers in a RethinkDB cluster may be given zero or more _tags_ that can be used in table configurations to map replicas to servers specified by tag.

A server can be given tags with the `--server_tags` option on startup:

```
rethinkdb --server_tags us,us_west
```


While running, a server's configuration can be changed by writing to the `rethinkdb.server_config` [system table](/docs/system-tables/).

```py
# get server by UUID
r.db('rethinkdb').table('server_config').get(
    'd5211b11-9824-47b1-9f2e-516a999a6451').update(
    {tags: ['default', 'us', 'us_west']}.run(conn)
```

If no tags are specified on startup, the server will be started with one tag, `default`. Changing the sharding/replica information from the web UI or from ReQL commands that do not specify server tags will affect all servers with the `default` tag. This means that if you remove the `default` tag from a server, or start it without that tag, it wil not be affected by changes from the web UI. 

When servers are tagged, you can use the tags in the [reconfigure](/api/python/reconfigure) command. To assign 3 replicas of the `users` table to `us_west` and 2 to `us_east`:

```py
r.table('users').reconfigure(shards=2, replicas={'us_west':3, 
    'us_east':2}).run(conn)
```

If you remove *all* of a server's tags and then reconfigure all the cluster's tables, that server will be taken out of service.

```py
# decommission a server
r.db('rethinkdb').table('server_config').get(
    'd5211b11-9824-47b1-9f2e-516a999a6451').update(
    {tags: []}.run(conn)
r.db('database').reconfigure(shards=2, replicas=3).run(conn)
```

Note that tables are configured on startup and when the `reconfigure` command is called, but the configurations are *not* stored by the server otherwise. To reconfigure tables consistently&mdash;especially if your configuration uses server tags&mdash;you should save the configuration in a script. Read more about this in [Administration tools][at].

## Write acks and durability ##

Two settings for tables, write acknowledgements and write durability, cannot be set through either the web interface or the `reconfigure` command. They must be set by modifying the `table_config` table for individual tables.

The write acknowledgement setting for a table controls when the cluster acknowledges a write request as fulfilled. There are three possible settings:

* `majority`: The cluster sends the acknowledgement when the majority of replicas have acknowledged it. This is the default.
* `single`: The cluster sends the acknowledgement when any replica has acknowledged it.
* complex: a list of requirements describing sets of replicas with either `majority` or `single` write acknowledgements. This takes the form of `[{replicas: ["replica1", "replica2", ...], acks: "single"}, {replicas: ["replica3", "replica4", ...], acks: "majority"}]`.

To change these settings for a table:

```py
r.db('rethinkdb').table('table_config').get(
    '31c92680-f70c-4a4b-a49e-b238eb12c023').update(
        {"write_acks": "single"}).run(conn)
```
