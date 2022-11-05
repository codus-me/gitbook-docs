# 15.3. 平行查詢計畫

Because each worker executes the parallel portion of the plan to completion, it is not possible to simply take an ordinary query plan and run it using multiple workers. Each worker would produce a full copy of the output result set, so the query would not run any faster than normal but would produce incorrect results. Instead, the parallel portion of the plan must be what is known internally to the query optimizer as a _partial plan_; that is, it must be constructed so that each process which executes the plan will generate only a subset of the output rows in such a way that each required output row is guaranteed to be generated by exactly one of the cooperating processes. Generally, this means that the scan on the driving table of the query must be a parallel-aware scan.

## 15.3.1. Parallel Scans

The following types of parallel-aware table scans are currently supported.

* In a _parallel sequential scan_, the table's blocks will be divided among the cooperating processes. Blocks are handed out one at a time, so that access to the table remains sequential.
* In a _parallel bitmap heap scan_, one process is chosen as the leader. That process performs a scan of one or more indexes and builds a bitmap indicating which table blocks need to be visited. These blocks are then divided among the cooperating processes as in a parallel sequential scan. In other words, the heap scan is performed in parallel, but the underlying index scan is not.
* In a _parallel index scan_ or _parallel index-only scan_, the cooperating processes take turns reading data from the index. Currently, parallel index scans are supported only for btree indexes. Each process will claim a single index block and will scan and return all tuples referenced by that block; other processes can at the same time be returning tuples from a different index block. The results of a parallel btree scan are returned in sorted order within each worker process.

Other scan types, such as scans of non-btree indexes, may support parallel scans in the future.

## 15.3.2. Parallel Joins

Just as in a non-parallel plan, the driving table may be joined to one or more other tables using a nested loop, hash join, or merge join. The inner side of the join may be any kind of non-parallel plan that is otherwise supported by the planner provided that it is safe to run within a parallel worker. Depending on the join type, the inner side may also be a parallel plan.

* In a _nested loop join_, the inner side is always non-parallel. Although it is executed in full, this is efficient if the inner side is an index scan, because the outer tuples and thus the loops that look up values in the index are divided over the cooperating processes.
* In a _merge join_, the inner side is always a non-parallel plan and therefore executed in full. This may be inefficient, especially if a sort must be performed, because the work and resulting data are duplicated in every cooperating process.
* In a _hash join_ (without the "parallel" prefix), the inner side is executed in full by every cooperating process to build identical copies of the hash table. This may be inefficient if the hash table is large or the plan is expensive. In a _parallel hash join_, the inner side is a _parallel hash_ that divides the work of building a shared hash table over the cooperating processes.

## 15.3.3. Parallel Aggregation

PostgreSQL supports parallel aggregation by aggregating in two stages. First, each process participating in the parallel portion of the query performs an aggregation step, producing a partial result for each group of which that process is aware. This is reflected in the plan as a `Partial Aggregate` node. Second, the partial results are transferred to the leader via `Gather` or `Gather Merge`. Finally, the leader re-aggregates the results across all workers in order to produce the final result. This is reflected in the plan as a `Finalize Aggregate` node.

Because the `Finalize Aggregate` node runs on the leader process, queries which produce a relatively large number of groups in comparison to the number of input rows will appear less favorable to the query planner. For example, in the worst-case scenario the number of groups seen by the `Finalize Aggregate` node could be as many as the number of input rows which were seen by all worker processes in the `Partial Aggregate` stage. For such cases, there is clearly going to be no performance benefit to using parallel aggregation. The query planner takes this into account during the planning process and is unlikely to choose parallel aggregate in this scenario.

Parallel aggregation is not supported in all situations. Each aggregate must be [safe](https://www.postgresql.org/docs/12/parallel-safety.html) for parallelism and must have a combine function. If the aggregate has a transition state of type `internal`, it must have serialization and deserialization functions. See [CREATE AGGREGATE](https://www.postgresql.org/docs/12/sql-createaggregate.html) for more details. Parallel aggregation is not supported if any aggregate function call contains `DISTINCT` or `ORDER BY` clause and is also not supported for ordered set aggregates or when the query involves `GROUPING SETS`. It can only be used when all joins involved in the query are also part of the parallel portion of the plan.

## 15.3.4. Parallel Append

每當 PostgreSQL 需要將來自多個資料源的資料列合併到一個結果集合之中時，它都會使用一個 Append 或 MergeAppend 的查詢計劃節點。在執行 UNION ALL 或掃描分割區資料表時，通常會是這種情況。這樣的節點可以在平行查詢計劃中使用，就像在其他任何查詢計劃中一樣。但是，在平行查詢計劃中，查詢計劃程序可以改為使用「Parallel Append」節點。

當在平行查詢計劃中使用 Append 節點時，每個流程將按照它們出現的順序執行子計劃，以便所有參與的流程合作執行第一個子計劃，直到完成為止，然後移至第二個計劃。 大約在同一時間。 當使用 Parallel Append 時，執行程序時，將在其子計劃中盡可能平均地分散參與的程序，以便同時執行多個子計劃。 這樣可以避免競爭使用，也可以避免在從未執行子計劃的程序中負擔子計劃的啟動成本。

同樣，與一般的 Append 節點不同，一般的 Append 節點在平行查詢計劃中使用時只能具有 partial 子項，而 Parallel Append 節點可以同時具有 partial 和 non-partial 子計劃。non-partial 子項只能透過一個程序進行掃描，因為多次掃描它們會產生重複的結果內容。因此，即使沒有有效的局部計劃，涉及附加多個結果集合的計劃也可以實作粗粒度的平行處理。例如，考慮對分割資料表的查詢，該查詢只能透過使用不支援平行掃描的索引來有效地實作。計劃程序可以選擇“一般索引掃描”計劃的“Parallel Append”；每個單獨的索引掃描都必須由一個程序執行才能完成，但是不同的程序可以同時執行不同的掃描。

[enable\_parallel\_append](../../server-administration/server-configuration/query-planning.md#enable\_parallel\_append-boolean) 可用於停用此功能。

## 15.3.5. Parallel Plan Tips

If a query that is expected to do so does not produce a parallel plan, you can try reducing [parallel\_setup\_cost](https://www.postgresql.org/docs/12/runtime-config-query.html#GUC-PARALLEL-SETUP-COST) or [parallel\_tuple\_cost](https://www.postgresql.org/docs/12/runtime-config-query.html#GUC-PARALLEL-TUPLE-COST). Of course, this plan may turn out to be slower than the serial plan which the planner preferred, but this will not always be the case. If you don't get a parallel plan even with very small values of these settings (e.g. after setting them both to zero), there may be some reason why the query planner is unable to generate a parallel plan for your query. See [Section 15.2](https://www.postgresql.org/docs/12/when-can-parallel-query-be-used.html) and [Section 15.4](https://www.postgresql.org/docs/12/parallel-safety.html) for information on why this may be the case.

When executing a parallel plan, you can use `EXPLAIN (ANALYZE, VERBOSE)` to display per-worker statistics for each plan node. This may be useful in determining whether the work is being evenly distributed between all plan nodes and more generally in understanding the performance characteristics of the plan.