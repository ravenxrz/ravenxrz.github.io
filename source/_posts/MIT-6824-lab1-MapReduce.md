---
title: MIT-6824-lab1-MapReduce
categories: MIT6824
date: 2022-02-13 14:50:31
tags:
---

## 0. 前言 

从这个系统开一个新坑 -- MIT-6824， 分布式系统。早就将这门课纳入了必学的课程规划中，因为从学习编程到现在，我所学过的东西都是单机的，而这门课也算是和 CMU15445, MIT6828等知名课程的同级别课程，于是选中了本门课。

本次课程选用的是，MIT6824 Spring 2020。

> 杂谈:由于本门课程的所有lab均采用golang实现，所以花了几天的时间学习了golang，还好之前对C、C++语言比较熟悉，golang学习起来基本没有什么压力，在整个lab实现过程中，我发现我非常喜欢这门语言，在语言语法，语言工具链，调试工具等方面都做得非常好。唯一让我迷惑了一段时间的是go语言的包管理，虽然现在统一采样了module来管理，但是包括部分tutorial和mit6824的lab都没有基于最新的module来做，所以这需要花费时间了解下go语言的包管理历史。

ok，废话不多说。本文主要是对lab1的一个实现记录与总结。

<!--more-->

## 1. lab1环境搭建

详细的搭建过程参考: http://nil.csail.mit.edu/6.824/2020/labs/lab-mr.html

我使用vscode+golang extension来编程，按照参考中所说搭建后，遇到了代码依然跑不起来的情况。 这主要就是golang的包管理问题。有两种解决方案：

1. module方式
2. GOPATH方式

### 1. module方式

在src目录下，执行 `go mod init mit6824`, 会得到一个 `go.mod` 文件，然后将所有源文件中出现了类似 `import "../xx"` 的 地方，都采用 `import "mit6824/xx"`代替，接着执行参考中提到的运行代码即可。

```shell
$ cd ~/6.824
$ cd src/main
$ go build -buildmode=plugin ../mrapps/wc.go
$ rm mr-out*
$ go run mrsequential.go wc.so pg*.txt
$ more mr-out-0
```

这种方式是比较推荐的，但是我在实际使用过程中，会发现 vscode的golang插件会时不时抽风，没有代码提示和静态语法检查。于是改用了方式2.

### 2. GOPATH 方式

