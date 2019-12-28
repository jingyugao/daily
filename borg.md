# Borg
Borg是一套集群管理系统。通过下面几个功能实现了资源的高利用率。
* `combining admission control`，组合访问控制（权限系统模型）
* `task-pack`，
* `over-commitment`，过载使用，允许使用超过配额的资源。
* `process-level performance`, 没有VM，速度快。
另外提供了故障恢复，调度策略来减少关联失败。(???) 
为了使用方便，提供了一个简单的任务描述语言，命名服务整合，实时任务监控。和系统行为分析与模仿的工具。

## Introduction
Borg提供三个优点
1. 隐藏了资源管理和故障处理的细节。用户可以关注与应用开发。
2. 高可靠，高可用。
3. 可以高效的跨越几万台机器跑任务。

## The user perspective
Borg的用户是谷歌开发人员和系统管理员(谷歌称之为SRE）。用户通过*jobs*的形式提交*work*给Borg，每个*job*包括多个*task*,
一个*job*下的*task*都是相同的*program*（二进制程序）。每个job跑在一个Borg的*cell*中。*cell*是一个集群单元。
```
                  work
         /  ........|.........      \
       job  :      job2      :     job3
            :    /   |   \   :
            :taskA  taskB  ..:
            :................:
              Cell
```                          
       
### The workload
Cell的workload分为异质的两部分。
* 在线部分。长时间运行，从不宕机，延迟敏感。例如gmail，googl docs等web服务。
* 离线部分。需要数天完成，短期内性能波动不敏感。
一个Cell混合这两种workload，取决于他们的major租户(有的cell就是用来跑密集的批处理任务的），而且会随着时间变化。
面向最终用户的任务是一直在跑，批处理任务来来去去。Borg要很好的处理好这些情况。
Borg把*job*按照优先级分为*prod*和*non-prod*.在线业务大多数是*prod*的，离线业务大多数是*non-prod*的。
一个典型的cell中，*prod jobs*分配70%的cpu资源并占据60%的cpu usage；分配55%的内存并占据85%的内粗。
分配和占据的含义参加下文。

### Clusters and cells
cell的机器属于一个*cluster*。一个*cluster*处于一个*datacenter building*（物理机房）。许多个物理机房组成一个*site*。
中等规模的cell包括10k台机器..........
总之谷歌的机器非常多。机房特别多。处理器种类多，各种硬件应有尽有。

### Jobs and tasks
一个*job*的属性包括`name`,`owner`和`task number`。*
总之谷歌的机器非常多。机房特别多。处理器种类多，各种硬件应有尽有。*job*可以约束*task*的一些特别的属性例如os版本，ip地址等。
一个*job*只在一个*cell*中运行。Borg没有使用VM机制，没有*virtualization*的开销。因为设计borg的时候还没有硬件支持虚拟化。
job下的每个task的配置基本上都一样，但可以有不同的比如 command-line flag。
用户可以在*job*运行的时候改变部分或者全部*tasks*的属性。用户发布新的job configuration，然后使Borg更新tasks。这是轻量级的非原子的事务，结束之前可以撤销。更新是滚动的，可以限制更新过程中被中断(调度或抢占）的数量。如果某个变更导致数量超出，这个变更会被跳过。
一些更新需要任务重启。一些更新可能会使task不再适合机器，导致task停止和重新调度。有的更新可能不需要重启或者移动task。
### Allocs
Borg的*alloc*是某台机器上一组保留的资源配额，用来让一个或者多个task跑。这些资源无论是否被使用，都一直保留。
*Allocs*可以用于为未来的任务保留资源，*task*重启期间，这些资源依然保留。
  一个*alloc set*类似一个*job*。是一组在不同机器上的*alloc*。
 
### Priority, quota, and admission control
当任务过多怎么办。对应的策略是*priority*和*quota*(优先级和限额）。
   每个job都有一个*priority*。高优先级的task可以通过牺牲低优先级的task来获得资源，即使稍后要被抢占或者杀死。
Borg为不同的用途定义了不重叠的优先级段。包括*monitoring*，*prod*，*batch*和*best effort*（测试或者无要求的任务）
  尽管一个被抢占的*task*通常会在*cell*的其他地方继续运行，但是有可能发生*preemption cascades*。一个高优先级的task抢占了较低的task，这个task又抢占了另一个更低的task。为了降低这种情况，禁止*prod*优先级的任务相互枪战。
  *Priority*对task来说相当重要，决定task运行还是等待。*Quota*用于决定哪个job被调度。*Quota*的表达形式是某个priiority对应的资源权重向量。权重决定了一个job一段时间内可以请求的最大资源。*quota-checking*是权限控制的一部分。*quota*不足的job会在提交时被立刻拒绝。
  高优先级的quota花费比低优先级的多。*prod-priority quota*受限于cell实际可用的资源。如果资源足够，那么prod-job就会投入运行。我们鼓励用户购买超额资源，许多用户买超额资源是为了应对规模增长。我们通过超额出售lower-priority的quota来应对。
  Quota分配是独立于borg之外的，跟物理容量有关，不同的数据中心有不同的价格和容量。用户的jobs只有配额足够的时候才可以运行。
 
### Naming and monitoring
Borg提供了“Borg name service”（BNS），包括cell名，job名，和task序号。Borg把task的hostname和port写入Chunbby。例如某个用户ubar在cell cc的job jfoo的第50个task，可以通过"50.jfoo.ubar.cc.borg.google.com"访问。Borg还写了job size和task健康信息，用来提供负载均衡信息。




Reference：
https://pdos.csail.mit.edu/6.824/papers/borg.pdf
https://www.lagou.com/lgeduarticle/74124.html



