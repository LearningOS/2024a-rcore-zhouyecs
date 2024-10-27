#### 简单总结你实现的功能

- syscall/process.rs: 使用`sys_task_info`返回taskinfo时,我们需要知道当前task的status, syscall_times和time,可以设置三个函数分别获取:`get_task_status`,`get_task_syscall_times`,`get_task_syscall_times`。因为_ti这个指针可能是null,dangling, unaligned的情况,所以需要使用unsafe进行处理；
- syscall/mod.rs: 根据提示“系统调用次数可以考虑在进入内核态系统调用异常处理函数之后,进入具体系统调用函数之前维护。”,在此处调用了函数`add_syscall_times`,增加task的系统调用次数；
- task/task.rs: 为TaskControlBlock增加了task_syscall_times和task_start_time两个变量,分别记录task的系统调用次数和task的启动时间,需要返回时,用当前时间`get_time_ms`减去开始时间即可；
- task/mod.rs: 实现`add_syscall_times`,`get_task_status`,`get_task_syscall_times`,`get_task_syscall_times`。

#### 问答题

1. 正确进入 U 态后,程序的特征还应有:使用 S 态特权指令,访问 S 态寄存器后会报错。 请同学们可以自行测试这些内容（运行 三个 bad 测例 (ch2b_bad_*.rs) ）, 描述程序出错行为,同时注意注明你使用的 sbi 及其版本。

sbi版本:[rustsbi] RustSBI version 0.3.0-alpha.2, adapting to RISC-V SBI v1.0.0

ch2b_bad_address: 
```
[kernel] PageFault in application, bad addr = 0x0, bad instruction = 0x804003a4, kernel killed it.
[kernel] Panicked at src/syscall/fs.rs:11 called `Result::unwrap()` on an `Err` value: Utf8Error { valid_up_to: 3, error_len: Some(1) }
// 向地址0x0写入数据
```
ch2b_bad_instrcutions:
```
[kernel] IllegalInstruction in application, kernel killed it.
[kernel] Panicked at src/syscall/fs.rs:11 called `Result::unwrap()` on an `Err` value: Utf8Error { valid_up_to: 3, error_len: Some(1) }
// 使用S态特权指令
```
ch2b_bad_register:
```
[kernel] IllegalInstruction in application, kernel killed it.
[kernel] Panicked at src/trap/mod.rs:72 Unsupported trap Exception(InstructionFault), stval = 0x0!
// 访问S态寄存器
```

2. 深入理解 trap.S 中两个函数 __alltraps 和 __restore 的作用,并回答如下问题:

- L40:刚进入 __restore 时,a0 代表了什么值。请指出 __restore 的两种使用情景。

[rCore-Tutorialv3](https://rcore-os.cn/rCore-Tutorial-Book-v3/chapter2/4trap-handling.html#id8)中提到，“第 33 行令 a0<-sp，让寄存器 a0 指向内核栈的栈指针也就是我们刚刚保存的 Trap 上下文的地址，这是由于我们接下来要调用 trap_handler 进行 Trap 处理，它的第一个参数 cx 由调用规范要从 a0 中获取。当 trap_handler 返回之后，使用 __restore 从保存在内核栈上的 Trap 上下文恢复寄存器。最后通过一条 sret 指令回到应用程序执行。"

 __restore 用途:系统调用或者异常结束后,从保存在内核栈上的 Trap 上下文恢复寄存器,即s=>u

- L43-L48:这几行汇编代码特殊处理了哪些寄存器?这些寄存器的的值对于进入用户态有何意义?请分别解释。
```
ld t0, 32*8(sp)
ld t1, 33*8(sp)
ld t2, 2*8(sp)
csrw sstatus, t0
csrw sepc, t1
csrw sscratch, t2
```

处理了sstatus,sepc和sscratch三个寄存器,参考[特权级切换相关的控制状态寄存器](https://rcore-os.cn/rCore-Tutorial-Book-v3/chapter2/4trap-handling.html#id4)
sstatus:SPP 等字段给出 Trap 发生之前 CPU 处在哪个特权级（S/U）等信息
spec:当 Trap 是一个异常的时候,记录 Trap 发生之前执行的最后一条指令的地址
sscratch:指向用户栈

- L50-L56:为何跳过了 x2 和 x4?
```
ld x1, 1*8(sp)
ld x3, 3*8(sp)
.set n, 5
.rept 27
   LOAD_GP %n
   .set n, n+1
.endr
```

跳过 x2(stack pointer) :后面处理,见L33-35,将 sscratch 的值读到寄存器 t2 并保存到内核栈上
跳过 x4(thread pointer) :参考用户栈与内核栈,除非我们手动出于一些特殊用途使用它,否则一般也不会被用到

- L60:该指令之后,sp 和 sscratch 中的值分别有什么意义?
```
csrrw sp, sscratch, sp
```

该指令之后, sp 指向用户栈栈顶, sscratch 保存进入 Trap 之前的状态并指向内核栈栈顶

- __restore:中发生状态切换在哪一条指令?为何该指令执行之后会进入用户态?

参考[特权级切换的硬件控制机制](https://rcore-os.cn/rCore-Tutorial-Book-v3/chapter2/4trap-handling.html#trap-hw-mechanism)，发生状态切换在sret。当 CPU 完成 Trap 处理准备返回的时候，需要通过一条 S 特权级的特权指令 sret 来完成，这一条指令具体完成以下功能：CPU 会将当前的特权级按照 sstatus 的 SPP 字段设置为 U 或者 S ；CPU 会跳转到 sepc 寄存器指向的那条指令，然后继续执行。

- L13:该指令之后,sp 和 sscratch 中的值分别有什么意义?
```
csrrw sp, sscratch, sp
```

交换 sscratch 和 sp 的效果。在这一行之前 sp 指向用户栈， sscratch 指向内核栈，现在 sp 指向内核栈， sscratch 指向用户栈。

- 从 U 态进入 S 态是哪一条指令发生的?

call trap_handler

#### 荣誉准则

1. 在完成本次实验的过程（含此前学习的过程）中,我曾分别与 以下各位 就（与本次实验相关的）以下方面做过交流,还在代码中对应的位置以注释形式记录了具体的交流对象及内容:

无

2. 此外,我也参考了 以下资料 ,还在代码中对应的位置以注释形式记录了具体的参考来源及内容:

无

3. 我独立完成了本次实验除以上方面之外的所有工作,包括代码与文档。 我清楚地知道,从以上方面获得的信息在一定程度上降低了实验难度,可能会影响起评分。

4. 我从未使用过他人的代码,不管是原封不动地复制,还是经过了某些等价转换。 我未曾也不会向他人（含此后各届同学）复制或公开我的实验代码,我有义务妥善保管好它们。 我提交至本实验的评测系统的代码,均无意于破坏或妨碍任何计算机系统的正常运转。 我清楚地知道,以上情况均为本课程纪律所禁止,若违反,对应的实验成绩将按“-100”分计。