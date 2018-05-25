
#JStorm Worker Scheduling

@(学习笔记)[Storm]
目录

[TOC]
## 基础概念
- 拓扑
- 组件
    - Spout
    - Bolt
- Nimbus vs Supervisor
- Worker, Executor, Task.

不同的标识符:
- topology id:
```
// NimbusHandler.submitTopologyWithOpts()
int counter = data.getSubmittedCount().incrementAndGet();
topologyId = Common.topologyNameToId(topologyName, counter);
```
- supervisor id:
```
supervisorId = UUID.randomUUID().toString();
```
- task id:
```
public static Map<Integer, TaskInfo> mkTaskInfo(Map<Object, Object> stormConf, StormTopology sysTopology, String topologyid) {
    // use TreeMap to make task as sequence
    Map<Integer, TaskInfo> rtn = new TreeMap<Integer, TaskInfo>();
    Integer count = 0;
    count = mkTaskMaker(stormConf, sysTopology.get_bolts(), rtn, count);
    count = mkTaskMaker(stormConf, sysTopology.get_spouts(), rtn, count);
    count = mkTaskMaker(stormConf, sysTopology.get_state_spouts(), rtn, count);
    return rtn;
}

for (int i = 0; i < parallelism; i++) {
    cnt++;
    TaskInfo taskInfo = new TaskInfo((String) entry.getKey(), componentType);
    rtn.put(cnt, taskInfo);
}
```
## 1. 整体逻辑
1. Supervisor启动时注册Supervisor信息到zookeeper并定时更新。
2. 用户提交topology, 触发Nimbus分配worker和task.
## 2. 资源注册
### Supervisor信息注册
涉及到的关键类:
- `com.alibaba.jstorm.daemon.supervisor.Supervisor`
- `com.alibaba.jstorm.daemon.supervisor.Heartbeat`
- `com.alibaba.jstorm.daemon.supervisor.SupervisorInfo`

关键数据结构:
```java
public class SupervisorInfo implements Serializable {
    private final String hostName; // 为机器ip或者`supervisor.hostname`中设置的值。
    private final String supervisorId;
    private Integer timeSecs;
    private Integer uptimeSecs;
    private Set<Integer> workerPorts; // 一般为`supervisor.slots.ports`, 指定了`supervisor.enable.auto.adjust.slots`时例外
}
```
JStorm在启动Supervisor进程时，同时会启动一个线程，用于定时向zookeeper更新自身的Supervisor信息。这个线程由`Heartbeat`类实现。在`Supervisor`类中可以看到其启动代码:
```java
Heartbeat hb = new Heartbeat(conf, stormClusterState, supervisorId, localState, checkStatus);
hb.update();
AsyncLoopThread heartbeat = new AsyncLoopThread(hb, false, null, Thread.MIN_PRIORITY, true); // 在构造函数中启动线程
threads.add(heartbeat); // 对线程统一管理
```
Heartbeat类的`update`方法完成信息的更新:
```java
public void update() {
    supervisorInfo.setTimeSecs(TimeUtils.current_time_secs());
    supervisorInfo.setUptimeSecs((int) (TimeUtils.current_time_secs() - startTime)); // startTime中构造函数中取当前系统时间

    if (!checkStatus.equals(oldCheckStatus)) {
        if (checkStatus.getType() == MachineCheckStatus.StatusType.warning
                || checkStatus.getType() == MachineCheckStatus.StatusType.error
                || checkStatus.getType() == MachineCheckStatus.StatusType.panic) {
            Set<Integer> ports = new HashSet<Integer>();
            supervisorInfo.setWorkerPorts(ports);
            LOG.warn("due to no enough resourse, limit supervisor's ports and block scheduling");
        } else {
            updateSupervisorInfo();
        }
        oldCheckStatus.SetType(checkStatus.getType());
    } else {
        updateSupervisorInfo();
    }
    stormClusterState.supervisor_heartbeat(supervisorId, supervisorInfo);
}
private void updateSupervisorInfo() {
    // 如果没有设置supervisor.enable.auto.adjust.slots的话，就是supervisor.slots.ports中设置的那些端口。如果设置了前者，或者是后者为空，则又有一套特殊逻辑来自动分配slots。
    List<Integer> portList = calculatorAvailablePorts();
    Set<Integer> ports = JStormUtils.listToSet(portList);
    supervisorInfo.setWorkerPorts(ports);
}
```
每次更新做了以下事情:
- 设置当前supervisor的运行时长，更新时间戳。
- 检查当前机器的状态，如果状态不佳，比如遇到错误、故障，则清空该Supervisor的worker ports. 这样，nimbus将不再分配任务到该机器。
- 设置当前supervisor的worker ports.

