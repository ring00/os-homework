# 实验六：调度器

## 练习一

### sched_class分析

`struct sched_class`定义如下

```c
struct sched_class {
    const char *name;
    void (*init)(struct run_queue *rq);
    void (*enqueue)(struct run_queue *rq, struct proc_struct *proc);
    void (*dequeue)(struct run_queue *rq, struct proc_struct *proc);
    struct proc_struct *(*pick_next)(struct run_queue *rq);
    void (*proc_tick)(struct run_queue *rq, struct proc_struct *proc);
};
```

`sched_class`通过内部的函数指针来帮助灵活切换不同的调度算法，各个函数指针功能如下

* `init`：初始化进程调度队列
* `enqueue`：将进程放入调度队列
* `dequeue`：将进程移除调度队列
* `pick_next`：选出下一个运行的进程
* `proc_tick`：通知进程刚刚过去一个tick，可用于进一步处理

### 调度执行过程

* 关闭中断
* 将当前进程设为无需调度
* 如果当前进程处于就绪状态，则加入调度队列尾部
* 从调度队列头选出一个进程
* 运行该进程
* 使能中断

### 多级反馈队列调度算法

在多级反馈队列调度算法中，操作系统会维护多个FIFO队列，每个队列的优先级不同，优先级越高，时间片越短。

对于一个进程，它在调度系统中的运行情况大致如下

* 将新进程插入最高优先级队列尾部
* 进程逐渐移动到队首并被调度执行
* 若进程在限定时间内结束，则离开系统
* 若进程请求IO等阻塞操作，则被移出当前队列，当进程就绪时被放回同一个队列的队尾
* 若进程未在限定时间内结束，则打断该进程并将进程移至次优先的队列尾部
* 该过程会反复执行，直到进程结束或进程已到达优先级最低的队列为止

从上述过程可以看出该算法更偏向短任务，十分适合实时操作系统

## 练习二

### stride_init

* 初始化指向就绪进程的指针
* 初始化就绪进程数为0

### stride_enqueue

* 检查proc->run_link，保证为空
* 检查rq->lab6_run_pool是否为NULL
    * 若为NULL，则初始化proc->run_link，并将rq->lab6_run_pool指向它
    * 否则，调用skew_heap_insert将其插入斜堆
* 若进程已无时间片或者时间片数大于允许的最大值，则将其设为rq->max_time_slice
* 将proc->rq指向rq
* 就绪数加1

### stride_dequeue

* 检查`proc->rq == rq && rq->proc_num > 0`
* 调用skew_heap_remove将进程移出队列
* 就绪进程数减1

### stride_pick_next

* 直接选择堆顶的进程
* 将该进程的stride加上BIG_STRIDE除以优先级

## 总结

### 实现与参考答案的区别

实现逻辑基本相同，`BIG_STRIDE`取值不同，实验后发现只要不取极端值都没有影响

### 知识点

* 进程调度算法
    * 时间片轮算法
    * stride scheduling算法
* 进程切换过程
