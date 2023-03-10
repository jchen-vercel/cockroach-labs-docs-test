---
title: SHOW TRACE
summary: The SHOW TRACE statement...
toc: true
---

<span class="version-tag">New in v1.1:</span> The `SHOW TRACE` [statement](sql-statements.html) returns details about how CockroachDB executed a statement or series of statements. These details include messages and timing information from all nodes involved in the execution, providing visibility into the actions taken by CockroachDB across all of its software layers.

You can use `SHOW TRACE` to debug why a query is not performing as expected, to add more information to bug reports, or to generally learn more about how CockroachDB works.


## Usage Overview

There are two distinct ways to use `SHOW TRACE`:

Statement | Usage
----------|------
[`SHOW TRACE FOR <stmt>`](#show-trace-for-stmt) | Execute a single [explainable](sql-grammar.html#explainable_stmt) statement and return a trace of its actions.
[`SHOW TRACE FOR SESSION`](#show-trace-for-session) | Return a trace of all executed statements recorded during a session.

### `SHOW TRACE FOR <stmt>`

This use of `SHOW TRACE` executes a single [explainable](sql-grammar.html#explainable_stmt) statement and then returns messages and timing information from all nodes involved in its execution. It's important to note the following:

- `SHOW TRACE FOR <stmt>` executes the target statement and, once execution has completed, then returns a trace of the actions taken. For example, tracing an `INSERT` statement inserts data and then returns a trace, a `DELETE` statement deletes data and then returns a trace, etc. This is different than the [`EXPLAIN`](explain.html) statement, which does not execute its target statement but instead returns details about its predicted execution plan.
    - The target statement must be an [`explainable`](sql-grammar.html#explainable_stmt) statement. All non-explainable statements are not supported.
    - The target statement is always executed with the local SQL engine. Due to this [known limitation](https://github.com/cockroachdb/cockroach/issues/16562), the trace will not reflect the way in which some statements would have been executed when not the target of `SHOW TRACE FOR <stmt>`. This limitation does not apply to `SHOW TRACE FOR SESSION`.

- If the target statement encounters errors, those errors are not returned to the client. Instead, they are included in the trace. This has the following important implications for [transaction retries](transactions.html#transaction-retries):
    - Normally, individual statements (considered implicit transactions) and multi-statement transactions sent as a single batch are [automatically retried](transactions.html#automatic-retries) by CockroachDB when [retryable errors](transactions.html#error-handling) are encountered due to contention. However, when such statements are the target of `SHOW TRACE FOR <stmt>`, CockroachDB does **not** automatically retry.
    - When each statement in a multi-statement transaction is sent individually (as opposed to being batched), if one of the statements is the target or `SHOW TRACE <stmt>`, retryable errors encountered by that statement will not be returned to the client.

    {{site.data.alerts.callout_success}}Given these implications, when you expect transaction retries or want to trace across retries, it's recommended to use <code>SHOW TRACE FOR SESSION</code>.{{site.data.alerts.end}}

### `SHOW TRACE FOR SESSION`

This use of `SHOW TRACE` returns messages and timing information for all statements recorded during a session. It's important to note the following:

- `SHOW TRACE FOR SESSION` only returns the most recently recorded traces, or for a currently active recording of traces.
    - To start recording traces during a session, enable the `tracing` session variable via [`SET tracing = on;`](set-vars.html#set-tracing).
    - To stop recording traces during a session, disable the `tracing` session variable via [`SET tracing = off;`](set-vars.html#set-tracing).

- In contrast to `SHOW TRACE FOR <stmt>`, recording traces during a session does not effect the execution of any statements traced. This means that errors encountered by statements during a recording are returned to clients. CockroachDB will [automatically retry](transactions.html#automatic-retries) individual statements (considered implicit transactions) and multi-statement transactions sent as a single batch when [retryable errors](transactions.html#error-handling) are encountered due to contention. Also, clients will receive retryable errors required to handle [client-side retries](transactions.html#client-side-intervention). As a result, traces of all transaction retries will be captured during a recording.

- `SHOW TRACE FOR <stmt>` overwrites the last recorded trace. This means that if you enable session recording, disable session recording, execute `SHOW TRACE FOR <stmt>`, and then execute `SHOW TRACE FOR SESSION`, the response will be the trace for `SHOW TRACE FOR <stmt>`, not for the previously recorded session.

## Required Privileges

For `SHOW TRACE FOR <stmt>`, the user must have the appropriate [privileges](privileges.html) for the statement being traced. For `SHOW TRACE FOR SESSION`, no privileges are required.

## Syntax

<section>{% include {{ page.version.version }}/sql/diagrams/show_trace.html %}</section>

## Parameters

Parameter | Description
----------|------------
`KV` | If specified, the returned messages are restricted to those describing requests to and responses from the underlying key-value [storage layer](architecture/storage-layer.html), including per-result-row messages.<br><br>For `SHOW KV TRACE FOR <stmt>`, per-result-row messages are included.<br><br>For `SHOW KV TRACE FOR SESSION`, per-result-row messages are included only if the session was/is recording with `SET tracing = kv;`.
`explainable_stmt` | The statement to execute and trace. Only [explainable](sql-grammar.html#explainable_stmt) statements are supported.

## Trace Description

CockroachDB's definition of a "trace" is a specialization of [OpenTracing's](https://opentracing.io/docs/overview/what-is-tracing/#what-is-opentracing) definition. Internally, CockroachDB uses OpenTracing libraries for tracing, which also means that
it can be easily integrated with OpenTracing-compatible trace collectors; for example, Lightstep and Zipkin are already supported.

Concept | Description
--------|------------
**trace** | Information about the sub-operations performed as part of a high-level operation (a query or a transaction). This information is internally represented as a tree of "spans", with a special "root span" representing a query execution in the case of `SHOW TRACE FOR <stmt>` or a whole SQL transaction in the case of `SHOW TRACE FOR SESSION`.
**span** | A named, timed operation that describes a contiguous segment of work in a trace. Each span links to "child spans", representing sub-operations; their children would be sub-sub-operations of the grandparent span, etc.<br><br>Different spans can represent (sub-)operations that executed either sequentially or in parallel with respect to each other. (This possibly-parallel nature of execution is one of the important things that a trace is supposed to describe.) The operations described by a trace may be _distributed_, that is, different spans may describe operations executed by different nodes.
**message** | A string with timing information. Each span can contain a list of these. They are produced by CockroachDB's logging infrastructure and are the same messages that can be found in node [log files](debug-and-error-logs.html) except that a trace contains message across all severity levels, whereas log files, by default, do not. Thus, a trace is much more verbose than logs but only contains messages produced in the context of one particular traced operation.

To further clarify these concepts, let's look at a visualization of a trace for one statement. This particular trace is visualized by [Lightstep](http://lightstep.com/) (docs on integrating Lightstep with CockroachDB coming soon). The image only shows spans, but in the tool, it would be possible drill down to messages. You can see names of operations and sub-operations, along with parent-child relationships and timing information, and it's easy to see which operations are executed in parallel.

<div style="text-align: center;"><img src="{{ 'images/v1.1/trace.png' | relative_url }}" alt="Lightstep example" style="border:1px solid #eee;max-width:100%" /></div>

## Response

{{site.data.alerts.callout_info}}The format of the <code>SHOW TRACE</code> response may change in future versions.{{site.data.alerts.end}}

CockroachDB outputs traces in linear tabular format. Each result row represents either a span start (identified by the `=== SPAN START: <operation> ===` message) or a log message from a span. Rows are generally listed in their timestamp order (i.e., the order in which the events they represent occurred) with the exception that messages from child spans are interleaved in the parent span according to their timing. Messages from sibling spans, however, are not interleaved with respect to one another.

The following diagram shows the order in which messages from different spans would be interleaved in an example trace. Each box is a span; inner-boxes are child spans. The numbers indicate the order in which the log messages would appear in the virtual table.

~~~
 +-----------------------+
 |           1           |
 | +-------------------+ |
 | |         2         | |
 | |  +----+           | |
 | |  |    | +----+    | |
 | |  | 3  | | 4  |    | |
 | |  |    | |    |  5 | |
 | |  |    | |    | ++ | |
 | |  +----+ |    |    | |
 | |         +----+    | |
 | |          6        | |
 | +-------------------+ |
 |            7          |
 +-----------------------+
~~~

Each row contains the following columns:

Column | Type | Description
-------|------|------------
`timestamp` | timestamptz | The absolute time when the message occurred.
`age` | interval | The age of the message relative to the beginning of the trace (i.e., the beginning of the statement execution in the case of `SHOW TRACE FOR <stmt>` and the beginning of the recording in the case of `SHOW TRACE FOR SESSION`.
`message` | string.html | The log message.
`context` | string | A prefix of the respective log message indicating meta-information about the message's context. This is the same information that appears in the beginning of log file messages in between square brackets (e.g, `[client=[::1]:49985,user=root,n1]`).
`operation` | string | The name of the operation (or sub-operation) on whose behalf the message was logged.
`span` | tuple(int, int) | A tuple containing the index of the transaction that generated the message (always `0` for `SHOW TRACE FOR <stmt>`) and the index of the span within the virtual list of all spans if they were ordered by the span's start time.

## Examples

### Trace a simple `SELECT`

~~~ sql
> SHOW TRACE FOR SELECT * FROM foo;
~~~

~~~
+----------------------------------+------------+-------------------------------------------------------+-----------------------------------+-----------------------------------+-------+
|            timestamp             |    age     |                        message                        |              context              |             operation             | span  |
+----------------------------------+------------+-------------------------------------------------------+-----------------------------------+-----------------------------------+-------+
| 2017-10-03 18:43:06.878722+00:00 | 0s         | === SPAN START: sql txn implicit ===                  | NULL                              | sql txn implicit                  | (0,0) |
| 2017-10-03 18:43:06.879117+00:00 | 395??s810ns | === SPAN START: starting plan ===                     | NULL                              | starting plan                     | (0,1) |
| 2017-10-03 18:43:06.879124+00:00 | 402??s807ns | === SPAN START: consuming rows ===                    | NULL                              | consuming rows                    | (0,2) |
| 2017-10-03 18:43:06.879155+00:00 | 433??s27ns  | querying next range at /Table/51/1                    | [client=[::1]:49985,user=root,n1] | sql txn implicit                  | (0,0) |
| 2017-10-03 18:43:06.879183+00:00 | 461??s194ns | r18: sending batch 1 Scan to (n1,s1):1                | [client=[::1]:49985,user=root,n1] | sql txn implicit                  | (0,0) |
| 2017-10-03 18:43:06.879202+00:00 | 480??s687ns | sending request to local server                       | [client=[::1]:49985,user=root,n1] | sql txn implicit                  | (0,0) |
| 2017-10-03 18:43:06.879216+00:00 | 494??s435ns | === SPAN START: /cockroach.roachpb.Internal/Batch === | NULL                              | /cockroach.roachpb.Internal/Batch | (0,3) |
| 2017-10-03 18:43:06.879219+00:00 | 497??s599ns | 1 Scan                                                | [n1]                              | /cockroach.roachpb.Internal/Batch | (0,3) |
| 2017-10-03 18:43:06.879221+00:00 | 499??s782ns | read has no clock uncertainty                         | [n1]                              | /cockroach.roachpb.Internal/Batch | (0,3) |
| 2017-10-03 18:43:06.879226+00:00 | 504??s105ns | executing 1 requests                                  | [n1,s1]                           | /cockroach.roachpb.Internal/Batch | (0,3) |
| 2017-10-03 18:43:06.879233+00:00 | 511??s539ns | read-only path                                        | [n1,s1,r18/1:/{Table/51-Max}]     | /cockroach.roachpb.Internal/Batch | (0,3) |
| 2017-10-03 18:43:06.87924+00:00  | 518??s150ns | command queue                                         | [n1,s1,r18/1:/{Table/51-Max}]     | /cockroach.roachpb.Internal/Batch | (0,3) |
| 2017-10-03 18:43:06.879247+00:00 | 525??s568ns | waiting for read lock                                 | [n1,s1,r18/1:/{Table/51-Max}]     | /cockroach.roachpb.Internal/Batch | (0,3) |
| 2017-10-03 18:43:06.879287+00:00 | 565??s196ns | read completed                                        | [n1,s1,r18/1:/{Table/51-Max}]     | /cockroach.roachpb.Internal/Batch | (0,3) |
| 2017-10-03 18:43:06.879318+00:00 | 596??s812ns | plan completed execution                              | [client=[::1]:49985,user=root,n1] | consuming rows                    | (0,2) |
| 2017-10-03 18:43:06.87932+00:00  | 598??s552ns | resources released, stopping trace                    | [client=[::1]:49985,user=root,n1] | consuming rows                    | (0,2) |
+----------------------------------+------------+-------------------------------------------------------+-----------------------------------+-----------------------------------+-------+
(16 rows)
~~~

{{site.data.alerts.callout_success}}You can use <code>SHOW TRACE</code> as the <a href="table-expressions.html">data source</a> for a <code>SELECT</code> statement, and then filter the values with the <code>WHERE</code> clause. For example, to see only messages about spans starting, you might execute <code>SELECT * FROM [SHOW TRACE FOR <stmt>] where message LIKE '=== SPAN START%'</code>.{{site.data.alerts.end}}

### Trace conflicting transactions

In this example, we use two terminals concurrently to generate conflicting transactions.

1. In terminal 1, create a table:

    {% include copy-clipboard.html %}
    ~~~ sql
    > CREATE TABLE t (k INT);
    ~~~

2. Still in terminal 1, open a transaction and perform a write without closing the transaction:

    {% include copy-clipboard.html %}
    ~~~ sql
    > BEGIN;
    ~~~

    {% include copy-clipboard.html %}
    ~~~ sql
    > INSERT INTO t VALUES (1);
    ~~~

    Press enter one more time to send these statements to the server.

3. In terminal 2, execute and trace a conflicting read:

    {% include copy-clipboard.html %}
    ~~~ sql
    > SELECT age, span, message FROM [SHOW TRACE FOR SELECT * FROM t];
    ~~~

    You'll see that this statement is blocked until the transaction in terminal 1 finishes.

4. Back in terminal 1, finish the transaction:

    {% include copy-clipboard.html %}
    ~~~ sql
    > COMMIT;
    ~~~

5. Back in terminal 2, you'll see the completed trace:

    {{site.data.alerts.callout_success}}Check the lines starting with <code>#Annotation</code> for insights into how the conflict is traced.{{site.data.alerts.end}}

	~~~ shell
	+-------------------+--------+-------------------------------------------------------------------------------------------------------+
	|        age        |  span  |            message                                                                                    |
	+-------------------+--------+-------------------------------------------------------------------------------------------------------+
	| 0s                | (0,0)  | === SPAN START: sql txn implicit ===                                                                  |
	| 409??s750ns        | (0,1)  | === SPAN START: starting plan ===                                                                     |
	| 417??s68ns         | (0,2)  | === SPAN START: consuming rows ===                                                                    |
	| 446??s968ns        | (0,0)  | querying next range at /Table/61/1                                                                    |
	| 474??s387ns        | (0,0)  | r42: sending batch 1 Scan to (n1,s1):1                                                                |
	| 491??s800ns        | (0,0)  | sending request to local server                                                                       |
	| 503??s260ns        | (0,3)  | === SPAN START: /cockroach.roachpb.Internal/Batch ===                                                 |
	| 506??s135ns        | (0,3)  | 1 Scan                                                                                                |
	| 508??s385ns        | (0,3)  | read has no clock uncertainty                                                                         |
	| 512??s176ns        | (0,3)  | executing 1 requests                                                                                  |
	| 518??s675ns        | (0,3)  | read-only path                                                                                        |
	| 525??s357ns        | (0,3)  | command queue                                                                                         |
	| 531??s990ns        | (0,3)  | waiting for read lock                                                                                 |
    | # Annotation: The following line identifies the conflict, and some of the lines below it describe the conflict resolution          |
	| 603??s363ns        | (0,3)  | conflicting intents on /Table/61/1/285895906846146561/0                                               |
	| 611??s228ns        | (0,3)  | replica.Send got error: conflicting intents on /Table/61/1/285895906846146561/0                       |
	| # Annotation: The read is now going to wait for the writer to finish by executing a PushTxn request.                               |
	| 615??s680ns        | (0,3)  | pushing 1 transaction(s)                                                                              |
	| 630??s734ns        | (0,3)  | querying next range at /Table/61/1/285895906846146561/0                                               |
	| 646??s292ns        | (0,3)  | r42: sending batch 1 PushTxn to (n1,s1):1                                                             |
	| 658??s613ns        | (0,3)  | sending request to local server                                                                       |
	| 665??s738ns        | (0,4)  | === SPAN START: /cockroach.roachpb.Internal/Batch ===                                                 |
	| 668??s765ns        | (0,4)  | 1 PushTxn                                                                                             |
	| 671??s770ns        | (0,4)  | executing 1 requests                                                                                  |
	| 677??s182ns        | (0,4)  | read-write path                                                                                       |
	| 681??s909ns        | (0,4)  | command queue                                                                                         |
	| 693??s718ns        | (0,4)  | applied timestamp cache                                                                               |
	| 794??s20ns         | (0,4)  | evaluated request                                                                                     |
	| 807??s125ns        | (0,4)  | replica.Send got error: failed to push "sql txn" id=23fce0c4 key=/Table/61/1/285895906846146561/0 ... |
	| 812??s917ns        | (0,4)  | 62cddd0b pushing 23fce0c4 (1 pending)                                                                 |
	| 4s348ms604??s506ns | (0,4)  | result of pending push: "sql txn" id=23fce0c4 key=/Table/61/1/285895906846146561/0 rw=true pri=0  ... |
	| # Annotation: The writer is detected to have finished.                                                                             |
	| 4s348ms609??s635ns | (0,4)  | push request is satisfied                                                                             |
	| 4s348ms657??s576ns | (0,3)  | 23fce0c4-1d22-4321-9779-35f0f463b2d5 is now COMMITTED                                                 |
    | # Annotation: The write has committed. Some cleanup follows.                                                                       |
	| 4s348ms659??s899ns | (0,3)  | resolving intents [wait=true]                                                                         |
	| 4s348ms669??s431ns | (0,17) | === SPAN START: storage.intentResolve: resolving intents ===                                          |
	| 4s348ms726??s582ns | (0,17) | querying next range at /Table/61/1/285895906846146561/0                                               |
	| 4s348ms746??s398ns | (0,17) | r42: sending batch 1 ResolveIntent to (n1,s1):1                                                       |
	| 4s348ms758??s761ns | (0,17) | sending request to local server                                                                       |
	| 4s348ms769??s344ns | (0,18) | === SPAN START: /cockroach.roachpb.Internal/Batch ===                                                 |
	| 4s348ms772??s713ns | (0,18) | 1 ResolveIntent                                                                                       |
	| 4s348ms776??s159ns | (0,18) | executing 1 requests                                                                                  |
	| 4s348ms781??s364ns | (0,18) | read-write path                                                                                       |
	| 4s348ms786??s536ns | (0,18) | command queue                                                                                         |
	| 4s348ms797??s901ns | (0,18) | applied timestamp cache                                                                               |
	| 4s348ms868??s521ns | (0,18) | evaluated request                                                                                     |
	| 4s348ms875??s924ns | (0,18) | acquired {raft,replica}mu                                                                             |
	| 4s349ms150??s556ns | (0,18) | applying command                                                                                      |
	| 4s349ms232??s373ns | (0,3)  | read-only path                                                                                        |
	| 4s349ms237??s724ns | (0,3)  | command queue                                                                                         |
	| 4s349ms241??s857ns | (0,3)  | waiting for read lock                                                                                 |
	| # Annotation: This is where we would have been if there hadn't been a conflict.                                                    |
	| 4s349ms280??s702ns | (0,3)  | read completed                                                                                        |
	| 4s349ms330??s707ns | (0,2)  | output row: [1]                                                                                       |
	| 4s349ms333??s718ns | (0,2)  | output row: [1]                                                                                       |
	| 4s349ms336??s53ns  | (0,2)  | output row: [1]                                                                                       |
	| 4s349ms338??s212ns | (0,2)  | output row: [1]                                                                                       |
	| 4s349ms339??s111ns | (0,2)  | plan completed execution                                                                              |
	| 4s349ms341??s476ns | (0,2)  | resources released, stopping trace                                                                    |
	+-------------------+--------+-------------------------------------------------------------------------------------------------------+
	~~~

### Trace a transaction retry

In this example, we use session tracing to show an [automatic transaction retry](transactions.html#automatic-retries). Like in the previous example, we'll have to use two terminals because retries are induced by unfortunate interactions between transactions.

1. In terminal 1, unset the `smart_prompt` shell option, turn on trace recording, and then start a transaction:

    {% include copy-clipboard.html %}

    ~~~ sql
    > \unset smart_prompt
    ~~~

    {% include copy-clipboard.html %}
    ~~~ sql
    > SET tracing = cluster;
    ~~~

    {% include copy-clipboard.html %}
    ~~~ sql
    > BEGIN;
    ~~~

    Starting a transaction gets us an early timestamp, i.e., we "lock" the snapshot of the data on which the transaction is going to operate.

2. In terminal 2, perform a read:

    {% include copy-clipboard.html %}
    ~~~ sql
    > SELECT * FROM t;
    ~~~

    This read is performed at a timestamp higher than the timestamp of the transaction running in terminal 1. Because we're running at the [`SERIALIZABLE` transaction isolation level](architecture/transaction-layer.html#isolation-levels) (the default), if the system allows terminal 1's transaction to commit, it will have to ensure that ordering terminal 1's transaction *before* terminal 2's transaction is valid; this will become relevant in a second.

3. Back in terminal 1, execute and trace a conflicting write:

    {% include copy-clipboard.html %}
    ~~~ sql
    > INSERT INTO t VALUES (1);
    ~~~

    At this point, the system will detect the conflict and realize that the transaction can no longer commit because allowing it to commit would mean that we have changed history with respect to terminal 2's read. As a result, it will automatically retry the transaction so it can be serialized *after* terminal 2's transaction. The trace will reflect this retry.

4. Turn off trace recording and request the trace:

    {% include copy-clipboard.html %}
	~~~ sql
	> SET tracing = off;
	~~~

    {% include copy-clipboard.html %}
	~~~ sql
	> SELECT age, message FROM [SHOW TRACE FOR SESSION];
	~~~

    {{site.data.alerts.callout_success}}Check the lines starting with <code>#Annotation</code> for insights into how the retry is traced.{{site.data.alerts.end}}

	~~~ shell
	+--------------------+---------------------------------------------------------------------------------------------------------------+
	|        age         |        message                                                                                                |
	+--------------------+---------------------------------------------------------------------------------------------------------------+
	| 0s                 | === SPAN START: sql txn implicit ===                                                                          |
	| 123??s317ns         | AutoCommit. err: <nil>                                                                                        |
	|                    | txn: "sql txn implicit" id=64d34fbc key=/Min rw=false pri=0.02500536 iso=SERIALIZABLE stat=COMMITTED ...      |
	| 1s767ms959??s448ns  | === SPAN START: sql txn ===                                                                                   |
	| 1s767ms989??s448ns  | executing 1/1: BEGIN TRANSACTION                                                                              |
	| # Annotation: First execution of INSERT.                                                                                           |
	| 13s536ms79??s67ns   | executing 1/1: INSERT INTO t VALUES (1)                                                                       |
	| 13s536ms134??s682ns | client.Txn did AutoCommit. err: <nil>                                                                         |
	|                    | txn: "unnamed" id=329e7307 key=/Min rw=false pri=0.01354772 iso=SERIALIZABLE stat=COMMITTED epo=0 ...         |
	| 13s536ms143??s145ns | added table 't' to table collection                                                                           |
	| 13s536ms305??s103ns | query not supported for distSQL: mutations not supported                                                      |
	| 13s536ms365??s919ns | querying next range at /Table/61/1/285904591228600321/0                                                       |
	| 13s536ms400??s155ns | r42: sending batch 1 CPut, 1 BeginTxn to (n1,s1):1                                                            |
	| 13s536ms422??s268ns | sending request to local server                                                                               |
	| 13s536ms434??s962ns | === SPAN START: /cockroach.roachpb.Internal/Batch ===                                                         |
	| 13s536ms439??s916ns | 1 CPut, 1 BeginTxn                                                                                            |
	| 13s536ms442??s413ns | read has no clock uncertainty                                                                                 |
	| 13s536ms447??s42ns  | executing 2 requests                                                                                          |
	| 13s536ms454??s413ns | read-write path                                                                                               |
	| 13s536ms462??s456ns | command queue                                                                                                 |
	| 13s536ms497??s475ns | applied timestamp cache                                                                                       |
	| 13s536ms637??s637ns | evaluated request                                                                                             |
	| 13s536ms646??s468ns | acquired {raft,replica}mu                                                                                     |
	| 13s536ms947??s970ns | applying command                                                                                              |
	| 13s537ms34??s667ns  | coordinator spawns                                                                                            |
	| 13s537ms41??s171ns  | === SPAN START: [async] kv.TxnCoordSender: heartbeat loop ===                                                 |
	| # Annotation: The conflict is about to be detected in the form of a retriable error.                                               |
	| 13s537ms77??s356ns  | automatically retrying transaction: sql txn (id: b4bd1f60-30d9-4465-bdb6-6b553aa42a96) because of error:      |
	|                      HandledRetryableTxnError: serializable transaction timestamp pushed (detected by SQL Executor)                |
	| # Annotation: Second execution of INSERT.                                                                                          |
	| 13s537ms83??s369ns  | executing 1/1: INSERT INTO t VALUES (1)                                                                       |
	| 13s537ms109??s516ns | client.Txn did AutoCommit. err: <nil>                                                                         |
	|                    | txn: "unnamed" id=1228171b key=/Min rw=false pri=0.02917782 iso=SERIALIZABLE stat=COMMITTED epo=0             |
	|                      ts=1507321556.991937203,0 orig=1507321556.991937203,0 max=1507321557.491937203,0 wto=false rop=false          |
	| 13s537ms111??s738ns | releasing 1 tables                                                                                            |
	| 13s537ms116??s944ns | added table 't' to table collection                                                                           |
	| 13s537ms163??s155ns | query not supported for distSQL: writing txn                                                                  |
	| 13s537ms192??s584ns | querying next range at /Table/61/1/285904591231418369/0                                                       |
	| 13s537ms209??s601ns | r42: sending batch 1 CPut to (n1,s1):1                                                                        |
	| 13s537ms224??s219ns | sending request to local server                                                                               |
	| 13s537ms233??s350ns | === SPAN START: /cockroach.roachpb.Internal/Batch ===                                                         |
	| 13s537ms236??s572ns | 1 CPut                                                                                                        |
	| 13s537ms238??s39ns  | read has no clock uncertainty                                                                                 |
	| 13s537ms241??s255ns | executing 1 requests                                                                                          |
	| 13s537ms245??s473ns | read-write path                                                                                               |
	| 13s537ms248??s915ns | command queue                                                                                                 |
	| 13s537ms261??s543ns | applied timestamp cache                                                                                       |
	| 13s537ms309??s401ns | evaluated request                                                                                             |
	| 13s537ms315??s302ns | acquired {raft,replica}mu                                                                                     |
	| 13s537ms580??s149ns | applying command                                                                                              |
	| 18s378ms239??s968ns | executing 1/1: COMMIT TRANSACTION                                                                             |
	| 18s378ms291??s929ns | querying next range at /Table/61/1/285904591228600321/0                                                       |
	| 18s378ms322??s473ns | r42: sending batch 1 EndTxn to (n1,s1):1                                                                      |
	| 18s378ms348??s650ns | sending request to local server                                                                               |
	| 18s378ms364??s928ns | === SPAN START: /cockroach.roachpb.Internal/Batch ===                                                         |
	| 18s378ms370??s772ns | 1 EndTxn                                                                                                      |
	| 18s378ms373??s902ns | read has no clock uncertainty                                                                                 |
	| 18s378ms378??s613ns | executing 1 requests                                                                                          |
	| 18s378ms386??s573ns | read-write path                                                                                               |
	| 18s378ms394??s316ns | command queue                                                                                                 |
	| 18s378ms417??s576ns | applied timestamp cache                                                                                       |
	| 18s378ms588??s396ns | evaluated request                                                                                             |
	| 18s378ms597??s715ns | acquired {raft,replica}mu                                                                                     |
	| 18s383ms388??s599ns | applying command                                                                                              |
	| 18s383ms494??s709ns | coordinator stops                                                                                             |
	| 23s169ms850??s906ns | === SPAN START: sql txn implicit ===                                                                          |
	| 23s169ms885??s921ns | executing 1/1: SET tracing = off                                                                              |
	| 23s169ms919??s90ns  | query not supported for distSQL: SET / SET CLUSTER SETTING should never distribute                            |
	+--------------------+---------------------------------------------------------------------------------------------------------------+
	~~~

## See Also

- [`EXPLAIN`](explain.html)
- [`SET (session settings)`](set-vars.html)