## 3. 资源的分配

Nimbus Server对外提供Thrift服务，以便用户提交和管理topology以及查询集群状态。Thrift 服务由`ServiceHandler`类实现。用户通过`submitTopology`接口提交topology。在Nimbus内部，提交topology的逻辑是由`submitTopologyWithOpts`函数完成的，其主要伪代码如下:
```java
// create /local-dir/nimbus/topologyId/xxxx files
setupStormCode(conf, topologyId, uploadedJarLocation, stormConf, normalizedTopology);

// generate TaskInfo for every bolt or spout in ZK
// /ZK/tasks/topoologyId/xxx
setupZkTaskInfo(conf, topologyId, stormClusterState);

// make assignments for a topology
LOG.info("Submit for " + topologyName + " with conf " + serializedConf);
makeAssignment(topologyName, topologyId, options.get_initial_status());

// when make assignment for a topology,so remove the topologyid form
// pendingSubmitTopologys
data.getPendingSubmitTopoloygs().remove(topologyId);

// push start event after startup
this.data.getMetricRunnable().pushEvent(startEvent);

notifyTopologyActionListener(topologyName, "submitTopology");
```
可以看到，任务的分配是在`ServiceHandler.makeAssignment()`函数中执行的。这个函数创建一个`TopologyAssignEvent`事件，并交给`TopologyAssign`类来执行。
```java
private void makeAssignment(String topologyName, String topologyId, TopologyInitialStatus status) {
	TopologyAssignEvent assignEvent = new TopologyAssignEvent();
	assignEvent.setTopologyId(topologyId);
	assignEvent.setScratch(false);
	assignEvent.setTopologyName(topologyName);
	assignEvent.setOldStatus(Thrift.topologyInitialStatusToStormStatus(status));

	TopologyAssign.push(assignEvent);
	boolean isSuccess = assignEvent.waitFinish();
	if (isSuccess == true) {
	    LOG.info("Finish submit for " + topologyName);
	} else {
	    throw new FailedAssignTopologyException(assignEvent.getErrorMsg());
	}
}
```
`TopologyAssign`继承自`Runnable`接口，是一个持续运行的线程: 它维护一个并发队列，接收`TopologyAssignEvent`事件，并执行任务分配:
```java    
protected boolean doTopologyAssignment(TopologyAssignEvent event) {
    Assignment assignment;
    Assignment oldAssignment = null;
    boolean isReassign = event.isScratch();
    if (isReassign) {
        oldAssignment = nimbusData.getStormClusterState().assignment_info(event.getTopologyId(), null);
    }
    assignment = mkAssignment(event);

    // notify jstorm monitor on task assign/reassign/rebalance
    TaskStartEvent taskEvent = new TaskStartEvent();
    taskEvent.oldAssignment = oldAssignment;
    taskEvent.newAssignment = assignment;
    taskEvent.topologyId = event.getTopologyId();
    taskEvent.clusterName = nimbusData.getClusterName();
    taskEvent.timestamp = System.currentTimeMillis();

    Map<Integer, String> task2Component;
    // get from nimbus cache first
    Map<Integer, TaskInfo> taskInfoMap = Cluster.get_all_taskInfo(nimbusData.getStormClusterState(), event.getTopologyId());
    if (taskInfoMap != null) {
        task2Component = Common.getTaskToComponent(taskInfoMap);
    } else {
        task2Component = Common.getTaskToComponent(Cluster.get_all_taskInfo(nimbusData.getStormClusterState(), event.getTopologyId()));
    }
    taskEvent.task2Component = task2Component;
    nimbusData.getMetricRunnable().pushEvent(taskEvent);

    if (!isReassign) {
        setTopologyStatus(event);
    }

    if (assignment != null)
        backupAssignment(assignment, event);
    event.done();
    return true;
}
```
它实际上是对`mkAssignment`函数的一个封装:
```java
public Assignment mkAssignment(TopologyAssignEvent event) throws Exception {
    String topologyId = event.getTopologyId();
    LOG.info("Determining assignment for " + topologyId);
    TopologyAssignContext context = prepareTopologyAssign(event);
    Set<ResourceWorkerSlot> assignments = null;
    if (!StormConfig.local_mode(nimbusData.getConf())) {
        IToplogyScheduler scheduler = schedulers.get(DEFAULT_SCHEDULER_NAME);
        assignments = scheduler.assignTasks(context);

    } else {
        assignments = mkLocalAssignment(context);
    }
    Assignment assignment = null;
    if (assignments != null && assignments.size() > 0) {
        Map<String, String> nodeHost = getTopologyNodeHost(context.getCluster(), context.getOldAssignment(), assignments);
        Map<Integer, Integer> startTimes = getTaskStartTimes(context, nimbusData, topologyId, context.getOldAssignment(), assignments);
        String codeDir = StormConfig.masterStormdistRoot(nimbusData.getConf(), topologyId);
        assignment = new Assignment(codeDir, assignments, nodeHost, startTimes);
        //  the topology binary changed.
        if (event.isScaleTopology()){
            assignment.setAssignmentType(Assignment.AssignmentType.ScaleTopology);
        }
        StormClusterState stormClusterState = nimbusData.getStormClusterState();
        stormClusterState.set_assignment(topologyId, assignment);
        // update task heartbeat's start time
        NimbusUtils.updateTaskHbStartTime(nimbusData, assignment, topologyId);
        NimbusUtils.updateTopologyTaskTimeout(nimbusData, topologyId);
        LOG.info("Successfully make assignment for topology id " + topologyId + ": " + assignment);
    }
    return assignment;
}
```
其中的核心代码是:
```
IToplogyScheduler scheduler = schedulers.get(DEFAULT_SCHEDULER_NAME);
assignments = scheduler.assignTasks(context);
```
即，JStorm调用的是一个默认Scheduler, 其实现类是`DefaultTopologyScheduler`:
```java
// com.alibaba.jstorm.scheduler.default_assign.DefaultTopologyScheduler:129
//代码有删减
@Override
public Set<ResourceWorkerSlot> assignTasks(TopologyAssignContext context){
    int assignType = context.getAssignType();

    DefaultTopologyAssignContext defaultContext = new DefaultTopologyAssignContext(context);
    // ...... 如果assignType是rebalance, 则先释放目前占用的slot.
    //
    Set<Integer> needAssignTasks = getNeedAssignTasks(defaultContext);

    Set<ResourceWorkerSlot> keepAssigns = getKeepAssign(defaultContext, needAssignTasks); // 通常是0
    Set<ResourceWorkerSlot> ret = new HashSet<ResourceWorkerSlot>();
    ret.addAll(keepAssigns);
    ret.addAll(defaultContext.getUnstoppedWorkers());

    int allocWorkerNum = defaultContext.getTotalWorkerNum() - defaultContext.getUnstoppedWorkerNum() - keepAssigns.size();
    // as an example, this would be 2, 2, 0
    LOG.info("allocWorkerNum=" + allocWorkerNum + ", totalWorkerNum=" + defaultContext.getTotalWorkerNum() + ", keepWorkerNum=" + keepAssigns.size());

    // 执行实际的 workers分配。
    List<ResourceWorkerSlot> availableWorkers = WorkerScheduler.getInstance().getAvailableWorkers(defaultContext, needAssignTasks, allocWorkerNum);
    TaskScheduler taskScheduler = new TaskScheduler(defaultContext, needAssignTasks, availableWorkers);
    Set<ResourceWorkerSlot> assignment = new HashSet<ResourceWorkerSlot>(taskScheduler.assign());
    ret.addAll(assignment);
    LOG.info("Keep Alive slots:" + keepAssigns);
    LOG.info("Unstopped slots:" + defaultContext.getUnstoppedWorkers());
    LOG.info("New assign slots:" + assignment);

    return ret;
}
```
通常来说, `allocWorkerNum`与`totalWorkerNum`是相等的，都等于用户设置的workers数量。 worker的分配由`WorkerScheduler`实现:
```java
public List<ResourceWorkerSlot> getAvailableWorkers(
	DefaultTopologyAssignContext context, Set<Integer> needAssign,
	int allocWorkerNum) {
int workersNum = getAvailableWorkersNum(context);
if (workersNum < allocWorkerNum) {
	throw new FailedAssignTopologyException(
			"there's no enough worker. allocWorkerNum="
					+ allocWorkerNum + ", availableWorkerNum="
					+ workersNum);
}
workersNum = allocWorkerNum;
List<ResourceWorkerSlot> assignedWorkers = new ArrayList<ResourceWorkerSlot>();
// userdefine assignments, but dont't try to use custom scheduling for
// TM bolts now.
getRightWorkers(
		context,
		needAssign,
		assignedWorkers,
		workersNum,
		getUserDefineWorkers(context, ConfigExtension
				.getUserDefineAssignment(context.getStormConf())));

// old assignments
if (ConfigExtension.isUseOldAssignment(context.getStormConf())) {
	getRightWorkers(context, needAssign, assignedWorkers, workersNum,
			context.getOldWorkers());
} else if (context.getAssignType() == TopologyAssignContext.ASSIGN_TYPE_REBALANCE
		&& context.isReassign() == false) {
	int cnt = 0;
	for (ResourceWorkerSlot worker : context.getOldWorkers()) {
		if (cnt < workersNum) {
			ResourceWorkerSlot resFreeWorker = new ResourceWorkerSlot();
			resFreeWorker.setPort(worker.getPort());
			resFreeWorker.setHostname(worker.getHostname());
			resFreeWorker.setNodeId(worker.getNodeId());
			assignedWorkers.add(resFreeWorker);
			cnt++;
		} else {
			break;
		}
	}
}
// calculate rest TM bolts
int workersForSingleTM = 0;
if (context.getAssignSingleWorkerForTM()) {
	for (Integer taskId : needAssign) {
		String componentName = context.getTaskToComponent().get(taskId);
		if (componentName.equals(Common.TOPOLOGY_MASTER_COMPONENT_ID)) {
			workersForSingleTM++;
		}
	}
}
LOG.info("Get workers from user define and old assignments: "
		+ assignedWorkers);

int restWokerNum = workersNum - assignedWorkers.size();
if (restWokerNum < 0)
	throw new FailedAssignTopologyException(
			"Too much workers are needed for user define or old assignments. workersNum="
					+ workersNum + ", assignedWokersNum="
					+ assignedWorkers.size());

for (int i = 0; i < restWokerNum; i++) {
	assignedWorkers.add(new ResourceWorkerSlot());
}
List<SupervisorInfo> isolationSupervisors = this
		.getIsolationSupervisors(context);
if (isolationSupervisors.size() != 0) {
	putAllWorkerToSupervisor(assignedWorkers,
			getResAvailSupervisors(isolationSupervisors));
} else {
    // 执行实际的分配
	putAllWorkerToSupervisor(assignedWorkers,
			getResAvailSupervisors(context.getCluster()));
}
this.setAllWorkerMemAndCpu(context.getStormConf(), assignedWorkers);
// 怎么到这里来的。这里已经完成了workers的分配。
LOG.info("Assigned workers=" + assignedWorkers);
return assignedWorkers;
}
```
继续分配:
```java
private void putAllWorkerToSupervisor(
		List<ResourceWorkerSlot> assignedWorkers,
		List<SupervisorInfo> supervisors) {
	for (ResourceWorkerSlot worker : assignedWorkers) {
		if (worker.getHostname() != null) {
			for (SupervisorInfo supervisor : supervisors) {
				if (NetWorkUtils.equals(supervisor.getHostName(),
						worker.getHostname())
						&& supervisor.getAvailableWorkerPorts().size() > 0) {
					putWorkerToSupervisor(supervisor, worker);
					break;
				}
			}
		}
	}
	supervisors = getResAvailSupervisors(supervisors);
	Collections.sort(supervisors, new Comparator<SupervisorInfo>() {

		@Override
		public int compare(SupervisorInfo o1, SupervisorInfo o2) {
			// TODO Auto-generated method stub
			return -NumberUtils.compare(
					o1.getAvailableWorkerPorts().size(), o2
							.getAvailableWorkerPorts().size());
		}

	});
	putWorkerToSupervisor(assignedWorkers, supervisors);
}
```
具体选择哪些worker:
```java
private void putWorkerToSupervisor(
	List<ResourceWorkerSlot> assignedWorkers,
	List<SupervisorInfo> supervisors) {
	int allUsedPorts = 0;
	for (SupervisorInfo supervisor : supervisors) {
		int supervisorUsedPorts = supervisor.getWorkerPorts().size()
				- supervisor.getAvailableWorkerPorts().size();
		allUsedPorts = allUsedPorts + supervisorUsedPorts;
	}
	// per supervisor should be allocated ports in theory
	int theoryAveragePorts = (allUsedPorts + assignedWorkers.size())
			/ supervisors.size() + 1;
	// supervisor which use more than theoryAveragePorts ports will be
	// pushed overLoadSupervisors
	List<SupervisorInfo> overLoadSupervisors = new ArrayList<SupervisorInfo>();
	int key = 0;
	Iterator<ResourceWorkerSlot> iterator = assignedWorkers.iterator();
	while (iterator.hasNext()) {
		if (supervisors.size() == 0)
			break;
		if (key >= supervisors.size())
			key = 0;
		SupervisorInfo supervisor = supervisors.get(key);
		int supervisorUsedPorts = supervisor.getWorkerPorts().size()
				- supervisor.getAvailableWorkerPorts().size();
		if (supervisorUsedPorts < theoryAveragePorts) {
			ResourceWorkerSlot worker = iterator.next();
			if (worker.getNodeId() != null)
				continue;
			worker.setHostname(supervisor.getHostName());
			worker.setNodeId(supervisor.getSupervisorId());
			worker.setPort(supervisor.getAvailableWorkerPorts().iterator()
					.next());
			supervisor.getAvailableWorkerPorts().remove(worker.getPort());
			if (supervisor.getAvailableWorkerPorts().size() == 0)
				supervisors.remove(supervisor);
			key++;
		} else {
			overLoadSupervisors.add(supervisor);
			supervisors.remove(supervisor);
		}
	}
	// rest assignedWorkers will be allocate supervisor by deal
	Collections.sort(overLoadSupervisors, new Comparator<SupervisorInfo>() {

		@Override
		public int compare(SupervisorInfo o1, SupervisorInfo o2) {
			// TODO Auto-generated method stub
			return -NumberUtils.compare(
					o1.getAvailableWorkerPorts().size(), o2
							.getAvailableWorkerPorts().size());
		}

	});
	key = 0;
	while (iterator.hasNext()) {
		if (overLoadSupervisors.size() == 0)
			break;
		if (key >= overLoadSupervisors.size())
			key = 0;
		ResourceWorkerSlot worker = iterator.next();
		if (worker.getNodeId() != null)
			continue;
		SupervisorInfo supervisor = overLoadSupervisors.get(key);
		worker.setHostname(supervisor.getHostName());
		worker.setNodeId(supervisor.getSupervisorId());
		worker.setPort(supervisor.getAvailableWorkerPorts().iterator()
				.next());
		supervisor.getAvailableWorkerPorts().remove(worker.getPort());
		if (supervisor.getAvailableWorkerPorts().size() == 0)
			overLoadSupervisors.remove(supervisor);
		key++;
	}
}
```

## 验证方法
对比日志: `%JSTORM_HOME%/logs/nimbus.log`
实际提交topology验证。


## Task部分:
