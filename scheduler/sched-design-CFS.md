# CFS 调度器


## 1.概述 

CFS代表`完全公共调度器`,这是新的`桌面`进程调度器实现.

由Ingo Molnar实现并merge到内核Linux2.6.23中.
它代替了之前由`vanilla scheduler`实现的`SCHED_OTHER`交互式代码.

CFS设计的80%可以总结为一句话:CFS在真实硬件上实现了一种“理想的、CPU上精确多任务"的基本模型.

"理想的多任务CPU"是指一个CPU有100%的物理能力并行地以精确的速度运行每个任务,每个任务以1/nr_running的速度.比如,如果有2个任务处于运行状态,那么每个任务使用50%的物理能力---即，实际上是并行的.

在真实的硬件上,在任意时刻,我们只能运行一个任务,所以我们需要引入`虚拟运行时间`的概念.
一个任务的虚拟运行时间决定了该任务在上面提到的多任务CPU上下次开始执行的时间片.
在实际中,任务的`虚拟运行时间`是它的实际运行时间与总共运行任务数计算后的结果.


## 2. 一些实际细节

在CFS实现里,`虚拟运行时间`通过每个任务的`p->se.vruntime(单位ns)`来表示和跟踪.In CFS the virtual runtime is expressed and tracked via the per-task
这样,才有可能准确的计算一个任务`期望获得的CPU time`.
一些细节: 在`理想`的硬件上,所有任务都含有相同的
`p->se.vruntime value`.这样,任务将会同时执行并且没有任务会从理想CPU的共享中失去平衡.

CFS的任务挑选逻辑基于`p->se.vruntime`,并且相当简单:总是挑选`p->se.vruntime`值最小的任务(比如,当前执行时间最少的任务).

CFS总是尝试将CPU时间在可运行的任务间分配的尽可能地接近"理想硬件上多任务运行"的情形.

CFS设计的大部分剩余部分都脱离了这个简单的概念，添加了一些附加的调整，如nice值,多进程,和识别sleepers的算法.


3.  RBTREE
==============

CFS的设计相当激进：它没有使用老的`runqueues`数据结构,而是使用了基于时间排序的`rbtree`来构建要执行任务的时间线.
runqueues, but it uses a time-ordered rbtree to build a "timeline" of future
task execution, and thus has no "array switch" artifacts (by which both the
previous vanilla scheduler and RSDL/SD are affected).

CFS also maintains the rq->cfs.min_vruntime value, which is a monotonic
increasing value tracking the smallest vruntime among all tasks in the
runqueue.  The total amount of work done by the system is tracked using
min_vruntime; that value is used to place newly activated entities on the left
side of the tree as much as possible.

The total number of running tasks in the runqueue is accounted through the
rq->cfs.load value, which is the sum of the weights of the tasks queued on the
runqueue.

CFS maintains a time-ordered rbtree, where all runnable tasks are sorted by the
p->se.vruntime key. CFS picks the "leftmost" task from this tree and sticks to it.
As the system progresses forwards, the executed tasks are put into the tree
more and more to the right --- slowly but surely giving a chance for every task
to become the "leftmost task" and thus get on the CPU within a deterministic
amount of time.

Summing up, CFS works like this: it runs a task a bit, and when the task
schedules (or a scheduler tick happens) the task's CPU usage is "accounted
for": the (small) time it just spent using the physical CPU is added to
p->se.vruntime.  Once p->se.vruntime gets high enough so that another task
becomes the "leftmost task" of the time-ordered rbtree it maintains (plus a
small amount of "granularity" distance relative to the leftmost task so that we
do not over-schedule tasks and trash the cache), then the new leftmost task is
picked and the current task is preempted.



## 4. CFS的一些特性

CFS使用ns粒度进行计算分配,而不再依赖jiffies或者其它HZ相关的东西.
这样CFS调度器没有了之前调度器`时间片`的概念,并且没有任何启发式的算法.
目前仅有一个中心调参(必须打开CONFIG_SCHED_DEBUG):
   /proc/sys/kernel/sched_min_granularity_ns
从`桌面(低延迟)`到`服务器(批处理)负载`时可以调整调度参数.默认的配置比较适合桌面负载.
`SCHED_BATCH`同样由CFS调度模块控制.

Due to its design, the CFS scheduler is not prone to any of the "attacks" that
exist today against the heuristics of the stock scheduler: fiftyp.c, thud.c,
chew.c, ring-test.c, massive_intr.c all work fine and do not impact
interactivity and produce the expected behavior.

The CFS scheduler has a much stronger handling of nice levels and SCHED_BATCH
than the previous vanilla scheduler: both types of workloads are isolated much
more aggressively.

