# lotus存储管理调度
### 版本
以下分析基于版本 v1.10  

### 代码

管理代码位于[manager.go](https://github.com/filecoin-project/lotus/blob/v1.1.0/extern/sector-storage/manager.go#L65)，调度代码位于[sched.go](https://github.com/filecoin-project/lotus/blob/v1.1.0/extern/sector-storage/sched.go#L53)

miner通过[scheduler](https://github.com/filecoin-project/lotus/blob/v1.1.0/extern/sector-storage/sched.go#L53)选择合适的worker，将需要执行的任务分配给worker执行。这些任务包括

```
const (
	TTAddPiece   TaskType = "seal/v0/addpiece"
	TTPreCommit1 TaskType = "seal/v0/precommit/1"
	TTPreCommit2 TaskType = "seal/v0/precommit/2"
	TTCommit1    TaskType = "seal/v0/commit/1" // NOTE: We use this to transfer the sector into miner-local storage for now; Don't use on workers!
	TTCommit2    TaskType = "seal/v0/commit/2"

	TTFinalize TaskType = "seal/v0/finalize"

	TTFetch        TaskType = "seal/v0/fetch"
	TTUnseal       TaskType = "seal/v0/unseal"
	TTReadUnsealed TaskType = "seal/v0/unsealread"
)
```

### 概念

- [schedWindow](https://github.com/filecoin-project/lotus/blob/v1.1.0/extern/sector-storage/sched.go#L107)：一个需要处理的任务请求列表，这些任务接受一样的资源（CPU、GPU、内存范围）。一个schedWindow只会分配给一个worker处理。
- [workerHandle](https://github.com/filecoin-project/lotus/blob/v1.1.0/extern/sector-storage/sched.go#L79): 负责执行管理worker、创建schedWindow和将已经分配给worker的任务调用相应的worker执行。
- [activeWindows](https://github.com/filecoin-project/lotus/blob/v1.1.0/extern/sector-storage/sched.go#L90)： 每个workerHandle拥有一个activeWindows，存放已经分配的schedWindow。workerHandle的工作就是从这里取出每个schedWindow中的每个任务调用worker执行。
- [scheduler](https://github.com/filecoin-project/lotus/blob/v1.1.0/extern/sector-storage/sched.go#L53)：根据一定的算法选择合适的worker，放任务放入worker对应schedWindow中。
- [WorkerSelector](https://github.com/filecoin-project/lotus/blob/v1.1.0/extern/sector-storage/sched.go#L47): 定义了抽象的接口，有两个函数，其中`Ok`用于判断一个worker是否可以执行某个任务。`Cmp`用于比较两个worker的优先级。

### 调度逻辑

#### 协程1
- 当一个任务调度进来后，创建一个[workerRequest](https://github.com/filecoin-project/lotus/blob/v1.1.0/extern/sector-storage/sched.go#L171)， 放入channel，等待结果。

#### 协程2
 - 从channel中获取workerRequest，放入[队列](https://github.com/filecoin-project/lotus/blob/v1.1.0/extern/sector-storage/sched.go#L238)。
 - 从队列里面[取出workerRequest](https://github.com/filecoin-project/lotus/blob/v1.1.0/extern/sector-storage/sched.go#L340), 遍历所有worker的schedWindow，记录满足条件的worker，这些选中的worker都是有能力执行该任务的。
 - 再用rand.Shuffle打乱顺序，再排序。排序过程中用到了上面的WorkerSelector的Cmp函数。排在越靠前的schedWindow优先级最高。
 - 按优先级将任务放入schedWindow。
 
#### 协程3
- workerHandle从[activeWindows取出任务调用worker执行](https://github.com/filecoin-project/lotus/blob/v1.1.0/extern/sector-storage/sched.go#L597)




















