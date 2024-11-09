

1. 在完成本次实验的过程（含此前学习的过程）中，我曾分别与 以下各位 就（与本次实验相关的）以下方面做过交流，还在代码中对应的位置以注释形式记录了具体的交流对象及内容：

        无

2. 此外，我也参考了 以下资料 ，还在代码中对应的位置以注释形式记录了具体的参考来源及内容：

        https://learningos.cn/rCore-Camp-Guide-2024A/honorcode.html
        https://rcore-os.cn/rCore-Tutorial-Book-v3/index.html

3. 我独立完成了本次实验除以上方面之外的所有工作，包括代码与文档。 我清楚地知道，从以上方面获得的信息在一定程度上降低了实验难度，可能会影响起评分。

4. 我从未使用过他人的代码，不管是原封不动地复制，还是经过了某些等价转换。 我未曾也不会向他人（含此后各届同学）复制或公开我的实验代码，我有义务妥善保管好它们。 我提交至本实验的评测系统的代码，均无意于破坏或妨碍任何计算机系统的正常运转。 我清楚地知道，以上情况均为本课程纪律所禁止，若违反，对应的实验成绩将按“-100”分计。


考虑到代码可读性，主要对task进行修改
修改TCB为以下结构
```
pub struct TaskControlBlock {
    /// The task status in it's lifecycle
    pub task_status: TaskStatus,
    /// The task context
    pub task_cx: TaskContext,
    /// The start time of the task
    pub start_time: usize,
    /// The total running time of the task
    pub syscall_times: [u32; MAX_SYSCALL_NUM],
    /// The task runned or not
    pub runned: bool,
}
```
syscall_times用来记录系统调用次数
start_time记录程序首次加载的时间
runned标记程序是否已经被启动和上面的start_time配合
主要代码实现在TaskManager
```
impl TaskManager {
    ...
    fn set_start_time(&self) {
        let mut inner = self.inner.exclusive_access();
        let current = inner.current_task;
        inner.tasks[current].start_time = crate::timer::get_time_us();
    }

    fn get_runned_time(&self) -> usize {
        let inner = self.inner.exclusive_access();
        let current = inner.current_task;
        let current_time = crate::timer::get_time_us();
        let runned_time = current_time - inner.tasks[current].start_time;
        runned_time / 1000
    }

    fn add_syscall_count(&self, syscall_id: usize) {
        let mut inner = self.inner.exclusive_access();
        let current = inner.current_task;
        inner.tasks[current].syscall_times[syscall_id] += 1;
    }

    fn get_syscall_count(&self) -> [u32; MAX_SYSCALL_NUM] {
        let inner = self.inner.exclusive_access();
        let current = inner.current_task;
        inner.tasks[current].syscall_times.clone()
    }
    ...
}
```
set_start_time设置启动时间
get_runned_time获取运行时间
add_syscall_count增加syscall计数
get_syscall_count获取syscall计数
```
/// Set the start time of the current task
pub fn set_start_time() {
    TASK_MANAGER.set_start_time();
}
/// Get the runned time of the current task
pub fn get_runned_time() -> usize {
    TASK_MANAGER.get_runned_time()
}
/// Count the syscall of the current task
pub fn add_syscall_count(syscall_id  : usize) {
    TASK_MANAGER.add_syscall_count(syscall_id);
}

/// Get the syscall count of the current task
pub fn get_syscall_count()-> [u32; MAX_SYSCALL_NUM] {
    TASK_MANAGER.get_syscall_count()
}
```
封装函数
```
pub fn syscall(syscall_id: usize, args: [usize; 3]) -> isize {
    add_syscall_count(syscall_id);  
    ...
}
```
syscall计数增加，这里其实应该加个范围判断
```
pub fn sys_task_info(_ti: *mut TaskInfo) -> isize {
    trace!("kernel: sys_task_info");
    unsafe{
        let ti = _ti.as_mut().unwrap();
        ti.status = TaskStatus::Running;
        ti.syscall_times = get_syscall_count();
        ti.time = get_runned_time();

    }
    0
}
```
sys_task_info实现