SMP load-balancing has been reworked/sanitized: the runqueue-walking
assumptions are gone from the load-balancing code now, and iterators of the
scheduling modules are used.  The balancing code got quite a bit simpler as a
result.



## 5. 调度策略

CFS implements three scheduling policies:

  - SCHED_NORMAL (traditionally called SCHED_OTHER): The scheduling
    policy that is used for regular tasks.

  - SCHED_BATCH: Does not preempt nearly as often as regular tasks
    would, thereby allowing tasks to run longer and make better use of
    caches but at the cost of interactivity. This is well suited for
    batch jobs.

  - SCHED_IDLE: This is even weaker than nice 19, but its not a true
    idle timer scheduler in order to avoid to get into priority
    inversion problems which would deadlock the machine.

SCHED_FIFO/_RR are implemented in sched/rt.c and are as specified by
POSIX.

The command chrt from util-linux-ng 2.13.1.1 can set all of these except
SCHED_IDLE.



## 6. 调度类

The new CFS scheduler has been designed in such a way to introduce "Scheduling
Classes," an extensible hierarchy of scheduler modules.  These modules
encapsulate scheduling policy details and are handled by the scheduler core
without the core code assuming too much about them.

sched/fair.c implements the CFS scheduler described above.

sched/rt.c implements SCHED_FIFO and SCHED_RR semantics, in a simpler way than
the previous vanilla scheduler did.  It uses 100 runqueues (for all 100 RT
priority levels, instead of 140 in the previous scheduler) and it needs no
expired array.

Scheduling classes are implemented through the sched_class structure, which
contains hooks to functions that must be called whenever an interesting event
occurs.

This is the (partial) list of the hooks:

 - enqueue_task(...)

   Called when a task enters a runnable state.
   It puts the scheduling entity (task) into the red-black tree and
   increments the nr_running variable.

 - dequeue_task(...)

   When a task is no longer runnable, this function is called to keep the
   corresponding scheduling entity out of the red-black tree.  It decrements
   the nr_running variable.

 - yield_task(...)

   This function is basically just a dequeue followed by an enqueue, unless the
   compat_yield sysctl is turned on; in that case, it places the scheduling
   entity at the right-most end of the red-black tree.

 - check_preempt_curr(...)

   This function checks if a task that entered the runnable state should
   preempt the currently running task.

 - pick_next_task(...)

   This function chooses the most appropriate task eligible to run next.

 - set_curr_task(...)

   This function is called when a task changes its scheduling class or changes
   its task group.

 - task_tick(...)

   This function is mostly called from time tick functions; it might lead to
   process switch.  This drives the running preemption.




## 7.  分组调度扩展到CFS

Normally, the scheduler operates on individual tasks and strives to provide
fair CPU time to each task.  Sometimes, it may be desirable to group tasks and
provide fair CPU time to each such task group.  For example, it may be
desirable to first provide fair CPU time to each user on the system and then to
each task belonging to a user.

CONFIG_CGROUP_SCHED strives to achieve exactly that.  It lets tasks to be
grouped and divides CPU time fairly among such groups.

CONFIG_RT_GROUP_SCHED permits to group real-time (i.e., SCHED_FIFO and
SCHED_RR) tasks.

CONFIG_FAIR_GROUP_SCHED permits to group CFS (i.e., SCHED_NORMAL and
SCHED_BATCH) tasks.

   These options need CONFIG_CGROUPS to be defined, and let the administrator
   create arbitrary groups of tasks, using the "cgroup" pseudo filesystem.  See
   Documentation/admin-guide/cgroup-v1/cgroups.rst for more information about this filesystem.

When CONFIG_FAIR_GROUP_SCHED is defined, a "cpu.shares" file is created for each
group created using the pseudo filesystem.  See example steps below to create
task groups and modify their CPU share using the "cgroups" pseudo filesystem::

	# mount -t tmpfs cgroup_root /sys/fs/cgroup
	# mkdir /sys/fs/cgroup/cpu
	# mount -t cgroup -ocpu none /sys/fs/cgroup/cpu
	# cd /sys/fs/cgroup/cpu

	# mkdir multimedia	# create "multimedia" group of tasks
	# mkdir browser		# create "browser" group of tasks

	# #Configure the multimedia group to receive twice the CPU bandwidth
	# #that of browser group

	# echo 2048 > multimedia/cpu.shares
	# echo 1024 > browser/cpu.shares

	# firefox &	# Launch firefox and move it to "browser" group
	# echo <firefox_pid> > browser/tasks

	# #Launch gmplayer (or your favourite movie player)
	# echo <movie_player_pid> > multimedia/tasks
