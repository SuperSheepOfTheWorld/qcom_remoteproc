下面按**生命周期**把这些函数串起来讲。你先抓住三个全局概念，后面每个函数就都好理解了：

1. **两种“启动”来源**：`rproc_boot()`（内核自己加载固件启动）vs `rproc_attach()`（别的实体已经启动了远程核，我们只是“接上去”）。
2. **状态机** `rproc->state`：`OFFLINE / RUNNING / CRASHED / DETACHED / ATTACHED / DELETED`。
3. **引用计数** `rproc->power`：`rproc_boot/attach` 加，`rproc_shutdown/detach` 减；只有减到 0 才真正关机/分离。

---

## 1. `rproc_alloc()` / `rproc_add()` —— 创建与注册

### `rproc_alloc()`

只**分配**一个 `struct rproc`，不注册。关键动作：

```c
rproc = kzalloc(sizeof(struct rproc) + len, GFP_KERNEL);
rproc->priv = &rproc[1];              // 平台驱动私有数据跟在后面
rproc->auto_boot = true;              // 默认开机自动 boot
device_initialize(&rproc->dev);
rproc->dev.parent = dev;              // 父设备 = 平台设备
rproc->dev.type = &rproc_type;        // release = rproc_type_release
idr_init(&rproc->notifyids);          // vring notifyid 分配器
rproc->index = ida_alloc(&rproc_dev_index, GFP_KERNEL);  // remoteprocN 的 N

rproc_alloc_firmware(...);            // rproc->firmware
rproc_alloc_ops(rproc, ops);          // 拷贝 ops，并填默认实现
...
rproc->state = RPROC_OFFLINE;         // 初始状态
```

`rproc_alloc_ops()` 里有个重要点：**如果平台驱动没提供 `load`，就默认用 ELF loader**：

```c
rproc->ops->load = rproc_elf_load_segments;
rproc->ops->parse_fw = rproc_elf_load_rsc_table;
rproc->ops->find_loaded_rsc_table = ...;
rproc->ops->sanity_check = ...;
rproc->ops->get_boot_addr = ...;
```

所以绝大多数 rproc 驱动（包括 qcom_q6v5_pas）只实现 `start/stop`，ELF 解析和 resource table 处理都走默认。

`rproc_alloc()` 不注册设备，失败路径走 `put_device()` 而不是直接 `kfree`，因为 `device_initialize()` 之后所有权归 device core。

### `rproc_add()`

把 `rproc` 正式注册到内核：

```c
rproc_validate(rproc);              // 状态必须是 OFFLINE 或 DETACHED，且 ops 齐全
rproc_char_device_add(rproc);       // 建 /dev/remoteprocN
device_add(dev);                    // 建 /sys/class/remoteproc/remoteprocN/
rproc_create_debug_dir(rproc);      // 建 /sys/kernel/debug/remoteproc/remoteprocN/

if (rproc->auto_boot)
    rproc_trigger_auto_boot(rproc); // 异步 request_firmware → rproc_auto_boot_callback → rproc_boot

list_add_rcu(&rproc->node, &rproc_list);  // 供 rproc_get_by_phandle 查找
```

注意 `rproc_validate()`：

- `RPROC_OFFLINE` 必须有 `ops->start`
- `RPROC_DETACHED` 必须有 `ops->attach`，且不能有 `cached_table`
- 其他状态直接 `-EINVAL`

`rproc_trigger_auto_boot()` 里分两种情况：

```c
if (rproc->state == RPROC_DETACHED)
    return rproc_boot(rproc);        // 直接 attach，不用等固件

return request_firmware_nowait(...); // 异步加载固件，回调里调 rproc_boot
```

**这里体现了 attach 路径的设计：DETACHED 状态下不需要固件，直接 attach。**

---

## 2. `rproc_boot()` —— 启动的入口

```c
mutex_lock_interruptible(&rproc->lock);

if (rproc->state == RPROC_DELETED) { ret = -ENODEV; goto unlock; }

/* 引用计数：>1 说明已经被别人 boot 了，直接返回成功 */
if (atomic_inc_return(&rproc->power) > 1) { ret = 0; goto unlock; }

if (rproc->state == RPROC_DETACHED) {
    ret = rproc_attach(rproc);       // 路径 A：attach
} else {
    ret = request_firmware(&firmware_p, rproc->firmware, dev);
    ret = rproc_fw_boot(rproc, firmware_p);  // 路径 B：自己加载固件启动
    release_firmware(firmware_p);
}

if (ret) atomic_dec(&rproc->power);  // 失败回滚引用
mutex_unlock(&rproc->lock);
```

三个关键点：

- **`atomic_inc_return` 是引用计数核心**。第一次 boot 从 0→1，继续走；第二次 boot 1→2，直接返回，不会重复启动。
- **根据 `state` 分流**：`DETACHED` → `rproc_attach()`；其他（正常是 `OFFLINE`）→ `request_firmware` + `rproc_fw_boot()`。
- **失败要回滚 `power`**，否则以后再也 boot 不起来。

