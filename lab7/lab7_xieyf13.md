## 练习0：填写已有实验

## 练习1：理解内核级信号量的实现和基于内核级信号量的哲学家就餐问题（不需要编码）

相比于lab6，lab7使用信号量sem代替lock完成互斥。

- 请在实验报告中给出内核级信号量的设计描述，并说其大致执行流流程。

在sem.c和sem.h中定义及实现的信号量sem：
```
typedef struct {
    int value;
    wait_queue_t wait_queue;
} semaphore_t;

void sem_init(semaphore_t *sem, int value);
void up(semaphore_t *sem);
void down(semaphore_t *sem);
bool try_down(semaphore_t *sem);
```

up函数：先禁用中断，检查等待队列。若等待队列为空，value值加一；否则唤醒一个等待进程，然后恢复中断。

down函数：先屏蔽中断，若value的值大于0，则减一并恢复中断后退出。否则恢复中断，将当前进程加入到等待队列中，然后唤醒其他进程。直到该进程再次被唤醒，先屏蔽中断，将其从等待队列中删除，最后恢复中断。

- 请在实验报告中给出给用户态进程/线程提供信号量机制的设计方案，并比较说明给内核级提供信号量机制的异同。

相同：对于用户态进程/线程的信号量，信号量的各项操作都会转换为系统调用，在内核中会有一个信号量与之对应。每当用户进程调用信号量相关函数时，都会进入系统调用，由操作系统内核进行处理，之后再返回到用户态开始执行。

不同：用户态需要使用内核态信号量机制，并且在使用时进行系统调用。

## 练习2: 完成内核级条件变量和基于内核级条件变量的哲学家就餐问题（需要编码）

- 请在实验报告中给出内核级条件变量的设计描述，并说其大致执行流流程。


cond_signal

```
void 
cond_signal (condvar_t *cvp) {
   cprintf("cond_signal begin: cvp %x, cvp->count %d, cvp->owner->next_count %d\n", cvp, cvp->count, cvp->owner->next_count);  
   monitor_t* mtp = cvp->owner;
   if (cvp->count > 0){
	   mtp->next_count++;
	   up(&(cvp->sem));
	   down(&(mtp->next));
	   mtp->next_count--;
   }
   cprintf("cond_signal end: cvp %x, cvp->count %d, cvp->owner->next_count %d\n", cvp, cvp->count, cvp->owner->next_count);
}
```

首先进程判断cv.count，如果不大于0，则表示当前没有执行cond_wait而睡眠的进程，函数退出；如果大于0，这表示当前有执行cond_wait而睡眠的进程A，因此需要唤醒等待在cv.sem上睡眠的进程A。由于只允许一个进程在管程中执行，所以一旦进程B唤醒了别人（进程A），那么自己就需要睡眠。故让monitor.next_count加一，且让自己（进程B）睡在信号量monitor.next上。如果睡醒了，这让monitor.next_count减一。

cond_wait

```
void
cond_wait (condvar_t *cvp) {
    cprintf("cond_wait begin:  cvp %x, cvp->count %d, cvp->owner->next_count %d\n", cvp, cvp->count, cvp->owner->next_count);
    cvp->count ++;
    monitor_t* mtp = cvp->owner;
    if (mtp->next_count > 0)
    	up(&(mtp->next));
    else
    	up(&(mtp->mutex));
    down(&(cvp->sem));
    cvp->count --;
    cprintf("cond_wait end:  cvp %x, cvp->count %d, cvp->owner->next_count %d\n", cvp, cvp->count, cvp->owner->next_count);
}
```

如果调用了cond_wait函数，那么当前等待该条件的睡眠进程个数count加一。

如果next_count大于0，说明有大于等于1个进程执行cond_signal且睡着了，所以唤醒mtp的next信号量，然后进程睡在cv.sem上，如果睡醒了，则让cv.count减一，表示等待此条件的睡眠进程个数少了一个，

如果next_count小于等于0，表示目前没有进程执行cond_signal函数且睡着了，那需要唤醒的是由于互斥条件限制而无法进入管程的进程，所以要唤醒睡在monitor.mutex上的进程。然后进程A睡在cv.sem上，如果睡醒了，则让cv.count减一，表示等待此条件的睡眠进程个数少了一个，可继续执行了。

```
void phi_take_forks_condvar(int i) {
     down(&(mtp->mutex));
//--------into routine in monitor--------------
     // I am hungry
     // try to get fork
     state_condvar[i] = HUNGRY;
     phi_test_condvar(i);
     while (state_condvar[i] != EATING)
    	 cond_wait(&mtp->cv[i]);
//--------leave routine in monitor--------------
      if(mtp->next_count>0)
         up(&(mtp->next));
      else
         up(&(mtp->mutex));
}
```

当哲学家开始时，首先设置状态为HUNGRY，之后检测是否满足他的条件，如果不满足的话一直对条件变量进行wait操作。

```
void phi_put_forks_condvar(int i) {
     down(&(mtp->mutex));

//--------into routine in monitor--------------
     // I ate over
     // test left and right neighbors
     state_condvar[i] = THINKING;
     phi_test_condvar(LEFT);
     phi_test_condvar(RIGHT);
//--------leave routine in monitor--------------
     if(mtp->next_count>0)
        up(&(mtp->next));
     else
        up(&(mtp->mutex));
}
```

在哲学家放下筷子时，将他的状态设置为THINKING，同时test它左右两个人的状态。

- 请在实验报告中给出给用户态进程/线程提供条件变量机制的设计方案，并比较说明给内核级提供条件变量机制的异同。

相同：对于用户态进程/线程的条件变量，条件变量的各项操作都会转换为系统调用，在内核中会有一个条件变量与之对应。每当用户进程调用条件变量相关函数时，都会进入系统调用，由操作系统内核进行处理，之后再返回到用户态开始执行。

不同：用户态需要使用内核态条件变量机制，并且在使用时进行系统调用。

