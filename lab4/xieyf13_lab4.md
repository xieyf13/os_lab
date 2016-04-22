

练习1：分配并初始化一个进程控制块（需要编码）

alloc_proc函数（位于kern/process/proc.c中）负责分配并返回一个新的struct proc_struct结构，用于存储新建立的内核线程的管理信息。ucore需要对这个结构进行最基本的初始化，你需要完成这个初始化过程。

【提示】在alloc_proc函数的实现中，需要初始化的proc_struct结构中的成员变量至少包括：state/pid/runs/kstack/need_resched/parent/mm/context/tf/cr3/flags/name。
请在实验报告中简要说明你的设计实现过程。请回答如下问题：

请说明proc_struct中struct context context和struct trapframe *tf成员变量含义和在本实验中的作用是啥？（提示通过看代码和编程调试可以判断出来）
   
填充代码：
```
        proc->state = PROC_UNINIT;
        proc->pid = -1;
        proc->runs = 0;
        proc->kstack = 0;
        proc->need_resched = 0;
        proc->parent = NULL;
        proc->mm = NULL;
        memset(&(proc->context), 0, sizeof(struct context));
        proc->tf = NULL;
        proc->cr3 = boot_cr3;
        proc->flags = 0;
        memset(proc->name, 0, PROC_NAME_LEN);
```

proc_struct中<br>
context 为当前进程上下文 ，通过寄存器保存当前进程运行状态，可以通过保存数据恢复进程。<br> 
trapframe 为发生中断时进程上下文。创建进程时，将运行环境切换到kernrl_thread_entry完成创建。
 
 
练习2：为新创建的内核线程分配资源（需要编码）

创建一个内核线程需要分配和设置好很多资源。kernel_thread函数通过调用do_fork函数完成具体内核线程的创建工作。do_kernel函数会调用alloc_proc函数来分配并初始化一个进程控制块，但alloc_proc只是找到了一小块内存用以记录进程的必要信息，并没有实际分配这些资源。ucore一般通过do_fork实际创建新的内核线程。do_fork的作用是，创建当前内核线程的一个副本，它们的执行上下文、代码、数据都一样，但是存储位置不同。在这个过程中，需要给新内核线程分配资源，并且复制原进程的状态。你需要完成在kern/process/proc.c中的do_fork函数中的处理过程。它的大致执行步骤包括：<br> 

调用alloc_proc，首先获得一块用户信息块。<br> 
为进程分配一个内核栈。<br> 
复制原进程的内存管理信息到新进程（但内核线程不必做此事）
复制原进程上下文到新进程。<br> 
将新进程添加到进程列表。<br> 
唤醒新进程。<br> 
返回新进程号。<br> 

请在实验报告中简要说明你的设计实现过程。请回答如下问题：

请说明ucore是否做到给每个新fork的线程一个唯一的id？请说明你的分析和理由。

填充代码： 
   ```
	//获取用户信息块
    if ((proc = alloc_proc()) == NULL) {
        goto fork_out;
    }
	//设置父进程
    proc->parent = current;
	//分配两页内核栈
    if (setup_kstack(proc) != 0) {
        goto bad_fork_cleanup_proc;
    }
    if (copy_mm(clone_flags, proc) != 0) {
        goto bad_fork_cleanup_kstack;
    }
	//中断帧
    copy_thread(proc, stack, tf);

    bool intr_flag;
    local_intr_save(intr_flag);
    {
        proc->pid = get_pid();
        hash_proc(proc);
        list_add(&proc_list, &(proc->list_link));
        nr_process ++;
    }
    local_intr_restore(intr_flag);
	//等待唤醒
    wakeup_proc(proc);

    ret = proc->pid;
	```
ucore可以做到每个新fork的线程一个唯一的id，方式为通过get_pid()创建唯一的ID。当有新的id需求时，通过将上一个id+1.S遍历链表寻找next_safe,将其与新的id比较,若未占用，id可用，否则id+1,重复，直到寻找到可用的。
	
练习3：阅读代码，理解 proc_run 函数和它调用的函数如何完成进程切换的。（无编码工作）

请在实验报告中简要说明你对proc_run函数的分析。并回答如下问题：

在本实验的执行过程中，创建且运行了几个内核线程？<br> 
语句local_intr_save(intr_flag);....local_intr_restore(intr_flag);在这里有何作用?
```
void
proc_run(struct proc_struct *proc) {
    if (proc != current) {
        bool intr_flag;
        struct proc_struct *prev = current, *next = proc;
        local_intr_save(intr_flag);
        {
            current = proc;
            load_esp0(next->kstack + KSTACKSIZE);
            lcr3(next->cr3);
            switch_to(&(prev->context), &(next->context));
        }
        local_intr_restore(intr_flag);
    }
}
```
关中断，然后准备保存现场，切换堆栈，lcr3指令用于切换页目录表基址。
2个  idleproc  init_main<br>
local_intr_save(intr_flag); 关中断<br>
local_intr_restore(intr_flag); 开中断<br>
让中间语句正常执行