### 路径 B：`rproc_fw_boot()` 做什么

这是“自己加载固件启动”的主流程：

```c
rproc_fw_sanity_check(rproc, fw);        // 校验固件格式
rproc_enable_iommu(rproc);               // 有 IOMMU 就建 domain 并 attach
rproc_prepare_device(rproc);             // ops->prepare（平台相关，如 PAS 的 prepare）
rproc->bootaddr = rproc_get_boot_addr(rproc, fw);

rproc_parse_fw(rproc, fw);               // 解析 resource table / coredump 段
rproc->max_notifyid = -1;
rproc->nb_vdev = 0;

rproc_handle_resources(rproc, rproc_loading_handlers);   // 处理资源
rproc_alloc_registered_carveouts(rproc);                  // 分配 carveout
rproc_start(rproc, fw);                                   // 真正启动
```

`rproc_start()` 里：

```c
rproc_load_segments(rproc, fw);          // 把 ELF 段搬到目标内存
loaded_table = rproc_find_loaded_rsc_table(...);
memcpy(loaded_table, rproc->cached_table, rproc->table_sz);  // 把改好的表写回固件区
rproc->table_ptr = loaded_table;

rproc_prepare_subdevices(rproc);
rproc->ops->start(rproc);                // 平台驱动的 start（PAS 就是这里触发 Q6 启动）
rproc_start_subdevices(rproc);
rproc->state = RPROC_RUNNING;
```

**`cached_table` → `table_ptr` 的切换很重要**：解析时 `cached_table` 是内核里的副本，改 carveout/vring 信息都改它；真正启动前 copy 到固件所在的内存，`table_ptr` 指向固件里的那份，之后远程核看到的就是更新后的表。

### 路径 A：`rproc_attach()`

和 `rproc_fw_boot()` 类似，但**不加载固件**：

```c
rproc_enable_iommu(rproc);
rproc_prepare_device(rproc);
rproc_set_rsc_table(rproc);              // 从 ops->get_loaded_rsc_table 拿表
rproc_handle_resources(...);             // 同样处理资源
rproc_alloc_registered_carveouts(rproc);
__rproc_attach(rproc);                   // ops->attach → state = RPROC_ATTACHED
```

`rproc_set_rsc_table()` 有个细节：如果 `ops->detach` 存在，会 `kmemdup` 一份 `clean_table`，用来在 detach 时把固件里的表恢复原样，方便下次 re-attach。

---

## 3. `rproc_shutdown()` / `rproc_detach()` —— 关闭/分离

### `rproc_shutdown()`

```c
if (rproc->state != RPROC_RUNNING && rproc->state != RPROC_ATTACHED) {
    ret = -EINVAL; goto out;
}

/* 引用计数：不为 0 说明还有别人在用，直接返回 */
if (!atomic_dec_and_test(&rproc->power))
    goto out;

rproc_stop(rproc, false);
rproc_resource_cleanup(rproc);           // 释放 carveout/trace/mapping/vdev
rproc_unprepare_device(rproc);
rproc_disable_iommu(rproc);
kfree(rproc->cached_table); rproc->cached_table = NULL;
rproc->table_ptr = NULL; rproc->table_sz = 0;
```

`rproc_stop()` 里：

```c
rproc_stop_subdevices(rproc, crashed);
rproc_reset_rsc_table_on_stop(rproc);    // 把 table_ptr 切回 cached_table
rproc->ops->stop(rproc);                 // 平台驱动停核
rproc_unprepare_subdevices(rproc);
rproc->state = RPROC_OFFLINE;
```

**`atomic_dec_and_test` 是和 `rproc_boot` 的 `atomic_inc_return` 配对**：只有最后一个使用者关闭时才真正关机，否则只是减引用。

### `rproc_detach()`

和 `rproc_shutdown()` 结构几乎一样，区别：

- 要求 `state == RPROC_ATTACHED`
- 调 `__rproc_detach()` 而不是 `rproc_stop()`
- 结果是 `state = RPROC_DETACHED`

`__rproc_detach()` 里会调 `rproc_reset_rsc_table_on_detach()`，把固件里的 resource table 用 `clean_table` 恢复，这样下次 attach 时表是干净的。

**总结这两个函数**：

```
rproc_boot / rproc_attach     →  power++  →  state = RUNNING / ATTACHED
rproc_shutdown / rproc_detach →  power--  →  power==0 时 state = OFFLINE / DETACHED
```

---

## 4. `rproc_trigger_recovery()` —— 崩溃恢复

崩溃路径是异步的：

```c
rproc_report_crash(rproc, type);
    pm_stay_awake(rproc->dev.parent);
    queue_work(rproc_recovery_wq, &rproc->crash_handler);

rproc_crash_handler_work():
    mutex_lock(&rproc->lock);
    if (state == RPROC_CRASHED) return;          // 只处理第一次
    if (state == RPROC_OFFLINE)  goto out;       // 已停的不恢复
    state = RPROC_CRASHED;
    ++crash_cnt;
    mutex_unlock(&rproc->lock);

    if (!recovery_disabled)
        rproc_trigger_recovery(rproc);

    trace_android_vh_rproc_recovery(rproc);
    pm_relax(rproc->dev.parent);
```

