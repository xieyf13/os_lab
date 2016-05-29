## 练习0：填写已有实验

## 练习1：用 Round Robin 调度算法（不需要编码）

- 请理解并分析sched_calss中各个函数指针的用法，并接合Round Robin 调度算法描ucore的调度执行过程

`RR_enqueue`函数负责将一个进程加入到调度队列的末尾, 并初始化时间片长度, 等待被执行. 

`RR_dequeue`函数负责将指定的进程从调度队列中删去(因此严格的说这个操作是delete, 而不是dequeue)

`RR_init`初始化调度队列. 

`RR_proc_tick`每次时钟中断都会调用, 将当前进程的时间片剩余减一, 如果已经为零, 那么将该进程置于可被调度状态. 

`RR_pick_next`将队列中最靠前的进程选出并返回. 

就lab6而言, ucore中调度执行包含两部分, 一个部分是在每次时钟中断发生时, `RR_proc_tick`都会被调用, `current`进程的时间片剩余时间被修改, 或是进入可被调度状态. 

另一部分是`init`函数中最后的`cpu_idle`函数, 这个函数会不停地判断当前进程是否需要被调度, 如果需要被调度, 就调用`schdule`函数, `schdule`函数会检查当前进程是否仍然处于可运行的状态, 如果是, 那么, 就将当前进程重新加入调度队列当中, 之后, 从调度队列中选取下一个被执行的进程, 并将其从调度队列中删除, 执行该进程. 

- 如何设计实现”多级反馈队列调度算法“

多级反馈队列调度算法可以借助时间片轮转算法完成, 思路如下:

将ucore中现有的一个`run_list`改为一个`run_lists[]`, 而对于不同的调度队列, 设置不同的最大时间片时间, 例如100, 150, 200, 250, 300..., 几个队列的优先级也逐渐递减. 在进行enqueue时, 根据进程优先级, 将进程插入到特定的调度队列当中, 对于同一个调度队列中的进程, 采取时间片轮转算法进行调度, 选择下一个进程时, 优先从高优先级队列里进行选择. 

## 练习2: 实现 Stride Scheduling 调度算法（需要编码）
替换掉之前的RR调度算法，使用了链表和skew_heap对进行了实现。每个函数的实现如下：

#### strid_init函数

```
static void
stride_init(struct run_queue *rq) {
     /* LAB6: 2013011428 */
     list_init(&(rq->run_list));
     rq->lab6_run_pool = NULL;
     rq->proc_num = 0;
}
```

进行初始化操作，按要求进行即可。

#### enqueue

```
static void
stride_enqueue(struct run_queue *rq, struct proc_struct *proc) {
     /* LAB6: 2013011428 */
#if USE_SKEW_HEAP
     rq->lab6_run_pool =
          skew_heap_insert(rq->lab6_run_pool, &(proc->lab6_run_pool), proc_stride_comp_f);
#else
     assert(list_empty(&(proc->run_link)));
     list_add_before(&(rq->run_list), &(proc->run_link));
#endif
     if (proc->time_slice == 0 || proc->time_slice > rq->max_time_slice) {
          proc->time_slice = rq->max_time_slice;
     }
     proc->rq = rq;
     rq->proc_num ++;
}
```

如果使用堆时，采用skew_heap_insert函数进行插入操作，采用链表时使用list_add_before进行插入操作。更新入队进程所需要占用的时间片，设置proc的rq变量，增加进程数。

#### dequeue

```
static void
stride_dequeue(struct run_queue *rq, struct proc_struct *proc) {
     /* LAB6: 2013011428 */
#if USE_SKEW_HEAP
     rq->lab6_run_pool =
          skew_heap_remove(rq->lab6_run_pool, &(proc->lab6_run_pool), proc_stride_comp_f);
#else
     assert(!list_empty(&(proc->run_link)) && proc->rq == rq);
     list_del_init(&(proc->run_link));
#endif
     rq->proc_num --;
}
```

两种方法分别使用skew_heap_remove和list_del_init方法进行删除。

#### pick_next

```
static struct proc_struct *
stride_pick_next(struct run_queue *rq) {
     /* LAB6: 2013011428 */
#if USE_SKEW_HEAP
     if (rq->lab6_run_pool == NULL) return NULL;
     struct proc_struct *p = le2proc(rq->lab6_run_pool, lab6_run_pool);
#else
     list_entry_t *le = list_next(&(rq->run_list));

     if (le == &rq->run_list)
          return NULL;
     
     struct proc_struct *p = le2proc(le, run_link);
     le = list_next(le);
     while (le != &rq->run_list)
     {
          struct proc_struct *q = le2proc(le, run_link);
          if ((int32_t)(p->lab6_stride - q->lab6_stride) > 0)
               p = q;
          le = list_next(le);
     }
#endif
     if (p->lab6_priority == 0)
          p->lab6_stride += BIG_STRIDE;
     else p->lab6_stride += BIG_STRIDE / p->lab6_priority;
     return p;
}
```

这是调度算法的核心，使用堆时，堆顶元素即为最小值。采用链表的话需要遍历链表才能找到最小值。找到之后根据该进程的priority对步长stride进行增加。

#### proc_tick

```
static void
stride_proc_tick(struct run_queue *rq, struct proc_struct *proc) {
     /* LAB6: 2013011428 */
     if (proc->time_slice > 0) {
          proc->time_slice --;
     }
     if (proc->time_slice == 0) {
          proc->need_resched = 1;
     }
}
```

对于时间片进行处理，每次时钟中断时减一，值为0时则进行相关调度。
