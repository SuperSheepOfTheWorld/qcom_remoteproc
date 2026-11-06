看完了代码结构。下面结合这个代码库给你讲（路径都在 common/drivers/remoteproc/，头文件
 common/include/linux/remoteproc.h）。

 一、remoteproc 是干什么的

 一句话：Linux 用来管理"另一个 CPU 核"上运行的固件的框架。

 在你这台设备上，"另一个核"就是 ADSP / CDSP / Modem / SLPI 这些 DSP/协处理器。它们各自跑独立固件（比如 adsp.mbn），但共
 享内存、需要主 CPU（AP）来加载固件、上电、握手。

 remoteproc 做的事：
 - 把固件（ELF/.mbn）加载到远程核的内存
 - 上电、启动远程核
 - 解析固件里的"资源表"，建立共享内存映射、virtio 通道、日志/trace
 - 处理崩溃、coredump、恢复
 - 通过 sysfs / debugfs / 字符设备向用户态暴露控制接口

 二、分层架构

 ```
    用户态
      │  sysfs: /sys/class/remoteproc/remoteprocN/{state,firmware,name}
      │  cdev : /dev/remoteprocN   (remoteproc_cdev.c)
      │  debugfs: trace、coredump、recovery
      ▼
   ┌───────────────────────────────────────────────┐
   │ remoteproc_core.c  (2800行)   ← 框架核心        │
   │  rproc_alloc / rproc_add / rproc_boot /        │
   │  rproc_shutdown / rproc_attach / rproc_detach /│
   │  rproc_trigger_recovery                        │
   │  + resource table 解析、状态机、引用计数         │
   ├───────────────────────────────────────────────┤
   │ remoteproc_virtio.c  ← 根据 resource table     │
   │                        注册 virtio 设备         │
   │ remoteproc_elf_loader.c ← ELF 固件解析/加载     │
   │ remoteproc_coredump.c / debugfs / sysfs        │
   ├───────────────────────────────────────────────┤
   │ struct rproc_ops  ← 平台相关回调（由驱动实现）   │
   ├───────────────────────────────────────────────┤
   │ 具体驱动: qcom_q6v5_pas.c / qcom_q6v5_adsp.c /  │
   │ qcom_q6v5_mss.c / qcom_q6v5_wcss.c ...          │
   └───────────────────────────────────────────────┘
      ▼
    远程核 (ADSP/CDSP/Modem)
 ```

 关键：core 是通用的、平台无关的；rproc_ops 是平台相关的，由各家 SoC 驱动实现。这是整个框架的设计核心——把"通用生命周期"
 和"怎么给这个具体核上电/放固件"解耦。

 延伸阅读

