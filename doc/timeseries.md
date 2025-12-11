# Forked from Tembo Time-Series API

The purpose of this extension is to provide a cohesive user experience around the creation, maintenance, and use of time-series tables.

## Installation

## Getting Started

Assuming you already have a partitioned table created, simply call the `enable_ts_table` function with your table name.

```sql
SELECT enable_ts_table('sensor_readings');
```

With this one call, several things will happen:

  * The table will be restructured as a series of partitions using PostgreSQL's [native PARTITION features](https://www.postgresql.org/docs/current/ddl-partitioning.html)
  * Each partition covers a particular range of time (one week by default)
  * New partitions will be created for some time in the future (one month by default)
  * Once an hour, a maintenance job will create any missing partitions as well as needed future ones

## Using your tables

So you've got a table. Now what?

### Indexes

The time-series tables you create start out life as little more than typical [partitioned PostgreSQL tables](https://www.postgresql.org/docs/current/ddl-partitioning.html). But this simplicity also means all of PostgreSQL's existing functionality will "just work" with them. A fairly important piece of a time-series table is an index along the time dimension.

[Traditional B-Tree indexes](https://www.postgresql.org/docs/current/btree-intro.html) work well for time-series data, but you may wish to benchmark [BRIN indexes](https://www.postgresql.org/docs/current/brin-intro.html) as well, as they may perform better in specific query scenarios (often queries with _many_ results). Start with B-Tree if you don't anticipate more than a million records in each partition (by default, partitions are one week long).

### Partition Sizing

Related to the above information on indexes is the question of partition size. Because calculating the total size of partitioned tables can be tedious, this extension provides several easy-to-use views surfacing this information.

To examine the table (data), index, and total size for each of your partitions, simple query the time-series partition information view, `ts_part_info`. A general rule of thumb is that each partition should be able to fit within roughly one quarter of your available memory. This assumes that not much apart from the time-series workload is going on, and things like parallel workers may complicate matters, but work on getting partition total size down to around a quarter of your memory and you're off to a good start.

### Retention

On the other hand, you may be worried about plugging a firehose of data into your storage layer to begin with… While the `ts_table_info` view may allay your fears, at some point you _will_ want to remove some of your time-series data.

Fortunately, it's incredibly easy to simply drop time-series partitions on a schedule. Call `set_ts_retention_policy` with your time-series table and an interval (say, `'90 days'`) to establish such a policy. Once an hour, any partitions falling entirely outside the retention window will be dropped. Use `clear_ts_retention_policy` to revert to the default behavior (infinite retention). Each of these functions will return the previous retention policy when called.

### Compression

Compression options still be configured to maintain backwards compatibility, but no longer applies and the `apply_compression_policy` function has been stubbed out.

### Analytics Helpers

This extension includes several functions intended to make writing correct time-series queries easier. Certain concepts can be difficult to express in standard SQL and helper functions can aid in readability and maintainability.

#### `first` and `last`

These two functions help clean up the syntax of a fairly common pattern: a query is grouped by one dimension, but a user wants to know what the first or last row in a group is when ordered by a _different_ dimension.

For instance, you might have a cloud computing platform reporting metrics and wish to know the latest (in time) CPU utilization metric for each machine in the platform:

```sql
SELECT machine_id,
       last(cpu_util, recorded_at)
FROM events
GROUP BY machine_id;
```

#### `date_bin_table`

This function automates the tedium of aligning time-series values to a given width, or "stride", and makes sure to include NULL rows for any time periods where the source table has no data points.

It must be called against a time-series table, but apart from that consideration using it is pretty straightforward:

```sql
SELECT * FROM date_bin_table(NULL::target_table, '1 hour', '[2024-02-01 00:00, 2024-02-02 15:00]');
```

The output of this query will differ from simply hitting the target table directly in three ways:

  * Rows will be sorted by time, ascending
  * The time column's values will be binned to the provided width
  * Extra rows will be added for periods with no data. They will include the time stamp for that bin and NULL in all other columns

## Requirements

The `pg_timeseries` extension depends on other extensions:

* [pg_cron](https://github.com/citusdata/pg_cron)
* [pg_partman](https://github.com/pgpartman/pg_partman)

We recommend referring to documentation within these projects for more advanced use cases, or for a better understanding at how this extension works.

## Roadmap

This fork of `pg_timeseries` is primarily geared towards the [OpenNMS pg_timeseries plugin](https://github.com/dino2gnt/timeseries-integration-pgtimeseries), and getting the extension buildable again. 

  - Remove the requirement for the abandoned `columnar` extension
  - Remove the requirement for the abandoned Tembo fork of `pg_ivm`
  - Make it buildable without these requirements
  - Make sure it works with all currently supported releases of PostgreSQL
