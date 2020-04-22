# 15-Selected Parallel Algorithms Graph Algorithms

## 单源最短路SSSP

### Dijkstra

> Dijkstra is similar to Prim
>
> 所以相同的技术可以用到最小生成树上。

#### Partition

看输出数据划分，最终输出从s到其它点最小值，p进程n个点。

<img src="typora-user-images\image-20200422081247721.png" alt="image-20200422081247721" style="zoom:50%;" />

#### communication

每个进程局部选最小，然后进行全局reduction，找到全局最小后广播给所有进程，再用这个最小值update最短路。

#### complexity

挑选最小值这一步在这里不是最优的，可以通过最小堆完成，但由于是全局最小堆，有较大的communication开销。

<img src="typora-user-images\image-20200422115527135.png" alt="image-20200422115527135" style="zoom: 50%;" />

### Bellman-ford

#### 队列版本bellman-ford

如果第`i-1`次循环里，v没有被更新，即
$$
v.dist+c(v,w)\ge w.dist\ and\ v\ not\ updated,
$$
则`i`次循环中，还是不会被更新。

队列不用一个个地出，可以同时拿出来。为了防止race condition，对w加锁。

<img src="typora-user-images\image-20200422091415696.png" alt="image-20200422091415696" style="zoom:50%;" />

优化：让加锁区域变小。

<img src="typora-user-images\image-20200422091654853.png" alt="image-20200422091654853" style="zoom: 50%;" />

### delta-stepping

分轻边和重边，在bucket里面做relax（对很多节点的边做）如果delta足够小，则每次只会多一条边的距离，算法退化。防止过度relax，从权值最小的边开始，一步一步来。与Dijkstra相比，relax的更多，但控制了relax的范围。并行度在`B[i]`当中，可以扎堆完成。

<img src="typora-user-images\image-20200422091911670.png" alt="image-20200422091911670" style="zoom:50%;" />

#### 伪代码

![image-20200422114442743](typora-user-images\image-20200422114442743.png)

## All-Pair SP

### Dijkstra

执行n次单源最短路；两种并行策略。

### Floyd

核心部分是维护这个`di,j(k)`，动态规划算法

<img src="typora-user-images\image-20200422082201830.png" alt="image-20200422082201830" style="zoom:50%;" />

#### 划分方法

<img src="typora-user-images\image-20200422082844305.png" alt="image-20200422082844305" style="zoom:50%;" />

#### communication pattern

每一个元素都要对行和列做广播

![image-20200422083055445](typora-user-images\image-20200422083055445.png)

#### 复杂度分析

之前分析过broadcast复杂度为logp

<img src="typora-user-images\image-20200422083404304.png" alt="image-20200422083404304" style="zoom: 50%;" />

#### 伪代码

有些并行度没有发掘出来，两条黑线的地方需要synchronization。有的块都算完了，还要等其他人算完，才能进行下一轮。优化思路，把每个block看成生产者消费者，每次迭代大部分做消费者，如果行或列有当前的k，则它既是消费者又是生产者。

![image-20200422083608649](typora-user-images\image-20200422083608649.png)

#### pipelining

不需要同步，只要processor算完k-1轮且拿到相应D(k-1)矩阵的部分，就能开始做第k轮。

由于没有synchronization，则盯着最后一个数据看，则O(n)后能够拿到第一行第一列的数据。通过这种方法分析出算法的时间。

![image-20200422084608328](typora-user-images\image-20200422084608328.png)

pipelining情况下，communication复杂度降成线性

<img src="typora-user-images\image-20200422084900769.png" alt="image-20200422084900769" style="zoom: 50%;" />

#### 效率分析

等效率情况下，最多能够有多大processor，达到的复杂度是多少。例如最后一个，
$$
p\to n^2, \frac{n^3}{p}+n \to n
$$
![image-20200422085558474](typora-user-images\image-20200422085558474.png)

等效率关系可留成作业。



## 极大独立集

> 最大独立集NP难，找到局部最优解就可以返回

极大独立集：不是任何一个独立集的真子集。

每次选一个点，删除它的邻居，重复这个过程。需要分布式随机数生成算法，只需要和邻居节点进行交流。如果一个点的随机数比它周围的节点都小，则把它加到独立集中。比如这里就将1，5顶点加入到独立集中。

![image-20200422094921475](typora-user-images\image-20200422094921475.png)