- [sysfs、cdev、debugfs 是什么](基础知识补充/sysfs-cdev-debugfs是什么.md)
- [remoteproc_core 解释](基础知识补充/remoteproc_core解释.md)
- [remoteproc_组件 解释](基础知识补充/remoteproc组件解释.md)
- [qcom的PAS实现](基础知识补充/qcom的PAS实现.md)

 三、两个核心数据结构

 ### 1. struct rproc —— 一个远程处理器的抽象

 ```c
   struct rproc {
       struct list_head node;
       const char *name;            // 名字，如 "adsp", "cdsp"
       const char *firmware;        // 固件名，如 "qcom/adsp.mbn"
       struct rproc_ops *ops;       // 平台回调
       struct device dev;           // 对应 /sys/class/remoteproc/remoteprocN
       atomic_t power;              // 引用计数：多少人想让它开着
       unsigned int state;          // 状态机
       struct mutex lock;           // 保护 state/power 的锁 ← 你补丁加准入点的地方
       struct list_head rvdevs;     // 从 resource table 生成的 virtio 设备
       struct resource_table *table_ptr; // 固件里的资源表
       bool auto_boot;
       ...
   };
 ```

 状态机（enum rproc_state）：

 ```
   RPROC_OFFLINE(0) ? RPROC_RUNNING(2)
          ?                 ?
   RPROC_SUSPENDED(1)   RPROC_CRASHED(3)
   RPROC_DELETED(4)   RPROC_ATTACHED(5)   RPROC_DETACHED(6)
 ```

 ATTACH 表示"远程核已经被别人（如 bootloader）启动，我只接管管理"，这在 Android 上很常见。

 ### 2. struct rproc_ops —— 平台回调

 这个代码库里 rproc_ops 的完整成员（remoteproc.h）：

 ```c
   struct rproc_ops {
       int  (*prepare)(struct rproc *);      // 准备（时钟/电源域）
       int  (*unprepare)(struct rproc *);
       int  (*start)(struct rproc *);        // 启动远程核
       int  (*stop)(struct rproc *);
       int  (*attach)(struct rproc *);       // 接管已在运行的核
       int  (*detach)(struct rproc *);
       void (*kick)(struct rproc *, int vqid); // 通知远程核（doorbell）
       void *(*da_to_va)(struct rproc *, u64 da, size_t, bool *); // 设备地址→虚拟地址
       int  (*parse_fw)(struct rproc *, const struct firmware *);
       int  (*handle_rsc)(struct rproc *, u32, void *, int, int);  // 处理资源表项
       struct resource_table *(*find_loaded_rsc_table)(...);
       struct resource_table *(*get_loaded_rsc_table)(...);
       int  (*load)(struct rproc *, const struct firmware *);      // 加载固件段
       int  (*sanity_check)(struct rproc *, const struct firmware *);
       void (*coredump)(struct rproc *);
   };
 ```

 注意：prepare/start/stop/attach/detach 是可选的——不同平台实现不同子集。比如你这个代码库里的 Qualcomm ADSP：

 ```c
   static const struct rproc_ops adsp_ops = {
       .unprepare = adsp_unprepare,
       .start     = adsp_start,
       .stop      = adsp_stop,
       .da_to_va  = adsp_da_to_va,
       .parse_fw  = qcom_register_dump_segments,
       .load      = adsp_load,       // 通过 PAS 让 TZ 去加载固件
       .panic     = adsp_panic,
   };
 ```

 四、启动流程（rproc_boot 时序）

 这是最关键的一条链，你的补丁就插在这里：

 ```
   用户/内核调用 rproc_boot(rproc)
     │
     ├─ mutex_lock(&rproc->lock)          ← 你补丁的准入检查在这里之后
     ├─ if (rproc->state == RPROC_DELETED) 报错
     ├─ if (rproc->state == RPROC_OFFLINE)  → rproc_boot 走"冷启动"
     │     ├─ request_firmware()           加载固件文件
     │     ├─ rproc_parse_fw()             解析固件
     │     │     └─ ops->parse_fw() / handle_rsc() 逐项处理 resource table
     │     ├─ rproc_start()
     │     │     ├─ ops->prepare()
     │     │     ├─ ops->load()            把固件搬到远程核内存
     │     │     ├─ ops->start()           上电、释放复位、握手
     │     │     └─ rproc_virtio 初始化 vdev、通知远程核
     │     └─ state = RPROC_RUNNING
     │
     └─ else (state==RPROC_ATTACHED/DETACHED) → rproc_attach() 接管
           └─ ops->attach() + __rproc_attach() 重建 vdev
 ```

 关掉是 rproc_shutdown()：ops->stop() → ops->unprepare() → state=OFFLINE。注意它有引用计数 power，多个使用者时只有最后
 一个 rproc_shutdown 真正关。

 崩溃恢复是 rproc_trigger_recovery()：stop → 重新 request_firmware → start（你这套里由 qcom_sysmon 收到远程核的 panic
 通知后触发）。

 五、resource table / virtio / rpmsg 的关系

 这是 remoteproc 里最容易绕晕的部分，理清一条线：

 ```
   固件里内嵌一段 resource_table（ELF 的 .resource_table 段）
      │
      │  里面列出：
      │   - 共享内存区域 (carveout / vdev buffer / trace buffer)
      │   - virtio 设备 (vdev)：每个 vdev 有 id、status、vring 地址
      │   - 远程核的 trace 缓冲
      ▼
   remoteproc core 解析它
      ├─ 对每个 vdev → 创建 virtio_device (remoteproc_virtio.c)
      │                  └─ rpmsg 驱动绑定上去 → 建立 rpmsg 通道
      │                     └─ 上层用 rpmsg 和远程核收发消息（如 APR/GLINK 之下）
      ├─ 对每个 trace → debugfs 里出现 trace 文件
      └─ 对每个 carveout → iommu/预留内存映射
 ```

 所以：remoteproc 是"底座"，virtio 是"传输"，rpmsg 是"消息"。你设备的 ADSP 通信（sensor/audio 走 GLINK/APR）最终都建立
 在 remoteproc 之上。

# 六、Qualcomm 平台的具体实现

你这个代码库里 Qualcomm 相关文件：

| 文件 | 作用 |
|---|---|
| `qcom_q6v5.c` | 通用的 Q6v5 核启动/停止、watchdog、handover |
| `qcom_common.c` | 通用工具：SSR 通知、子系统重启、subsystem_get 等 |
| `qcom_sysmon.c` | 系统监控：接收远程核的 panic/ssr 通知，触发 recovery |
| `qcom_q6v5_pas.c` | ADSP/CDSP/MPSS 用这个（PAS = Peripheral Authentication Service，固件由 TZ 校验后加载） |
| `qcom_q6v5_adsp.c` | 另一种 ADSP 驱动（老平台） |
| `qcom_q6v5_mss.c` | Modem (MSS) |
| `qcom_q6v5_wcss.c` | WiFi (WCSS) |
| `qcom_pil_info.c` | 记录固件加载地址等（给 ramdump 用） |
 关键区别：Qualcomm 的 ADSP 固件不是 Linux 直接写进内存的，而是：

 ```
   adsp_load() → qcom_mdt_load() → scm_pas_...（进入 TEE/SCM）
      → 由 TrustZone 校验签名并加载固件到 ADSP 内存
      → 返回后 ops->start() 只做"释放复位 + 握手"
 ```

 所以 ops->load 在 Qualcomm 上是走 SCM（Secure Channel）调用的。