# Linux workqueue 讲解

## 一、它解决什么问题

内核里很多事不能"当场做完"：

- 中断处理程序（hardirq）要求快，不能在里面等 10 秒、不能睡眠；
- 有些工作要 `request_firmware()`、要 `mutex_lock()`、要 `wait_for_completion()`，这些都可能睡眠；
- 有些工作只是"稍后做一下"，不需要立刻做。

workqueue 就是内核提供的**"把一段函数推迟到进程上下文，由内核线程异步执行"**的机制。它是中断下半部（bottom half）里**唯一能睡眠**的那种。

---

## 二、核心概念：work 和 worker

三个东西：

```c
struct work_struct;        // 一件事（要执行的函数 + 数据）
struct workqueue_struct;   // 一个队列（一组 worker 的管理者）
kworker 线程;              // 真正干活的进程（ps 里看到的 kworker/uX:Y）
```

关系：

```
schedule_work(&work)
    │  把 work 挂到某个 workqueue 的链表上
    ▼
workqueue_struct
    │  唤醒 / 分配一个 kworker
    ▼
kworker 线程
    │  从链表取 work，调用 work->func(work)
    ▼
你的函数在进程上下文里执行
```

关键点：**`schedule_work()` 的调用者只是"投递"，不执行；执行者是 kworker 线程。**

---

## 三、最基础的用法

```c
#include <linux/workqueue.h>

static void my_work_fn(struct work_struct *work)
{
    // 这里在 kworker 线程里、进程上下文，可以睡眠
    mutex_lock(&my_lock);
    ...
    mutex_unlock(&my_lock);
}

static DECLARE_WORK(my_work, my_work_fn);   // 静态定义 + 初始化

// 或者动态：
struct work_struct my_work;
INIT_WORK(&my_work, my_work_fn);            // 运行时初始化

// 投递：
schedule_work(&my_work);                     // 挂到 system_wq，返回 bool
```

`schedule_work()` 内部其实就是 `queue_work(system_wq, work)`。

---

## 四、API 分层

### 1. 一次性 work

```c
INIT_WORK(&work, fn);
schedule_work(&work);          // 挂到 system_wq
queue_work(wq, &work);         // 挂到指定 wq
```

同一个 work 在**还没执行完**时再次 `queue_work` 不会重复入队（返回 false），这是 workqueue 自带的去重。

### 2. 延迟执行

```c
struct delayed_work dwork;
INIT_DELAYED_WORK(&dwork, fn);
schedule_delayed_work(&dwork, HZ);      // 1 秒后执行
queue_delayed_work(wq, &dwork, msecs_to_jiffies(500));
```

内部用 timer 到期后再把 work 挂到队列。

### 3. 周期性

workqueue 本身**不提供周期执行**。标准做法是在 fn 结尾再 `schedule_delayed_work()` 一次，自己续期。

### 4. 同步 / 取消

```c
flush_work(&work);              // 等这个 work 执行完（如果没在跑/没排队就直接返回）
flush_workqueue(wq);            // 等这个 wq 上所有 work 执行完
cancel_work_sync(&work);        // 取消排队；如果正在跑，等它跑完
cancel_delayed_work_sync(&dwork);
```

`cancel_work_sync()` 是驱动 remove 路径的标配：保证 work 不会在你释放资源后还去访问那些资源。

---

## 五、执行上下文

- 跑在 **kworker 内核线程**里，属于**进程上下文**；
- 因此**可以睡眠**：`mutex_lock`、`kmalloc(GFP_KERNEL)`、`wait_for_completion`、`request_firmware`、`msleep` 都可以；
- 但不能访问用户态地址（没有用户 mm），不能做需要 `current->mm` 的事。

对比其他下半部：

| 机制 | 上下文 | 能否睡眠 | 典型用途 |
|---|---|---|---|
| softirq | 中断上下文 | 不能 | 网络收包、块设备完成 |
| tasklet | 中断上下文 | 不能 | 老式下半部 |
| **workqueue** | **进程上下文（kworker）** | **能** | 需要睡眠、耗时、可能阻塞的工作 |
| 内核线程 | 进程上下文 | 能 | 长期循环、专用线程 |

