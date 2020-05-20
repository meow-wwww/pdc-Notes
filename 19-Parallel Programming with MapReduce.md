# 19-Parallel Programming with MapReduce

很多设计上的背景都是批处理。有一定的表达能力，用户不用自己选择机器、划分。自动并行化，让非专业的人也能写出。

## Execution Details

- Input reader，把输入分片。
- Map task，对每个记录用 Map function 作用，输出为`(key, value)`对。
- Shuffle/Partition and Sort，框架自动把相同 key 的记录分配给相同 reduce processor，sort 用于将相同 key 的记录集合起来。
- Reduce task，对每个 key 用 Reduce function 作用（作用在局部的 list 上）。

## Runtime Environment & Hadoop

### Runtime Environment

- <u>Partitioning</u> the input data
- <u>Scheduling</u> program across clusters of machines, Locality Optimization and Load balancing
- Dealing with <u>machine failure</u>
- managing inter-machine <u>communication</u>

### Fault Tolerance

要求user的task： deterministic and side-effect-free，容错机制才能执行

- Handled via re-execution of tasks

- Mappers save outputs to <u>local disk</u> before serving to reducers （map的结果保存在磁盘上，比较慢）
- If a task crashes:
  - Retry on another node（map没有依赖性，无影响；reduce输入是map的输出，存在磁盘上了，无影响）
  - If the same task repeatedly fails, fail the job or ignore that input block

- If a node crashes:
  - Relaunch its current tasks on other nodes
  - Relaunch <u>any maps</u> the node previously ran（因为输出文件一并丢失）

### Locality Optimization

- Leverage the <u>distributed file system</u> to schedule a map task on a machine that contains a replica of the corresponding input data. （顺着最开始分布式文件系统存储的分布做 map task，保证每个节点读入自己机器上的输入）

### Redundant Execution

有的节点可能特别慢，其他都要等着。冗余操作能够提升整体集群的效率。

- Near end of phase, spawn backup tasks, one to finish first wins. （同时执行一个 task，只保留最先出来的结果，其它丢掉）

### Skipping Bad Records

- Map/Reduce functions sometimes fail for particular inputs. （输入输出不合法，则直接 skip）
- Fixing the Bug might not be possible : Third Party Libraries. （第三方库预处理的锅，跟程序员无关）

### Miscellaneous Refinements

细节的 refinement

- Combiner function at a map task （combiner 作用相当于 mini-reduction，在输出到 reducer 之前把相同的 key 合并）

## Performance evaluation

### Two benchmarks:

- `MR_GrepScan` 找匹配的字符串
- `MR_SortSort ` Backup tasks reduce job completion time a lot! （备份能够减少执行时间，还模拟了比较差的情况，kill了200个task）

## Hadoop & Tools

把 MapReduce 应用到 Hadoop 上

- Pig Latin, a SQL-like high level data processing script language
- Hive, Data warehouse, SQL
- Mahout, Machine Learning algorithms on Hadoop
- HBase, Distributed data store as a large table

## MapReduce Use Case

讲 6 个应用，怎么把一个任务分解成 Map/Reduce

### Map Only processing

document classification

- Input: `(docno, doc_content)`, …
- Output: `(docno, [class, class, …])`, …

### Filtering and accumulation

Counting total enrollments of two given student classes （统计两门课的学生人数）

Map Function:

- Input: `(name, classid)`
- Output: `(classid, 1)`

Reduce Function:

- In: `(11493, [1, 1, …])`, `(11741, [1, 1, …])`
- Sum and Output: `(11493, 16)`, `(11741, 35)`

### Database join

数据库操作，两个表合成一个表

- Given two large lists: `(URL, ID)` and `(URL, doc_content)` pairs
- Produce `(URL, ID, doc_content)`  or `(ID, doc_content)`
- Map 直接 pass input
- Reduce 把 key value 调整一下

### Reversing graph edges

- Input example: adjacency list of graph（每个点的邻接点list，ppt上有例子）
- Map 把 adjacency list 打散，弄成点对形式，把顺序反过来
- 直接 reduce

### Producing inverted index for web search

倒排索引，搜索引擎中常用，正常索引是文章id+content，倒排：word+文章id

- Map: `(docid1, content1) →(t1, docid1) (t2, docid1) …` （t1, t2 表示需要统计的关键词）
- Shuffle by t, prepared for map-reducer communication
- Sort by t,  conducted in a reducer machine `(t5, docid1) (t4, docid3) … →(t4, docid3) (t4, docid1) (t5, docid1) …`
- Reduce:  `(t4, [docid3 docid1 …]) →(t, ilist)`

Using `Combine()` to Reduce Communication（调用的时机在 map 之后，在 map 的机器上）

- Map: `(docid1, content1) →(t1, ilist1,1) (t2, ilist2,1) (t3, ilist3,1) …`
  - Each output inverted list covers just one document
- Combine locally 
  - Sort by t
  - Combine:  `(t1 [ilist1,2 ilist1,3 ilist1,1 …]) →(t1, ilist1,27)`
  - Each output inverted list covers a sequence of documents （partial list，不是 final list）

- Shuffle by t
- Sort by t `(t4, ilist4,1) (t5, ilist5,3) … →(t4, ilist4,2) (t4, ilist4,4) (t4, ilist4,1) …`
- Reduce:  `(t7, [ilist7,2, ilist3,1, ilist7,4, …]) →(t7, ilistfinal)`

Construct Partitioned Indexes

- Useful when the document list of a term does not fit memory
- 每一项不用存全部的 list，只存一个分片

### PageRank graph processing

类似于 citation，有很多，或者被大牛引用

- Input: `(id1, [score1(t), out11, out12, ..]), (id2, [score2(t), out21, out22, ..]) ..`
- Output: `(id1, [score1(t+1), out11, out12, ..]), (id2, [score2(t+1), out21, out22, ..])`

具体实现他讲的不详细，放个网页供参考[基于MapReduce的PageRank算法实现](https://blog.csdn.net/a819825294/article/details/53674353)

# Parallel Programming with Spark

这些框架都不是重复造轮子。新的需求：complex apps & interactive queries

## lacks of MapReduce

- slow due to replication （每一次有三份，为了容错）& disk I/O，直观改进：In-Memory Data Sharing，问题是<u>如何容错/保持效率</u>？
- replication 是一样的，只是变成了内存里存三份。

### Resilient Distributed Dataset(RDDs)

- 加了约束，Immutable（不能修改），coarse-grained 的更新。  
- Fault recovery using lineage（用log，coarse-grained 的限制使 log 变得可能）
- 细节不再展开（看ppt）

