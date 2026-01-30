## 第一讲 Introduction

  ### 课程预览

  #### 为什么要 Distbution system ?   Answer高性能

  - Parallelism
  - Fault Tolerance
  - Physical security / isolated

  #### Infrastructure 

  - Storage
  - Computing
  - Communication

  目标是 隐藏细节， 使得分布式系统像一个单机系统

  #### Challenge 挑战

  - Concurrency

    强一致性 (Strong Consistency)

    ​	读取操作总是能返回最近一次写入的值。

    弱一致性 (Weak Consistency)

    ​	不保证读取立即看到最新的写入，可能会读到旧数据（Stale Data）。

  - Performance

  - Partial Failure

  ### MapReduce

  #### 1. Map 阶段 (Map Phase)

  **输入：** 原始输入文件（Input Files），通常切分为 $M$ 个 Split。**操作：** 用户编写的 `Map` 函数对每一条记录进行逻辑处理。**输出：** 中间结果（Intermediate）的 **$\langle K, V \rangle$** 对。

  > **板书重点：** Map 的输出并不会立即写入分布式文件系统（GFS），而是缓存在内存中，并定期溢写到 Map Worker 的**本地磁盘**。

  #### 2. Shuffle 阶段 (Shuffle Phase)

  这是系统自动完成的中间过程，其目的是将 **“所有 Map 输出的相同 Key”** 汇聚到同一个 Reduce 任务中。

  - **输入：** 各个 Map Worker 本地磁盘上的 **$\langle K, V \rangle$**。
  - **操作：** 

    1. **Partitioning (分区)：** Master 告诉 Map Worker 如何根据 Key 分组（常用 $hash(Key) \pmod R$）。
    2. **Sorting & Grouping (排序与分组)：** 相同 Key 的所有 Value 被收集在一起。
    3. **Network Transfer (数据拉取)：** Reduce Worker 通过 RPC 从各个 Map Worker 的本地磁盘拉取（Pull）属于自己的数据。
  - **输出：** 聚合后的 **$\langle K, [V_1, V_2, V_3, \dots] \rangle$**。

  #### 3. Reduce 阶段 (Reduce Phase)

  - **输入：** Shuffle 阶段生成的按 Key 分组的列表 **$\langle K, [V_1, V_2, V_3, \dots] \rangle$**。

  - **操作：** 用户编写的 `Reduce` 函数对 Value 列表进行迭代处理（例如求和、连接等）。

  - **输出：** 最终的 **$\langle K, \text{OUTPUT} \rangle$**。

  > **板书重点：** 每个 Reduce 任务会产生一个对应的输出文件（如 `res-0, res-1,...`），这些文件最终存储在分布式文件系统（GFS）中。

  

  #### 总结：数据流转换图

  $$\text{Input} \xrightarrow{\text{Map}} \langle K, V \rangle \xrightarrow{\text{Shuffle (Sort/Partition)}} \langle K, [V, \dots] \rangle \xrightarrow{\text{Reduce}} \text{Output}$$

  

  #### 常见面试/考试考点：

  - **谁负责启动 Reduce？** Master 节点。当所有 Map 任务完成后，Master 通知 Reduce Worker 开始执行。
  - **中间数据存哪？** Map 的中间结果存**本地磁盘**（Local Disk），Reduce 的结果存**分布式系统**（GFS）。
  - **为什么需要 Shuffle？** 因为分布式计算中，属于同一个 Key 的数据可能散落在全球/全机房的几千台 Map 机器上，必须通过网络重排（Shuffle）才能交给同一个 Reduce 函数处理。

  ```
  Map 机器 1 (本地磁盘) ----\                  /--> Reduce 机器 1 (处理 Key A-M)
                           \     Shuffle    /
  Map 机器 2 (本地磁盘) -------- 网络传输 --------> Reduce 机器 2 (处理 Key N-Z)
                           /     (HTTP/RPC) \
  Map 机器 3 (本地磁盘) ----/                  \--> Reduce 机器 3 (处理其他...)
  ```

  课程高价值的是 Lab。  这一章的作业是，Go语言实现的MapReduce

  文章阅读：  MapReduce
