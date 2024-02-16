---
title: '清华大学开源操作系统训练营: rCore chapter3练习'
date: 2023-12-11 22:54:06
category: 
- '训练营笔记'
- '清华大学开源操作系统训练营2023'
tags: 
- 'OS'
- '操作系统'
- 'Rust'
- 'riscv'
---

**练习实验书**: https://learningos.cn/rCore-Tutorial-Guide-2023A/chapter3/5exercise.html

**我的代码**: https://github.com/ToniXWD/2023a-rcore-ToniXWD/tree/ch3

# 1 编程作业
>> ch3 中，我们的系统已经能够支持多个任务分时轮流运行，我们希望引入一个新的系统调用 sys_task_info 以获取当前任务的信息...

1. 在`task.rs`中的`TaskControlBlock`结构体增加`sys_call_times`数组, 用于记录当前`task`中各个系统调用的次数
   ```rust
   /// The task control block (TCB) of a task.
    #[derive(Copy, Clone)]
    pub struct TaskControlBlock {
        /// The task status in it's lifecycle
        pub task_status: TaskStatus,
        /// The task context
        pub task_cx: TaskContext,
        /// syscall time count
        pub sys_call_times: [u32; MAX_SYSCALL_NUM], // 新增
    }
    ```
2. 每次执行系统调用时, 将全局变量`TASK_MANAGER`中当前任务`current_task`对应的`TaskControlBlock`结构体的系统调用记录自增
   ```rust
   pub fn syscall(syscall_id: usize, args: [usize; 3]) -> isize {
        increase_sys_call(syscall_id); // 新增
        match syscall_id {
            SYSCALL_WRITE => sys_write(args[0], args[1] as *const u8, args[2]),
            SYSCALL_EXIT => sys_exit(args[0] as i32),
            SYSCALL_YIELD => sys_yield(),
            SYSCALL_GET_TIME => sys_get_time(args[0] as *mut TimeVal, args[1]),
            SYSCALL_TASK_INFO => sys_task_info(args[0] as *mut TaskInfo),
            _ => panic!("Unsupported syscall_id: {}", syscall_id),
        }
    }  
    ```
3. 为`TaskManager`实现`get_sys_call_times`方法, 获取当前任务`current_task`对应的`TaskControlBlock`结构体的系统调用数组的拷贝
   ```rust
       fn get_sys_call_times(&self) -> [u32; MAX_SYSCALL_NUM] {
        let inner: core::cell::RefMut<'_, TaskManagerInner> = self.inner.exclusive_access();
        inner.tasks[inner.current_task].sys_call_times.clone()
    }
    ```
4. 完成`process.rs`的`sys_task_info`, 调用`get_sys_call_times`和`get_time_ms`获取`TaskInfo`结构体的`syscall_times`和`time`部分, `status`部分设为`Running`
   ```rust
   pub fn sys_task_info(_ti: *mut TaskInfo) -> isize {
        trace!("kernel: sys_task_info");
        unsafe {
            *_ti = TaskInfo {
                status: TaskStatus::Running,
                syscall_times: get_sys_call_times(),
                time: get_time_ms(),
            }
        }
        0
    }
    ```

# 2 简答作业
## 2.1 简答作业第一部分
正确进入 U 态后，程序的特征还应有：使用 S 态特权指令，访问 S 态寄存器后会报错。 请同学们可以自行测试这些内容 (运行 Rust 三个 bad 测例 (ch2b_bad_*.rs) ， 注意在编译时至少需要指定 LOG=ERROR 才能观察到内核的报错信息) ， 描述程序出错行为，同时注意注明你使用的 sbi 及其版本。

Rustsbi 版本为: 0.2.0-alpha.2

出现报错: 
```bash
[kernel] PageFault in application, bad addr = 0x0, bad instruction = 0x804003c4, kernel killed it.
[kernel] IllegalInstruction in application, kernel killed it.
[kernel] IllegalInstruction in application, kernel killed it.
```

`ch2b_bad_address.rs` 由于除0错误触发异常退出
`ch2b_bad_instructions.rs` 在用户态非法使用指令`sret`
`ch2b_bad_register.rs` 在用户态非法使用指令`csrr`


## 2.2 简答作业第二部分
深入理解 trap.S 中两个函数 __alltraps 和 __restore 的作用，并回答如下问题:
### 2.2.1 L40：刚进入 __restore 时，a0 代表了什么值。请指出 __restore 的两种使用情景。
> 1. 刚进入 __restore 时，a0 代表了系统调用的第一个参数
> 2. __restore 的作用包括:
>   - 从系统调用和异常返回时, 恢复要返回的用户态的上下文信息
>   - 任务切换时, 恢复要切换的任务的上下文信息

### 2.2.2 L43-L48：这几行汇编代码特殊处理了哪些寄存器？这些寄存器的的值对于进入用户态有何意义？请分别解释。
```bash
ld t0, 32*8(sp) # 内核栈 32*8(sp) 处存储了原 sstatus 寄存器的值, 将其读取到 t0
ld t1, 33*8(sp) # 内核栈 32*8(sp) 处存储了原 sepc 寄存器的值, 将其读取到 t1
ld t2, 2*8(sp) # 内核栈 32*8(sp) 处存储了原 sscratch 寄存器的值, 将其读取到 t2
csrw sstatus, t0 # 将 t0中原 sstatus 寄存器的值读取到 sstatus
csrw sepc, t1 # 将 t0中原 sepc 寄存器的值读取到 sepc
csrw sscratch, t2 # 将 t0中原 sscratch 寄存器的值读取到 sscratch
```

### 2.2.3 L50-L56：为何跳过了 x2 和 x4？
1. 跳过`x2`是因为`x2`对应的用户栈指针保存到了sscratch寄存器, 不需要从内核栈中进行恢复
2. 跳过`x4`是因为并没有使用它, 所以无需恢复

### 2.2.4 L60：该指令之后，sp 和 sscratch 中的值分别有什么意义？
`sp`指向用户栈, `sscratch`指向内核栈

### 2.2.5 __restore：中发生状态切换在哪一条指令？为何该指令执行之后会进入用户态？
`sret`后发生了状态切换, 执行该指令后, PC设置为 `sepc` 寄存器的值。`sepc` 存储着产生中断或异常前的指令地址，因此这实现了到原始代码的返回。

### 2.2.6 L13：该指令之后，sp 和 sscratch 中的值分别有什么意义？
```bash
csrrw sp, sscratch, sp
```
`sp`, `sscratch`寄存器的内容被交换, `sp`保存了原`sscratch`中的内核栈指针, `sscratch`保存了原`sp`中的用户栈栈指针

### 2.2.7 从 U 态进入 S 态是哪一条指令发生的？
`ecall`指令
