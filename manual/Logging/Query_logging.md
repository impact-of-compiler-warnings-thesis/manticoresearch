# Query logging


Query logging can be enabled by setting `query_log` directive in searchd section of the configuration file

```ini
searchd {
...
    query_log = /var/log/query.log
...
}
```
Queries can also be sent to syslog by setting `syslog` instead of a file path.

In this case all search queries will be sent to syslog daemon with `LOG_INFO` priority, prefixed with `[query]` instead of timestamp. Only `plain` log format is supported for syslog.


## Logging format

Two query log formats are supported:
* `plain`: recommended if most of your queries are purely full-text or you just don't care about non-full text components of your queries: filtering by attributes, sorting, grouping etc. You can't replay queries logged in `plain` format.
* `sphinxql`: recommended in all other cases (default). It also provides an easy way to replay logged queries.

### Plain log format

With `plain` log format Manticore logs all successfully executed search queries into a query log file. Here's an example:

```sql
[Fri Jun 29 21:17:58 2007] 0.004 sec 0.004 sec [all/0/rel 35254 (0,20)] [lj] test
[Fri Jun 29 21:20:34 2007] 0.024 sec 0.024 sec [all/0/rel 19886 (0,20) @channel_id] [lj] test
```

The log format is as follows:

```
[query-date] real-time wall-time [match-mode/filters-count/sort-mode total-matches (offset,limit) @groupby-attr] [table-name] query
```

* real-time is the time from start till finish of the query
* wall-time is like the real-time, excluding time spent on waiting for agents and merging result sets from them

`match-mode` can have one of the following values:

*   "all" for `SPH_MATCH_ALL` mode;
*   "any" for `SPH_MATCH_ANY` mode;
*   "phr" for `SPH_MATCH_PHRASE` mode;
*   "bool" for `SPH_MATCH_BOOLEAN` mode;
*   "ext" for `SPH_MATCH_EXTENDED` mode;
*   "ext2" for `SPH_MATCH_EXTENDED2` mode;
*   "scan" if the full scan mode was used, either by being specified with `SPH_MATCH_FULLSCAN` or if the query was empty.

`sort-mode` can have one of the following values:

*   "rel" for `SPH_SORT_RELEVANCE` mode;
*   "attr-" for `SPH_SORT_ATTR_DESC` mode;
*   "attr+" for `SPH_SORT_ATTR_ASC` mode;
*   "tsegs" for `SPH_SORT_TIME_SEGMENTS` mode;
*   "ext" for `SPH_SORT_EXTENDED` mode.

Note: the `SPH*` modes are specific to `sphinx` legacy interface. SQL and HTTP interfaces will log in most cases `ext2` as `match-mode` and `ext` and `rel` as `sort-mode`.

If Manticore was started with `--iostats` (ot it was enabled via `SET GLOBAL iostats=1`) the corresponding metrics will be included in the log. Then a query log entry might take the form of:

```
[Fri Jun 29 21:17:58 2021] 0.004 sec [all/0/rel 35254 (0,20)] [lj] [ios=6 kb=111.1 ms=0.5] test
```

where:
* `ios` - number of file I/O operations carried out;
* `kb` - amount of data in kilobytes read from the table files;
* `ioms` - time spent on I/O operations.


If Manticore was started with `--cpustats` (ot it was enabled via `SET GLOBAL cpustats=1`) metric `cpums` will be included in the log. The query log will then look so:

```
[Fri Jun 29 21:17:58 2021] 0.004 sec [all/0/rel 35254 (0,20)] [lj] [ios=6 kb=111.1 ms=0.5 cpums=0.3] test
```

where `cpums` is time in milliseconds spent on CPU processing the query.

### SQL log format

SQL format is default, but can be changed by searchd setting [query_log_format](../Server_settings/Searchd.md#query_log_format):

```ini
searchd {
...
    query_log = /var/log/query.log
    query_log_format = sphinxql
...
}
```
In this format, the example from the previous section would look as follows. (Wrapped below for readability, but with just one query per line in the actual log.)

```sql
/* Fri Jun 29 21:17:58.609 2007 2011 conn 2 real 0.004 wall 0.004 found 35254 */
SELECT * FROM test WHERE MATCH('test') OPTION ranker=proximity;

/* Fri Jun 29 21:20:34 2007.555 conn 3 real 0.024 wall 0.024 found 19886 */
SELECT * FROM test WHERE MATCH('test') GROUP BY channel_id OPTION ranker=proximity;
```

Note that **all** requests would be logged in this format, including those sent via SphinxAPI and SphinxSE, not just those sent via SQL. Also note, that this kind of logging works only with plain log files and will not work if you use 'syslog' service for logging.

The features of Manticore SQL log format compared to the plain text one are as follows:

* Full statement data will be logged where possible.
* Errors and warnings are logged.
* Query log can be replayedL.
* Additional performance counters (currently, per-agent distributed query times) are logged.

Use `sphinxql:compact_in` to shorten your `IN()` clauses in log if you have too many values in it.

Every request (including both SphinxAPI and SQL) request results in exactly one log line. Every log line must be a valid Manticore SQL statement that reconstructs the full request, except if the logged request is too big and needs shortening for performance reasons. Additional messages, counters, etc. can be logged in the comments section after the request.

## Logging only slow queries

By default all queries are logged. If it's desired to log only queries with execution times that exceed the specified limit, the `query_log_min_msec` directive can be used:

```ini
searchd {
...
    query_log = /var/log/query.log
    query_log_min_msec  = 1000
...
}
```

The expected unit of measurerement is milliseconds, but time suffix expressions can be used as well, like:

```ini
searchd {
...
    query_log = /var/log/query.log
    query_log_min_msec  = 1s
...
}
```

## Log file permission mode

By default the searchd and query log files are created with permission `600`, so only the user under which Manticore is running and `root` can read the log files. `query_log_mode` allows setting a different permission. This can be handy to allow other users to be able to read the log files (for example monitoring solutions running on non-root users).


```ini
searchd {
...
    query_log = /var/log/query.log
    query_log_mode = 666
...
}
```
