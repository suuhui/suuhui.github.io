## MapReduce论文

![Execution overview](/source/images/mapreduce-execution.jpeg)

Map调用通过自动将输入拆分为M组运行在不同的机器上。输入拆分可以并行运行在不同的机器上。Reduce调用通过一个分区函数将中间键拆分为R块实现分布式。分区函数和分块数R都由用户指定。MapReduce执行流程如下描述：
1. MapReduce库将输入划分成典型的16或64MB（用户通过可选参数控制），然后在一组机器上启动MapReduce进程
2. 其中，一个master进程，其余为worker进程。共有M个map任务和R个reduce任务。master挑选空闲的worker分配map或reduce任务
3. 被指定了map任务的worker从相应的文件中读取。它从输入中解析kv对，并将每一对传给用户自定义的Map函数。Map函数产出的中间kv对缓冲在内存中
4. 缓冲对定时写入本地磁盘，通过分区函数分成R块。本地磁盘上缓冲对的位置发送个master，master负责将其转发给reduce worker
5. 当一个reduce worker被master通知后，它使用rpc从map worker的机器上读取缓冲对。当读取完所有的中间键，reduce worker会将其排序，这样所有相同的key会聚合在一起。由于许多不同的key会路由到相同的reduce任务，因此排序是必须的。如果中间kv对过大，内存不足以容纳，则需要外部排序。
6. reduce worker遍历排序后的中间数据中的每个唯一键，将其传给用户自定义的reduce函数，该函数的输出被追加到该部分数据的最终输出文件中
7. 当所有的map和reduce任务结束后，唤醒用户程序。此时MapReduce调用将返回给用户程序

## 任务

实现一个分布式MapReduce，包含两个程序coordinator和worker。只有一个coordinator进程和多个worker进程并行执行。真实系统中，worker进程运行在不同的机器上，而这个实验中，你可以在一个机器上运行。worker通过RPC与coordinator进行通信。每个worker进程向coordinator索要任务，从一个或多个文件中读取任务的输入，执行任务，将结果输出到一个或多个文件中。coordinator应该检查worker是否在合理的时间内（此实验为10s）完成任务，否则将这个任务给其他的worker执行。

我们提供了少量代码让你开始。coordinator和worker的主协程代码分别在`main/mrcoordinator.go`和`main/mrwrokder.go`；不要修改这些代码，你应该把你的实现放在`mr/coordinator.go`、`mr/worker.go`和`mr/rpc.go`。

规则：

- map阶段应该把中间键分成`nReduce`个桶，`nReduce`是reduce任务数，由参数指定
- worker实现应该将第X个reduce任务的输出写入`mr-out-X`文件
- 每个reduce函数的输出对应`mr-out-X`文件中的一行。该行应该将key和value使用GO的`%v %v`格式化生成。可以参考`main/mrsequential.go`中"this is the correct format"的注释。如果不是这个格式，测试脚本将会失败。
- worker应该将中间Map输出到当前文件夹中的文件，以便worker后续将其作为reduce任务的输入
- `main/mrcoordinator.go`期望`mr/coordinator.go`实现`Done()`方法，当MapReduce任务完全完成时，返回true；此时`mrcoordinator.go`将会退出
- 当一个MapReduce任务完全完成，worker进程应对退出。一个简单的实现方式是，在调用中return：如果worker与coordinator无法正常通信，可以假设MapReduce任务已经完成，因此worker也可以终止。取决于你的实现，可能会发现，coordinator给worker一个伪任务`please exit`会很有用。