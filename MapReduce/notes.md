# MapReduce

## 背景

MapReduce 的诞生背景是 Google 内部有大量的爬虫文档、web 请求日志等大量数据计算的场景，最初通过代码来计算处理数据的方式让这块代码变得越来越复杂和不可维护，也因此，Google 尝试开发一个 framework，把这些细节抽象，对用户隐藏，而用户只需要编写 Map、Reduce 函数，然后使用 MapReduce 的 library 执行计算即可。

## 执行

另一方面，通过定义的分区方法（用户可以通过参数定义，比如 **hash(key) mod R** ），Map 输出的数据会根据 Key 分区成 R 份（这个R同样也可以通过参数定义）。

具体执行步骤如下：

1. MapReduce 框架会负责从输入文件（一般是保存在GFS上）读取数据然后产出 k/v 对，并把它们分区，一般切分成M份，每份 16MB ~ 64MB（用户可以通过参数定义），然后克隆出多份副本，分别分发给集群里多台机器；

2. 其中一份副本会分发给 master，其他即是负责该任务的 worker。因为分别有 M 份和 R 份数据，所以自然也就有这么些任务数，master 会负责分配空闲 worker 对应的任务；

3. 分配了 map 任务的 worker，会从相应的 input split 读取数据。它会把输入数据解析成 k/v 对，然后遍历每对数据，执行用户定义的 Map 方法。最终把执行了 Map 方法后的中间 k/v 数据存到内存；

4. 缓存的中间数据会被定期地写入到本地磁盘，并且根据分区函数（上文提到的 R 里面的分区方法）分到对应的 R 区域。这些存储到本地磁盘的地址会被传回给 master，然后最终会被 master 发给对应的 reduce worker；

5. 当 reduce worker 收到 master 发来的这些分区好的 R 分片数据的位置信息时，它会使用 RPC 从 map worker 那里读取缓存了的数据，**在读完所有中间数据后**，reduce worker 会将数据排序，这样相同的 key 会被分到一起。因为一般同一个 reduce worker 会收到相当多不同 key 的数据，因此排序是必要的，如果中间数据放内存里很大的话，可能还需要执行 **外部排序**；

6. reduce worker 会遍历排序好的中间数据，然后针对每一个遇到的唯一的中间 key ，它会将该 key 及对应的数据集传给用户的 Reduce 函数。Reduce 函数的输出会被追加到该 reduce 分区下的一个最终输出里；

7. 当所有 map 和 reduce 任务完成后，master 会唤醒用户程序。MapReduce 的调用会 return 给用户程序。

## master 做的事情

master 会维护每个 map\reduce task 的状态（idle、in-progress、completed），以及运行任务的每个worker节点的身份。

对于每个完成的 map 任务，master 会保存 map 任务产出的 R 中间文件的区域位置及大小等信息。完成时会通知正在处理 reduce 任务的 worker。

## 错误容忍

### 1、Worker 故障

master 会定期 ping 每个 worker 节点。如果在一个确定的时间里没有响应的话，master 会把该 worker 标记为失败。任何已经完成了的 map 任务会被重置为 idle（因为结果保存在那台故障节点的本地磁盘上，如果是保存到了 GFS 的话就不用重新分配重新跑了...），随后会被重新调度给其他 worker 节点，类似地，进行中的 map、reduce 任务也会被重置，重新分配给其他 worker 节点。

即便有大规模 worker 节点故障，MapReduce 会重新执行那些故障节点上的任务，然后尝试不断取得进展，直到最终完成任务。

### 2、Master 故障

master 端存储的状态数据（如当前分配的任务信息等）会定期写入 checkpoint 。随后 master 的新副本可以从 checkpoint 恢复数据启动。如果 master 挂了，客户端可以检查这个情况然后后面重试。

### 3、存在故障的一些语境

1、MapReduce 依赖于 map\reduce 任务输出的原子性提交来实现一致性；

