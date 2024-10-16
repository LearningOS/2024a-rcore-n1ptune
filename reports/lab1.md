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

