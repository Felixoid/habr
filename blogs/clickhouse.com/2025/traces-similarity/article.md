# Trace Similarity system on the top of ClickHouse to analyze crash stack traces from our CI

## Problem

Our CI system is very good at catching bugs, and sometimes it catches crash logs during AST Fuzzing, Stress and Functional tests. However, the crash reports are often very similar, and it is hard to understand which bugs are actually new and which ones are already known.

Quoting the [TraceSim: A Method for Calculating Stack Trace Similarity](https://arxiv.org/pdf/2009.12590) article:

> Many contemporary software products have subsystems for automatic crash reporting. However, it is well-known that the same bug can produce slightly different reports.

This paper's methodology is based on the idea that similar stack traces are likely to be caused by the same bug. Authors propose a method for calculating stack trace similarity, which can be used to group similar crash stack traces together.

The system to analyze and group traces from the cloud was first built by our SRE Michael Stetsyuk. It groups crash reports by their stack traces and allows to track them in the issue tracker.

Similar to that system, we wanted to build a trace similarity system on top of ClickHouse to analyze crash stack traces from our CI. The system should be able to:

- Parse crash reports and extract stack frames.
- Calculate similarity between stack traces.
- Group similar new stack traces together with known.
- Create and update issues on GitHub to track the bugs.

I'll describe below each step of the process in detail, so you can build a similar system for your own needs.

## How do we do it?

### Step zero: collect crash reports, create all the necessary tables

We need to collect crash reports from our CI system. Luckily, the system already works for a few years.

Briefly described, the system creates [Materialized views](https://clickhouse.com/docs/materialized-view/incremental-materialized-view) inserting new rows into remote ClickHouse cluster's tables, where the destination tables' names are calculated from the current table structures + additional columns. It allows to have historical date in tables with different structures, and also allows to have different tables for different CI jobs. The script, managing these tables, is located in our [ci](https://github.com/ClickHouse/ClickHouse/blob/b499c8df1f091cbba234bbf23ce8473a5834e8aa/ci/jobs/scripts/functional_tests/setup_log_cluster.sh#L74) directory.

We need the data from [system.crash_log](https://clickhouse.com/docs/operations/system-tables/crash-log) tables. That's how tables on the CI Logs cluster look like now:

```sql
:) SHOW TABLES LIKE 'crash_log_%';

    ┌─name───────────────────────────┐
 1. │ crash_log_12587237819650651296 │
 2. │ crash_log_12670418084883306529 │
 3. │ crash_log_1527066305010279420  │
 4. │ crash_log_15355897847728332522 │
 5. │ crash_log_15557244372725679386 │
 6. │ crash_log_18405985218782237968 │
 7. │ crash_log_2288102012038531617  │
 8. │ crash_log_3310266143589491008  │
 9. │ crash_log_6802555697904881735  │
10. │ crash_log_9016585404038675675  │
11. │ crash_log_9097266775814416937  │
12. │ crash_log_9243005856023138905  │
13. │ crash_log_9768092148702997133  │
14. │ crash_logs                     │
    └────────────────────────────────┘

:) SHOW CREATE TABLE crash_logs;

CREATE TABLE default.crash_logs
(
    `repo` String,
    `pull_request_number` UInt32,
    `commit_sha` String,
    `check_start_time` DateTime,
    `check_name` String,
    `instance_type` String,
    `instance_id` String,
    `hostname` LowCardinality(String),
    `event_date` Date,
    `event_time` DateTime,
    `timestamp_ns` UInt64,
    `signal` Int32,
    `thread_id` UInt64,
    `query_id` String,
    `trace` Array(UInt64),
    `trace_full` Array(String),
    `version` String,
    `revision` UInt32,
    `build_id` String
)
ENGINE = Merge('default', '^crash_log_')
```

<details><summary>Here is an example of the data we store there</summary>

```
SELECT *
FROM crash_logs
WHERE event_date = today()
LIMIT 1

Query id: 4cf55288-1f15-47c9-a73c-e14c165a8181

Row 1:
──────
repo:                ClickHouse/ClickHouse
pull_request_number: 85843
commit_sha:          41215391373ad2e277230939e887c834edeb16ce
check_start_time:    2025-08-19 10:07:50
check_name:          Stateless tests (arm_binary, parallel)
instance_type:       c8g.8xlarge
instance_id:         i-0b53d7dc362ab4cd8
hostname:            16ffef362c03
event_date:          2025-08-19
event_time:          2025-08-19 10:09:04
timestamp_ns:        1755598144624168055
signal:              6
thread_id:           6359
query_id:            5ce98be8-f4cb-4950-82a8-cc2720669cad
trace:               [280796479681009,280796479399548,280796479320368,188029769003464,188029769005984,188029769006680,188029679339096,188029679337384,188029679380404,188029836833040,188029837238400,188029837230992,188029837232044,188029915248680,188029915247868,188029915340828,188029915297288,188029915294080,188029915293160,…]
trace_full:          ['3. ? @ 0x000000000007f1f1','4. ? @ 0x000000000003a67c','5. ? @ 0x0000000000027130','6. ./ci/tmp/build/./src/Common/Exception.cpp:51: DB::abortOnFailedAssertion(String const&, void* const*, unsigned long, unsigned long) @ 0x000000000e33b1c8','7. ./ci/tmp/build/./src/Common/Exception.cpp:84: DB::handle_error_code(String const&, std::basic_string_view<char, std::char_traits<char>>, int, bool, std::vector<void*, std::allocator<void*>> const&) @ 0x000000000e33bba0','8. ./ci/tmp/build/./src/Common/Exception.cpp:135: DB::Exception::Exception(DB::Exception::MessageMasked&&, int, bool) @ 0x000000000e33be58','9. DB::Exception::Exception(String&&, int, String, bool) @ 0x0000000008db8658','10. DB::Exception::Exception(PreformattedMessage&&, int) @ 0x0000000008db7fa8','11. DB::Exception::Exception<>(int, FormatStringHelperImpl<>) @ 0x0000000008dc27b4','12. ./ci/tmp/build/./src/Storages/ObjectStorage/StorageObjectStorageConfiguration.cpp:212: DB::StorageObjectStorageConfiguration::addDeleteTransformers(std::shared_ptr<DB::RelativePathWithMetadata>, DB::QueryPipelineBuilder&, std::optional<DB::FormatSettings> const&, std::shared_ptr<DB::Context const>) const @ 0x00000000123eb110','13. ./ci/tmp/build/./src/Storages/ObjectStorage/StorageObjectStorageSource.cpp:555: DB::StorageObjectStorageSource::createReader(unsigned long, std::shared_ptr<DB::IObjectIterator> const&, std::shared_ptr<DB::StorageObjectStorageConfiguration> const&, std::shared_ptr<DB::IObjectStorage> const&, DB::ReadFromFormatInfo&, std::optional<DB::FormatSettings> const&, std::shared_ptr<DB::Context const> const&, DB::SchemaCache*, std::shared_ptr<Poco::Logger> const&, unsigned long, std::shared_ptr<DB::FormatParserSharedResources>, std::shared_ptr<DB::FormatFilterInfo>, bool) @ 0x000000001244e080','14.0. inlined from ./ci/tmp/build/./src/Storages/ObjectStorage/StorageObjectStorageSource.cpp:412: DB::StorageObjectStorageSource::createReader()','14. ./ci/tmp/build/./src/Storages/ObjectStorage/StorageObjectStorageSource.cpp:265: DB::StorageObjectStorageSource::lazyInitialize() @ 0x000000001244c390','15. ./ci/tmp/build/./src/Storages/ObjectStorage/StorageObjectStorageSource.cpp:274: DB::StorageObjectStorageSource::generate() @ 0x000000001244c7ac','16. ./ci/tmp/build/./src/Processors/ISource.cpp:144: DB::ISource::tryGenerate() @ 0x0000000016eb3828','17. ./ci/tmp/build/./src/Processors/ISource.cpp:110: DB::ISource::work() @ 0x0000000016eb34fc','18.0. inlined from ./ci/tmp/build/./src/Processors/Executors/ExecutionThreadContext.cpp:53: DB::executeJob(DB::ExecutingGraph::Node*, DB::ReadProgressCallback*)','18. ./ci/tmp/build/./src/Processors/Executors/ExecutionThreadContext.cpp:102: DB::ExecutionThreadContext::executeTask() @ 0x0000000016eca01c','19. ./ci/tmp/build/./src/Processors/Executors/PipelineExecutor.cpp:350: DB::PipelineExecutor::executeStepImpl(unsigned long, DB::IAcquiredSlot*, std::atomic<bool>*) @ 0x0000000016ebf608',…]
version:             ClickHouse 25.8.1.1
revision:            54501
build_id:            88351121A8340C83AB8A60BA97765ADC4B9B7786
```

</details>


Perfect, we have the raw data to analyze.

#### Separate collecting and analyzing clusters, create tables for analysis

To avoid the performance impact on the CI system, we will use a separate ClickHouse cluster to analyze the crash traces. The following tables and materialized view will manage updating the data from the remote CI Logs cluster:

```sql
-- Create a table to store the raw stack traces with details about where and when they were appeared
CREATE TABLE default.stack_traces
(
    `repo` LowCardinality(String),
    `pull_request_number` UInt32,
    `commit_sha` String,
    `check_name` String,
    `check_start_time` DateTime('UTC'),
    `event_time` DateTime,
    `timestamp_ns` UInt64,
    `trace_full` Array(String),
    `Version` UInt32 DEFAULT now()
)
ENGINE = ReplacingMergeTree(Version)
PARTITION BY toYYYYMM(event_time)
ORDER BY (event_time, timestamp_ns, repo, check_start_time)
SETTINGS index_granularity = 8192;

-- create a refresheable materialized view to collect the data from the crash_logs table for the last 100 days
-- https://clickhouse.com/docs/materialized-view/refreshable-materialized-view
CREATE MATERIALIZED VIEW default._stack_traces_mv
REFRESH EVERY 30 MINUTE TO default.stack_traces
AS SELECT
    repo,
    pull_request_number,
    commit_sha,
    check_name,
    check_start_time,
    event_time,
    timestamp_ns,
    trace_full,
    now() AS Version
FROM remoteSecure('[HOST]', 'default', 'crash_logs', 'username', 'password')
WHERE event_date >= today() - INTERVAL 100 DAY;

-- Create a table to store the info about the GitHub issues created for the stack traces
CREATE TABLE default.crash_issues
(
    `created_at` DateTime,
    `updated_at` DateTime,
    `closed_at` DateTime,
    `repo` String,
    `number` UInt32,
    `state` Enum8('open' = 1, 'closed' = 2),
    `stack_traces_full` Array(Array(String)),
    `stack_traces_hash` Array(UInt64)
)
ENGINE = ReplacingMergeTree(updated_at)
PARTITION BY toYYYYMM(created_at)
ORDER BY (repo, number)
SETTINGS index_granularity = 8192;
```

From this point, we will use data on this cluster. The data will be refreshed every 30 minutes, so we will have the latest crash events available for analysis.

### Step one: clean crash traces

Now, when we have the traces, we need to clean them up. The traces contain a lot of noise that can affect the similarity calculation. The noise can come from:

- Different compiler versions.
- Different build configurations (e.g. debug vs release).
- Different environments (e.g. different OS, different hardware).
- Different file names and line numbers in the stack traces.

To ease the analysis, we create two UDFs to clean the traces:

```sql
CREATE FUNCTION cleanStackFrame AS frame -> replaceRegexpOne(
  replaceRegexpAll(
    splitByString(
      -- second, strip the file name and line number, keep the function name
      ': ', splitByString(
        -- first, strip the frame address, keep the frame number, file name, line number, and function name
        ' @ ', frame, 2
      )[1], 2
    )[2],
    -- third, replace all ABI specific information with a placeholder
    '\\[abi:[^]]+\\]', '[$ABI]'),
  -- finally, delete all LLVM specific information
  '(\\s+[(])*[.]llvm[.].+', ''
);

CREATE FUNCTION cleanStackTrace AS trace -> arrayFilter(
  frame -> (
    -- second, keep only meaningful frames
    frame NOT IN ('', '?')), arrayMap(
      -- first, clean each frame of the stack trace
      fr -> cleanStackFrame(fr), trace
  )
);
```

These functions will be used to clean the stack traces before calculating their similarity on the fly. The `cleanStackFrame` function removes the noise from each frame, and the `cleanStackTrace` function applies it to the whole stack trace.

There's a possibility to use the functions to pre-process the data in the `stack_traces` table, but we will do it on the fly to avoid storing the cleaned traces in the database. It gives us more flexibility and allows to change the cleaning logic without processing all the data again. It's especially useful at the step of calculating weights for the frames, which will be described later.

### Step two: group identical traces and distinguish significant among them

Now, we have the cleaned stack traces, and we can start grouping them. Already processed traces present in the `crash_issues` table, so we can use it to find the existing issues.

```sql
WITH known_hashes AS
    (
        SELECT
            repo,
            -- Avoid default value of 'open' for state
            toString(state) AS state,
            updated_at,
            closed_at,
            arrayFlatten(groupArrayDistinct(stack_traces_hash)) AS known_hashes
        FROM default.crash_issues FINAL
        GROUP BY ALL
    )
SELECT
    repo,
    groupArrayDistinct((pull_request_number, commit_sha, check_name)) AS checks,
    cleanStackTrace(trace_full) AS trace_full,
    sipHash64(trace_full) AS trace_hash,
    length(groupArrayDistinct(pull_request_number) AS PRs) AS trace_count,
    has(PRs, 0) AS is_in_master
FROM default.stack_traces AS st
LEFT JOIN known_hashes AS kh ON (st.repo = kh.repo AND has(kh.known_hashes, trace_hash))
WHERE (st.event_time >= (now() - toIntervalDay(30)))
    AND (length(trace_full) > 5)
GROUP BY
    repo,
    trace_full
HAVING
    groupArrayDistinct(kh.state) = [''] -- new trace, no issues
    OR (groupArrayDistinct(kh.state) = ['closed'] AND max(st.check_start_time) > max(kh.closed_at)) -- new event after the issue is closed
ORDER BY max(st.event_time)
```

This query groups the stack traces by their cleaned version and calculates the hash of the trace. The hashes are checked against the `crash_issues` table to find out if the trace is already known. If there is an open issue, or the trace is created before the existing issues were closed, it is considered a known trace.

The query also counts the number of unique pull requests that contain the trace and checks if the trace is present in the master branch (pull request number 0). If the trace appears in less than 3 pull requests and wasn't seen in master/release branches, it's considered insignificant. Such traces aren't used to create issues, but they can be added to existing issues if they are similar enough to the known traces.


### Step three: calculate similarity between traces

The following query is the [TraceSim's paper](https://arxiv.org/pdf/2009.12590) implementation of the similarity calculation. ClickHouse has `arraySimilarity` function since version [25.4](https://clickhouse.com/docs/whats-new/changelog#254), the weights are calculated on the fly.

```sql
WITH
    1.97 AS alpha,
    2.0 AS beta,
    3.7 AS gamma,
    0.68 AS threshold,
    stack_frame_weights AS (
        WITH
            (
                SELECT count()
                FROM default.stack_traces
                FINAL
            ) AS total
        SELECT
            arrayJoin(cleanStackTrace(trace_full)) AS frame,
            countDistinct(trace_full) AS count,
            log(total / count) AS IDF,
            sigmoid(beta * (IDF - gamma)) AS weight
        FROM default.stack_traces
        FINAL
        GROUP BY frame
    ),
    (SELECT groupArray(weight) AS w, groupArray(frame) AS f FROM stack_frame_weights) AS weights,
    (trace -> arrayMap((_frame, pos) -> (pow(pos, -alpha) * arrayFirst(w, f -> (f = _frame), weights.w, weights.f)), trace, arrayEnumerate(trace))) AS get_trace_weights,
    ['DB::abortOnFailedAssertion(String const&, void* const*, unsigned long, unsigned long)','DB::handle_error_code(String const&, std::basic_string_view<char, std::char_traits<char>>, int, bool, std::vector<void*, std::allocator<void*>> const&)','DB::Exception::Exception(DB::Exception::MessageMasked&&, int, bool)','DB::Exception::Exception<String const&, String, String const&>(int, FormatStringHelperImpl<std::type_identity<String const&>::type, std::type_identity<String>::type, std::type_identity<String const&>::type>, String const&, String&&, String const&)','DB::paranoidCheckForCoveredPartsInZooKeeper(std::shared_ptr<zkutil::ZooKeeper> const&, String const&, StrongTypedef<unsigned int, DB::MergeTreeDataFormatVersionTag>, String const&, DB::StorageReplicatedMergeTree const&)','DB::StorageReplicatedMergeTree::executeDropRange(DB::ReplicatedMergeTreeLogEntry const&)','DB::StorageReplicatedMergeTree::executeLogEntry(DB::ReplicatedMergeTreeLogEntry&)','operator()','decltype(std::declval<DB::StorageReplicatedMergeTree::processQueueEntry(std::shared_ptr<DB::ReplicatedMergeTreeQueue::SelectedEntry>)::$_1&>()(std::declval<std::shared_ptr<DB::ReplicatedMergeTreeLogEntry>&>())) std::__invoke[$ABI]<DB::StorageReplicatedMergeTree::processQueueEntry(std::shared_ptr<DB::ReplicatedMergeTreeQueue::SelectedEntry>)::$_1&, std::shared_ptr<DB::ReplicatedMergeTreeLogEntry>&>(DB::StorageReplicatedMergeTree::processQueueEntry(std::shared_ptr<DB::ReplicatedMergeTreeQueue::SelectedEntry>)::$_1&, std::shared_ptr<DB::ReplicatedMergeTreeLogEntry>&)','bool std::__invoke_void_return_wrapper<bool, false>::__call[$ABI]<DB::StorageReplicatedMergeTree::processQueueEntry(std::shared_ptr<DB::ReplicatedMergeTreeQueue::SelectedEntry>)::$_1&, std::shared_ptr<DB::ReplicatedMergeTreeLogEntry>&>(DB::StorageReplicatedMergeTree::processQueueEntry(std::shared_ptr<DB::ReplicatedMergeTreeQueue::SelectedEntry>)::$_1&, std::shared_ptr<DB::ReplicatedMergeTreeLogEntry>&)','DB::ReplicatedMergeTreeQueue::processEntry(std::function<std::shared_ptr<zkutil::ZooKeeper> ()>, std::shared_ptr<DB::ReplicatedMergeTreeLogEntry>&, std::function<bool (std::shared_ptr<DB::ReplicatedMergeTreeLogEntry>&)>)','DB::StorageReplicatedMergeTree::processQueueEntry(std::shared_ptr<DB::ReplicatedMergeTreeQueue::SelectedEntry>)','DB::ExecutableLambdaAdapter::executeStep()','DB::MergeTreeBackgroundExecutor<DB::RoundRobinRuntimeQueue>::routine(std::shared_ptr<DB::TaskRuntimeData>)','DB::MergeTreeBackgroundExecutor<DB::RoundRobinRuntimeQueue>::threadFunction()'] AS new_trace,
    get_trace_weights(new_trace) AS new_trace_weights
SELECT
    arraySimilarity(
        new_trace,
        arrayJoin(stack_traces_full) AS trace_full,
        new_trace_weights,
        get_trace_weights(trace_full)
    ) AS similarity,
    repo,
    number,
    created_at,
    closed_at,
    stack_traces_full,
    stack_traces_hash
FROM default.crash_issues FINAL
WHERE repo = 'ClickHouse/ClickHouse'
    AND state = 'open'
    AND threshold <= similarity
```

In our system, significant traces are collected and processed on an hourly basis, so we can analyze them in a timely manner. If no similar traces are found for a significant new trace, we create a new issue in the GitHub repository and add the row to the `crash_issues` table.

Once a day we process all the traces including insignificant ones, to find out if they are similar to the known significant traces. If a trace with similarity higher the threshold is found, we add it to the existing issue.

A pinch of LLM helps us generating the issue title and generating possible reasons of the crash based on the stack trace. All automatically created issues can be found in the repository with [crash-ci](https://github.com/ClickHouse/clickhouse/issues?q=state%3Aopen%20label%3Acrash-ci) label. As always, your help is appreciated, so if you find an issue that can be fixed, please feel free to contribute!