一句话：**要睡眠就用 workqueue，不要用 tasklet。**

---

## 六、workqueue 的类型

### 1. 系统共享队列

内核预定义了几个，直接 `schedule_work()` / `queue_work()` 用：

| 队列 | 特点 |
|---|---|
| `system_wq` | 普通优先级，每 CPU 一个，最常用 |
| `system_highpri_wq` | 高优先级（`WQ_HIGHPRI`） |
| `system_long_wq` | 给可能长时间运行的工作，避免阻塞其他 work |
| `system_unbound_wq` | 不绑定 CPU（`WQ_UNBOUND`） |
| `system_freezable_wq` | 系统 suspend 时会冻结 |
| `system_power_efficient_wq` | 省电模式 |

### 2. 自定义队列

```c
struct workqueue_struct *my_wq;

my_wq = alloc_workqueue("my_wq", WQ_UNBOUND | WQ_MEM_RECLAIM, 0);
queue_work(my_wq, &work);
...
destroy_workqueue(my_wq);
```

`alloc_workqueue(name, flags, max_active)`：

- `flags` 控制行为（见下）；
- `max_active` 控制每个 CPU 上同时能跑多少个 work（0 = 默认，通常 256）。

### 3. 常用 flag

| flag | 含义 |
|---|---|
| `WQ_UNBOUND` | 不绑定 CPU，由全局 worker 池调度；适合 CPU 密集型或需要负载均衡 |
| `WQ_HIGHPRI` | 高优先级 worker |
| `WQ_MEM_RECLAIM` | 内存回收路径也要能用（rescue worker），驱动里常用 |
| `WQ_FREEZABLE` | suspend 时冻结 |
| `WQ_CPU_INTENSIVE` | 标记为 CPU 密集型，不占并发额度 |

绑定型（默认，不带 `WQ_UNBOUND`）是 per-CPU 的：每个 CPU 有自己的 worker 池，`queue_work` 投到当前 CPU 的池。`WQ_UNBOUND` 则是一个全局池。

---

## 七、并发管理：max_active

每个 workqueue 在每个 CPU 上有 `max_active` 个并发额度。超过额度的 work 会在队列里等。

- 默认绑定型 `max_active` 通常是 256；
- 设成 1 就变成"串行队列"（同一时刻只有一个 work 在跑）——这是做"串行化"的常用手段；
- `WQ_UNBOUND` 的 `max_active` 是整个 wq 共享的。

```c
// 串行 workqueue：同一时刻只跑一个 work
my_wq = alloc_workqueue("serial_wq", 0, 1);
```

---

## 八、常见坑

1. **`INIT_WORK` 必须在第一次 `queue_work` 之前**，否则排的是未初始化结构。
2. **remove 里要 `cancel_work_sync()`**，否则 work 可能在资源释放后还在跑 → use-after-free。
3. **work fn 里不能假设调用者的上下文**，它永远在 kworker 里跑。
4. **同一个 work 重复 `queue_work` 会被去重**（没执行完时返回 false），但 `queue_delayed_work` 会重置 timer。
5. **不要在自己的 work fn 里 `flush_work(自己)`**，会死锁。
6. **`WQ_MEM_RECLAIM`**：如果 work 可能参与内存回收路径，要加这个 flag，否则内存紧张时可能死锁。
7. **`destroy_workqueue()` 前要先 `flush_workqueue()` 或确保没有 pending work**，否则会 warn。

---

## 九、一句话总结

> workqueue = "把函数丢给 kworker 线程，在进程上下文异步执行"的机制。  
> `INIT_WORK` 绑定，`queue_work` 投递，`flush_work` / `cancel_work_sync` 同步清理。  
> 它和 tasklet/softirq 的根本区别是**能睡眠**，所以凡是需要 `mutex`、`request_firmware`、`wait_for_completion` 的推迟工作，都用它。