`rproc_trigger_recovery()` 本身：

```c
mutex_lock_interruptible(&rproc->lock);

/* 加锁后状态可能变了，必须再检查一次 */
if (rproc->state != RPROC_CRASHED)
    goto unlock_mutex;

if (rproc_has_feature(rproc, RPROC_FEAT_ATTACH_ON_RECOVERY))
    ret = rproc_attach_recovery(rproc);   // 先 __rproc_detach 再 __rproc_attach
else
    ret = rproc_boot_recovery(rproc);     // rproc_stop + coredump + request_firmware + rproc_start

mutex_unlock(&rproc->lock);
```

两条恢复路径：

- **`rproc_boot_recovery()`**：`rproc_stop(rproc, true)` → `ops->coredump()` → 重新 `request_firmware` → `rproc_start()`。适合“固件由我们加载”的核（PAS 常规路径）。
- **`rproc_attach_recovery()`**：`__rproc_detach()` → `__rproc_attach()`。适合“核还在跑、只是我们和它失联”的场景（ATTACH_ON_RECOVERY）。

**注意 `crash_handler_work` 里 `state` 检查和 `rproc_trigger_recovery` 里再加锁检查一次**，是因为中间可能被 `rproc_shutdown` 抢先改了状态。这个“双重检查”是你在 FF5 补丁里改生命周期入口时最容易踩坑的地方。

---

## 5. Resource Table 解析

这条链路是：

```c
rproc_parse_fw(rproc, fw)
    → rproc->ops->parse_fw(rproc, fw)        // 默认 rproc_elf_load_rsc_table
        → 找到固件里的 resource table
        → kmemdup 到 rproc->cached_table
        → rproc->table_ptr = rproc->cached_table
        → rproc->table_sz = ...

rproc_handle_resources(rproc, handlers)
    for (i = 0; i < table->num; i++) {
        hdr = table_ptr + table->offset[i];
        switch (hdr->type) {
            RSC_CARVEOUT → rproc_handle_carveout
            RSC_DEVMEM   → rproc_handle_devmem
            RSC_TRACE    → rproc_handle_trace
            RSC_VDEV     → rproc_handle_vdev
            vendor 区间 → rproc_handle_rsc
        }
    }
```

每个 handler 的职责：

- **`rproc_handle_carveout`**：登记 `rproc_mem_entry` 到 `rproc->carveouts`，此时不分配，只记录 `da/len/name`。
- **`rproc_handle_devmem`**：`iommu_map()` 把外设物理地址映射到设备地址，记录到 `rproc->mappings`。
- **`rproc_handle_trace`**：建 `rproc_debug_trace`，在 debugfs 建 `traceN` 文件。
- **`rproc_handle_vdev`**：登记 vring，后面 `rproc_alloc_vring()` 分配 notifyid，最后 `platform_device_register_data(dev, "rproc-virtio", ...)` 创建 virtio 设备。

然后 `rproc_alloc_registered_carveouts()` 真正调每个 `entry->alloc()`（默认 `rproc_alloc_carveout` → `dma_alloc_coherent`），并把分配到的 `pa/da/len` 写回 resource table。这就是为什么 **carveout 要先登记、后分配、最后回填表**。

三个 table 指针的关系，一句话记住：

```
cached_table : 内核里的可修改副本（解析后、启动前改它）
table_ptr    : 当前“生效”的表（启动后指向固件内存里的那份）
clean_table  : 原始干净副本（attach 场景下 detach 时恢复用）
```

---

## 6. 状态机 + 引用计数 总览

```
                 rproc_alloc
                     │
                  OFFLINE ─────────────────────────────┐
                     │ rproc_boot (request_firmware)    │
                     ▼                                  │
                  RUNNING ──crash──► CRASHED ──recovery─┘
                     │                    │
                     │ rproc_shutdown     │ rproc_boot_recovery
                     ▼                    ▼
                  OFFLINE              RUNNING

                 DETACHED ──────────────┐
                     │ rproc_boot(attach)│
                     ▼                   │
                 ATTACHED ──crash──► CRASHED ──attach_recovery──► ATTACHED
                     │
                     │ rproc_detach
                     ▼
                 DETACHED
```

引用计数：

```
rproc_boot()      atomic_inc_return(&power) > 1 → 直接返回
rproc_shutdown()  atomic_dec_and_test(&power) == 0 → 才真正 stop
rproc_attach()    atomic_inc_return(&power) > 1 → 直接返回
rproc_detach()    atomic_dec_and_test(&power) == 0 → 才真正 __rproc_detach
```

**这就是 remoteproc 的核心设计：`power` 管“有没有人用”，`state` 管“现在处于哪个阶段”，`cached_table/table_ptr` 管“resource table 当前在哪份”。**

---