2、每个处理中的任务会维护输出到一个私有的临时文件，reduce 会产出一份，map 会产出 R 份这样的文件（每个对应一个 reduce 任务），如果有 map 任务完成了，那个完成 map 任务的 worker 会发消息给 master，然后带上 R 份临时文件的信息，如果第一次收到它会记录这 R 份文件信息，如果 master 已经收到过它会忽略该信息；当一个 reduce 任务完成任务时，reduce worker 会原子性地将该临时输出文件重命名为最终的输出文件，如果执行过多次，那可能会有多份最终输出，MapReduce **通过底层文件系统提供的原子性重命名操作**来实现只有一份产出数据；

3、如果用户执行任务一个M可能产出的R每次执行不一样的话，不同的 map 执行可能会被不同的 R 任务执行产出不同的结果。

## 地区性

MapReduce 的核心瓶颈在网络带宽（ MIT 6.824 的讲师说 2020 年的情况大有改善，网络环境不再是之前核心交换这样的有带宽瓶颈的架构了）。GFS 把每个文件切成 64MB 的块，然后每块都会保存多份副本（一般是3）到不同的机器上。master 会把输入文件的地理位置信息考虑在内，尽量把 map 任务调度到有该输入数据副本的机器上。失败了的话，会尝试调度到离输入数据近的地方（比如同一个网络交换机下）。在一个拥有众多 worker 节点的集群里执行大型 MapReduce 操作时，绝大多数的输入数据是从本地读取的，然后不消耗网络带宽。

## 任务粒度

每个 map/reduce 任务对大概会占用1个byte的内存，总的状态信息占用内存会是 O(M*R) ，需要做的调度决策是 O(M+R)。

每份输入数据大小大概是 16MB 到 64MB（这样地区方面的优化也能做到最高效），并且分配的 R 会是机器数量的小几倍的量级。常常采取的配置是，M = 200,000，R = 5,000，worker 机器数量则是 2,000。

## 注意

1. 数据类型是可以由用户定义的，并不限制一定要是string；

2. reduce 的结果一般也是按照分区产出到对应的 R 输出文件，用户一般是不需要自行合并结果的，他们可以再把它们作为下一个 MapReduce 过程的输入数据；

3. 一旦发生故障后的重新调度的话，其他执行 reduce 任务的 worker 会收到这个信息的通知，并且会转向从代替的节点读取数据；

## 疑问

Q1. 这么大的数据排序也会是一个问题，怎么解决的呢？

Q2. map-reduce 的模式，尽管 map 会分发到各台机器，是分片了的，但是最终相同的 key 仍然会需要交给 reduce worker 来计算处理，这块是怎么优化性能的呢？

Q3. Map 产出的数据保存到本地磁盘这个过程，如果有数据损坏怎么办？是否是保存到了 GFS ？如果是的话，是否用到了临时文件避免部分写的情况？

Q4. 有哪些常见的 R partition 方法？

Q5. Map 数据的分区一般是按照什么规则去做的？

Q6. Reduce worker 怎么知道什么时候 ready 去跑了？换句话说，怎么知道什么时候 map 完成了一个 R 分区所有数据的执行？

TODO：

* 3.6 Backup Tasks
* 4 Refinements
* 4.1 Partitioning Function
* 4.2 Ordering Guarantees
* 4.3 Combiner Function
* 4.4 Input and Output Types
* 4.5 Side-effects
* 4.6 Skipping Bad Records
* 4.7 Local Execution
* 4.8 Status Information
* 4.9 Counters
* 5 Performance
* 5.1 Cluster Configuration
* 5.2 Grep
* 5.3 Sort
* 5.4 Effect of Backup Tasks
* 5.5 Machine Failures
* 6 Experience
* 6.1 Large-Scale Indexing
* 7 Related Work
* 8 Conclusions
* Acknowledgements
* Appendix