关于 GOPATH 是什么，如何使用这种旧方案管理项目，可以参考： [这里](https://www.digitalocean.com/community/tutorials/understanding-the-gopath)

我的做法是，将整个 `mit6824` 目录移动到 `$GOPATH/src`目录下，并把去掉了 `mit6824/src`目录，最终的文件组织结构如下：

```shell
.
└── mit6824
    ├── kvraft
    ├── labgob
    ├── labrpc
    ├── main
    ├── models
    ├── mr
    ├── mrapps
    ├── porcupine
    ├── raft
    ├── shardkv
    └── shardmaster
```

此时，同样需要将所有源文件中出现了类似 `import "../xx"` 的 地方，都采用 `import "mit6824/xx"`代替。同时改动 go env 中的 GO111MODULE为"auto"或者"off"。

至此，可以正确运行 `mrsequential.go` 

## 2. lab1题解

### 1. 过程分析

lab1需要我们手动实现一个简单的 MapReduce。 MapReduce由 Jeffrey Dean 和 Sanjay Ghemawa 两位超级巨佬提出的，推荐看一下paper，写得浅显易懂。

下图为MapReduce的架构图：

![image-20220213190550211](https://cdn.JsDelivr.net/gh/ravenxrz/PicBed/img/image-20220213190550211.png)

用红框标出来的就是本次实验我们需要实现的：

1. Master 进程，master是整个MapReduce task的管理器，管理着整个task的元数据，包括每个task下job的状态，下发任务到map worker或者reduce worker（但本lab中是由worker来主动请求task，再由master分配Task）
2. Worker进程，worker进程可以同时充当mapper或reducer，首先是Map Phase，mapper根据master传递过来的task执行map函数，生成 intermediate files在本地上，然后将这些intermediate files的位置信息返回到master， 当所有的mapper task均完成后，整个MapReduce Task进入Reduce Phase，Master下发reduce task到worker中，各个reducer根据这些tasks产出最终的output file。

**一些需要注意的点：**

1. 由于存在多个Worker进程同时向Master请求task，所以Master内部需要一些保证线程安全的操作，如mutex
2. Master需要感知每个下发下去的Task是否已经超时，如果超时可以认为该task对应worker已经die掉了，需要重新分发任务。 对于每个task来说，执行超过10s就认为该task已经超时，需要重新分配
3. 每个Map Worker都可能产出多个intermediate files， 具体来说，map worker根据 `hash(key) % nReducer` 来确定每个key该存放在哪个文件。 intermediate file的命名格式为 `mr-X-Y`, 其中X是map worker的id， Y是 hash % nReducer后的数字（其实也是后期reducer的id)。 另外， intermediate files中的 K/V 存储格式可以按照自己喜欢的方式序列化，lab原文中采用的json序列化的方式。
4. 任务执行完成后，Master能够主动退出，Worker在无法联系到Master或者从Master中获取到需要退出的信号后，直接退出
5. Worker 有时候是需要wait的，比如，当master中的所有map task已经分发下去，但是却没有完成的时候，又有worker来请求任务，此时master可以分发一个 `wait` 类型的特殊任务，worker接收这种类型的任务后，主动进入到wait阶段，等待一段时间后重新请求（可能此时所有的map tasks均已完成，master可以下发reduce类型的任务）
6. 为了保证file content的完成性，避免因为某个worker crash，后续读取到部分文件 content，在写入文件的时候，应该先写入到一个临时文件，然后重命名该文件，这其实依赖file system需要保证rename的过程是原子的。

**最后谈一下如何测试，** lab1提供了 `test-mr.sh`测试文件， 直接执行即可。

下面简单谈谈我个人的实现。

### 2. 数据结构

Task相关，可以同时表示 Map Task，Reduce Task以及特性的Wait Task。

```go
// Add your RPC definitions here.
type StateType uint8
type WorkerTaskType uint8

// Task State
const (
	IDLE StateType = 1 << iota
	IN_PROGRESS
	COMPLETED
)

// Task Type
const (
	MapTask WorkerTaskType = 1 << iota
	ReduceTask
	WaitTask // special task used when all task have been assigned
)

type Task struct {
	TaskId       int
	TaskType     WorkerTaskType
	PartitionNum int // input file for map should be split into # partitions
	StartTime    int64
	FileNames    []string
	TaskState    StateType
}
```

RPC相关：

1. 当worker请求一个task，无需入参，只用master返回一个Task
2. 当worker完成一个task，需要一些入参，这些定义如下

```go
type MapTaskDoneArg struct {
	TaskId                int
	IntermediateFileNames []string
}

type ReduceTaskDoneArg struct {
	TaskId int
}
```

Master相关， Master是管理了整个MapReduce过程中的大部分元数据，相对复杂一些：

```go
type ExecStateType StateType

// master execution state
const (
	MAP_PHASE ExecStateType = 1 << iota
	REDUCE_PHASE
	COMPLETE_ALL
)

type Master struct {
	// Your definitions here.
	mapTasks               []Task
	reduceTasks            []Task
	intermidateFileNames   map[int][]string // used for reduce task. key: partition id; value: file names in that partitiion
	completedMapTaskNum    int              // should be end with # = len(mapTask). i.e assert(completedMapTaskNum <= len(mapTask))
	completedReduceTaskNum int              // should be end with # = len(reduceTask).
	partitionNum           int
	execState              ExecStateType
	mu                     sync.Mutex
}
```

### 3. Master端

Master的主流程非常简单，有worker request来，则分发一定的task，有worker完成了任务，根据一定条件转换状态。

1. 请求相关

```go
// Your code here -- RPC handlers for the worker to call.
func (m *Master) RequestTask(meaningless *struct{} /* not use */, task *Task) error {
	m.mu.Lock()
	defer m.mu.Unlock()

	switch m.execState {
	case MAP_PHASE:
		*task = m.assignMapTask()
	case REDUCE_PHASE:
		*task = m.assignReduceTask()
	case COMPLETE_ALL:
		// return error intentionally
		return fmt.Errorf("all tasks have been completed")
	}
	return nil
}
```

 	2. task完成相关

```go

func (m *Master) MapTaskDone(arg *MapTaskDoneArg, meaningless *struct{}) error {
	m.mu.Lock()
	defer m.mu.Unlock()

	if m.execState != MAP_PHASE {
		log.Panic("current exec state is not at the map phase, current state is", m.execState)
	}
	// check whether the mapper is assumed timeout/die
	if arg.TaskId >= len(m.mapTasks) || m.mapTasks[arg.TaskId].TaskState != IN_PROGRESS {
		msg := fmt.Sprintf("task id error or task has completed, task id:%v, task state:%v", arg.TaskId, m.mapTasks[arg.TaskId].TaskState)
		log.Panic(msg)
	}

	// Split intermidate FileNames
	for _, fileName := range arg.IntermediateFileNames {
		// fileName looks like "map-X-Y", where Y is the partition id
		if fileName == "" {
			log.Panic("MapTaskDone: intermediate filename should not be empty")
		}
        partitionStr := string(fileName[len(fileName)-1])  // TODO: is there any better way to convert a char to int?
		partitionId, err := strconv.Atoi(partitionStr)
		if err != nil {
			log.Fatalf("convert string %v to int failed", partitionStr)
		}
		if m.intermidateFileNames[partitionId] == nil {
			m.intermidateFileNames[partitionId] = make([]string, 0)
		}
		m.intermidateFileNames[partitionId] = append(m.intermidateFileNames[partitionId], fileName)
	}
	m.mapTasks[arg.TaskId].TaskState = COMPLETED
	m.completedMapTaskNum++
	if m.completedMapTaskNum == len(m.mapTasks) {
		// do one more check for debugging
		for i := 0; i < len(m.mapTasks); i++ {
			if m.mapTasks[i].TaskState != COMPLETED {
				log.Panic("completedMapTaskNum is equal to len of mapTasks, but there are some mapTasks still not in the `COMPLETED` state")
			}
		}
		log.Println("all map tasks have been completed")
		m.execState = REDUCE_PHASE
        // all intermediate files have been generated, so we can create reduce tasks now
		m.createReduceTask()
	}
	return nil
}

func (m *Master) ReduceTaskDone(arg *ReduceTaskDoneArg, meaningless *struct{}) error {
	m.mu.Lock()
	defer m.mu.Unlock()

	if m.execState != REDUCE_PHASE {
		log.Panic("current exec state is not at the reduce phase, current state is", m.execState)
	}
	// check whether the reduce is assumed timeout/die
	taskIdx := -1
	for i := 0; i < len(m.reduceTasks); i++ {
		if m.reduceTasks[i].TaskId == arg.TaskId {
			if m.reduceTasks[i].TaskState != IN_PROGRESS {
				log.Panic("task state is wrong, which should be `IN_PROGRESS`, task id:", arg.TaskId)
			}
			taskIdx = i
			break
		}
	}

	m.reduceTasks[taskIdx].TaskState = COMPLETED
	m.completedReduceTaskNum++
	if m.completedReduceTaskNum == len(m.reduceTasks) {
		// do one more check for debugging
		for i := 0; i < len(m.reduceTasks); i++ {
			if m.reduceTasks[i].TaskState != COMPLETED {
				log.Panic("completedMapTaskNum is equal to len of reduceTasks,  but there are some reduceTasks still not in the `COMPLETED` state")
			}
		}
		log.Println("all reduce tasks have been completed")
		m.execState = COMPLETE_ALL
	}
	return nil
}
```

另外，master需要检测worker是否超时，这里我单独开了一个协程。

```go
func (m *Master) taskTimeoutChecker() {
	const delta = 10 // use 10 seconds as limit
	for {
		time.Sleep(1 * time.Second) // sleep 1s as a rateLimiter
		now := time.Now().Unix()    // by seconds
		{
			m.mu.Lock()
			var tasks []Task
			switch m.execState {
			case MAP_PHASE:
				tasks = m.mapTasks
			case REDUCE_PHASE:
				tasks = m.reduceTasks
			case COMPLETE_ALL:
				m.mu.Unlock()
				return
			}
			for i := 0; i < len(tasks); i++ {
				if tasks[i].TaskState == IN_PROGRESS && now-tasks[i].StartTime >= delta {
					// reset state
					tasks[i].TaskState = IDLE
					break
				}
			}
			m.mu.Unlock()
		}
	}
}
```

最后，主程序如下：

```go
//
// create a Master.
// main/mrmaster.go calls this function.
// nReduce is the number of reduce tasks to use.
//
func MakeMaster(files []string, nReduce int) *Master {
	m := Master{
		mapTasks:               make([]Task, 0),
		reduceTasks:            make([]Task, 0),
		intermidateFileNames:   make(map[int][]string),
		completedMapTaskNum:    0,
		completedReduceTaskNum: 0,
		partitionNum:           nReduce,
		execState:              MAP_PHASE,
	}
	// create mapTasks
	for i, fileName := range files {
		task := Task{
			TaskId:       i,
			TaskType:     MapTask,
			PartitionNum: nReduce,
			StartTime:    -1,
			FileNames:    []string{fileName},
			TaskState:    IDLE,
		}
		m.mapTasks = append(m.mapTasks, task) // maybe I should use pointer instead to save one copy cost?
	}

	// launch timeout checker
	go m.taskTimeoutChecker()

	m.server()
	return &m
}
```

### 4. Worker端

worker也相对简单，只是分为map worker和reduce worker。每个worker都是向master请求一个task，完成该task后上报master即可。

主程序

```go
//
// main/mrworker.go calls this function.
//
func Worker(mapf func(string, string) []KeyValue,
	reducef func(string, []string) string) {

	for {
		task := &Task{}
		if ok := call("Master.RequestTask", &struct{}{}, task); !ok {
			break
		}
		if task.TaskType == MapTask {
			log.Println("assigned map task")
			doMap(mapf, task)
		} else if task.TaskType == ReduceTask {
			log.Println("assigned reduce task")
			doReduce(reducef, task)
		} else if task.TaskType == WaitTask {
			log.Println("assigned wait task")
			time.Sleep(2 * time.Second)
		} else {
			log.Fatalf("not suppported task: %+v", task)
		}
	}
}
```

这了贴一下 doMap和 doReduce的整体代码

```go
func doMap(mapf func(string, string) []KeyValue, task *Task) {
	// read each input file, pass it to Map,  accumulate the kvpairs Map output.   then sort it
	kvpairs := []KeyValue{}
	for _, filename := range task.FileNames {
		file, err := os.Open(filename)
		if err != nil {
			log.Fatalf("cannot open %v", filename)
		}
		content, err := ioutil.ReadAll(file)
		if err != nil {
			log.Fatalf("cannot read %v", filename)
		}
		file.Close()
		kva := mapf(filename, string(content))
		kvpairs = append(kvpairs, kva...)
	}
	sort.Sort(ByKey(kvpairs))

    // split into tmp files， tmpFile format: tmp-X-Y
	tmpFileNames := splitKVPairs(kvpairs, task)
	// rename files
	intermediateFileNames := make([]string, 0)
	for fileName, fileHandler := range tmpFileNames {
		fileHandler.Close() // close first
		newFileName := strings.Replace(fileName, "tmp", "map", -1)
		os.Rename(fileName, newFileName)
		intermediateFileNames = append(intermediateFileNames, newFileName)
	}
	// notify master we have done this task
	finishMapTask(task.TaskId, intermediateFileNames)
}

func doReduce(reducef func(string, []string) string, task *Task) {
	// read kv pairs from files
	rawKVs := []KeyValue{}
	for _, filename := range task.FileNames {
		file, err := os.Open(filename)
		if err != nil {
			log.Fatalf("cannot open %v", filename)
		}
		rawKVs = append(rawKVs, loadKVPairs(file)...)
		file.Close()
	}
	sort.Sort(ByKey(rawKVs))
	// apply reduce
	retKVs := applyReduce(reducef, rawKVs)

	// wirte ret kv pairs into tmp file to prevent client will read partial contents in the further for this worker crashed
	tmpFileName := fmt.Sprintf("tmp-out-%d", task.TaskId)
	file, err := os.Create(tmpFileName)
	if err != nil {
		log.Fatalf("cannot create temp file %v", tmpFileName)
	}
	for _, kv := range retKVs {
		fmt.Fprintf(file, "%v %v\n", kv.Key, kv.Value)
	}
	// rename file
	newFileName := strings.Replace(tmpFileName, "tmp", "mr", -1)
	os.Rename(tmpFileName, newFileName)
	// notify master that we have done this task
	finishReduceTask(task.TaskId)
}
```

## 3. 总结

这是mit6824的第一个lab，整体来说，难度不大，虽然只是一个MapReduce的简化版，但是通过做lab对MapReduce的内部原理了解得更为清晰，同时也是熟悉自己的golang语言。