# 中间三层：ELF loader / virtio / coredump

好，接着上面的架构图，把中间这三层讲透。先给一张它们在启动流程里的位置图，你就能把之前学的 core 流程串起来：

```
  rproc_fw_boot()                          ← 你在 core 里学的主流程
    ├─ rproc_parse_fw()
    │     └─ rproc_elf_load_rsc_table()    ← 【ELF loader】找到并解析"资源表"
    │
    ├─ rproc_handle_resources()            ← 逐项处理资源表
    │     ├─ RSC_CARVEOUT → 预留内存
    │     ├─ RSC_TRACE    → debugfs trace
    │     └─ RSC_VDEV     → rproc_virtio    ← 【virtio】把 vdev 变成真设备
    │
    └─ rproc_start()
          └─ rproc_elf_load_segments()     ← 【ELF loader】把固件"段"搬进内存
```

下面一个个讲。

---

## 一、remoteproc_elf_loader.c —— "拆快递 + 找说明书"

### 先理解固件长什么样

ADSP 的固件（adsp.mdt / adsp.mbn）本质是一个 ELF 文件（一种可执行文件格式）。你可以把它想成一个快递箱：

```
  adsp.mbn (ELF)
  ├── ELF 头          ← "这是一个给 ARM 用的箱子"
  ├── Program Header  ← 装箱清单（哪些包裹要送到哪个地址）
  ├── Section Header  ← 更细的目录（每个部件的名字，如 .text/.data/.resource_table）
  └── 数据区
      ├── .text            ← 代码
      ├── .data            ← 数据
      └── .resource_table  ← ★ 说明书（告诉主 CPU 远程核需要什么资源）
```

ELF 加载器干三件事，对应三个函数：

### 1. `rproc_elf_sanity_check()` —— "验货"

```c
  int rproc_elf_sanity_check(struct rproc *rproc, const struct firmware *fw)
```

先看箱子是不是真的 ELF、是不是给对的架构（ARM64）、类型对不对。验不过就直接拒绝，不往下走。

### 2. `rproc_elf_load_segments()` —— "按清单把包裹搬到指定地址"

这是核心。它遍历 ELF 的 Program Header（装箱清单），对每个"要加载的段"（PT_LOAD）：

```c
  for (每个 program header) {
      da    = phdr->p_paddr;      // 目标地址（远程核地址空间里的地址）
      filesz = phdr->p_filesz;    // 文件里有多少字节
      memsz  = phdr->p_memsz;     // 内存里要占多少字节

      ptr = rproc_da_to_va(rproc, da, memsz, &is_iomem);  // 把"设备地址"翻译成"CPU 能写的地址"
      memcpy(ptr, elf_data + offset, filesz);              // 把数据搬过去
      if (memsz > filesz)
          memset(ptr + filesz, 0, memsz - filesz);         // 多出来的部分（BSS）清零
  }
```

具体例子，假设 adsp.mbn 里有两段：

| 段 | 目标地址 p_paddr | 文件大小 filesz | 内存大小 memsz | 干什么 |
|---|---|---|---|---|
| .text | 0x1000000 | 0x1000 | 0x1000 | 把 4KB 代码拷到 0x1000000 |
| .bss | 0x1001000 | 0 | 0x2000 | 文件里没数据，把 0x1001000~0x1003000 清零 |

**为什么要 `rproc_da_to_va`？** 因为 0x1000000 是远程核（ADSP）眼里的地址，不是 CPU 眼里的地址。要通过 IOMMU 映射或 carveout 映射翻译成 CPU 能写的虚拟地址。这正是你之前学的 `rproc_ops->da_to_va` 回调——Qualcomm 的 `adsp_da_to_va` 就是干这个的。

### 3. `find_table()` / `rproc_elf_load_rsc_table()` —— "找出说明书"

注意这里用的是 Section Header（不是 Program Header），按名字找 `.resource_table` 段：

```c
  for (i = 0; i < shnum; i++) {
      if (strcmp(name_table + name, ".resource_table"))
          continue;                       // 不是说明书的段，跳过
      table = (struct resource_table *)(elf_data + offset);   // 找到说明书
      if (table->ver != 1) return NULL;   // 只支持版本 1
      ...
  }
```

找到后，`rproc_elf_load_rsc_table()` 会把它 `kmemdup` 一份到内核内存，存到 `rproc->cached_table`。

**为什么要复制一份？** 因为 core 后面要改这张表（填 carveout 地址、vring 地址等），改的是内核里的副本（`cached_table`），等启动前再拷回固件内存。这就是你之前学的 `cached_table / table_ptr` 的关系。

---

## 二、remoteproc_virtio.c —— "把说明书里的信箱变成真的信箱"

### 先理解"资源表里的 vdev 是什么"

刚才找到的 `.resource_table`（说明书）里，会列出一堆资源项，其中最重要的是 vdev（虚拟设备）：

```
  resource_table:
    ├─ carveout: 给远程核的共享内存
    ├─ trace:    远程核的日志缓冲
    └─ vdev #1:  一个"信箱"，id=1（比如给 sensor 用）
    └─ vdev #2:  另一个"信箱"，id=2（比如给 audio 用）
```

每个 vdev 里有个 vring（虚拟队列）——就是一块共享内存环形队列，主 CPU 和远程核都能读写它，用来传数据。

### 类比

- vdev = 一个"信箱"
- vring = 信箱里的"信件格"（环形队列）
- notify（kick） = 按门铃，告诉对方"有新信了"

### 这个文件干了什么

**1. `rproc_virtio_probe()`（platform driver probe）**：
收到 core 在 `rproc_handle_vdev` 里注册的 `"rproc-virtio"` 平台设备，创建一个标准的 `virtio_device`。

```c
  vdev->id.device = id;                              // 信箱编号
  vdev->config = &rproc_virtio_config_ops;           // 一组回调
  vdev->dev.parent = dev;
  register_virtio_device(vdev);                      // 注册到 virtio 总线
```

**2. `rproc_virtio_find_vqs()`**：找到 vring 的内存，把共享内存和 vdev 关联起来。

```c
  /* 把 vdev 的 buffer 内存池关联到 vdev 子设备 */
  dma_declare_coherent_memory(dev, pa, mem->da, mem->len);
  ...
  register_virtio_device(vdev);
```

**3. `rproc_virtio_notify()`（就是 `rproc_ops->kick`）**：
主 CPU 往 vring 里放了数据后，调用它按门铃通知远程核：

```c
  static void rproc_virtio_notify(struct virtqueue *vq)
  {
      ...
      rproc->ops->kick(rproc, rvq->notifyid);   // ← 最终调用平台驱动的 kick
  }
```

**4. `rproc_virtio_config_ops`**：一组标准 virtio 回调（读/写 status、features 等），让 virtio 框架能操作这个设备。

```c
  static const struct virtio_config_ops rproc_virtio_config_ops = {
      .get_features     = rproc_virtio_get_features,
      .set_status       = rproc_virtio_set_status,
      .get_status       = rproc_virtio_get_status,
      .find_vqs         = rproc_virtio_find_vqs,
      .del_vqs          = rproc_virtio_del_vqs,
      ...
  };
```

### 注册之后发生什么

`register_virtio_device()` 之后，virtio 总线会匹配对应的驱动，比如 rpmsg 驱动绑上去，于是：

```
  ADSP 的 vdev  →  变成 /dev/rpmsg_ctrl0  →  rpmsg 通道  →  上层应用（如 sensor/audio）收发消息
```

这就是为什么你能通过 rpmsg/GLINK 和 ADSP 通信——底层就是这套 virtio。

---

## 三、remoteproc_coredump.c —— "出事故了，把现场打包"

### 场景

ADSP 崩溃了（panic / watchdog），你要分析原因，需要它崩溃那一刻的内存内容。

### 它做什么

把远程核指定的内存段（`dump_segments`）打包成一个 ELF core 文件：

```c
  void rproc_coredump(struct rproc *rproc)
  {
      // 1. 生成 ELF core 头
      // 2. 对每个要 dump 的段：
      for (每个 segment) {
          rproc_copy_segment(rproc, dest, segment);   // 把远程核内存拷出来
      }
      // 3. 通过 dev_coredumpv() 暴露给用户态
  }
```

相关函数：

- `rproc_coredump_add_segment()`：登记"哪些内存段要 dump"（通常在 `parse_fw` 里根据固件信息登记）
- `rproc_coredump()`：真正执行 dump
- `rproc_coredump_read()`：用户态读 `/sys/kernel/debug/remoteproc/remoteprocN/coredump` 时，把打包好的数据返回

### 具体例子

```
  ADSP 崩溃 → ops->coredump() → rproc_coredump()
      → 把 0x1000000~0x1100000（16MB）内存打包成 ELF core
      → 出现 /sys/kernel/debug/remoteproc/remoteproc0/coredump
      → 你用工具读出来，用 gdb/objdump 分析
```

---

## 四、debugfs / sysfs —— 控制面板（回顾 + 补充）

你之前问过这三个，这里只补充它们在这个中间层的角色：

| 接口 | 谁建的 | 建什么 | 用途 |
|---|---|---|---|
| sysfs | `remoteproc_sysfs.c` | state/firmware/name/recovery/coredump | 稳定控制面板（你的 FF5 节点也在这） |
| cdev | `remoteproc_cdev.c` | `/dev/remoteprocN` | 程序化控制（ioctl、fd 生命周期） |
| debugfs | `remoteproc_debugfs.c` | trace/resource_table/carveout_memories/coredump/crash | 调试（不稳定，生产可能不挂载） |

debugfs 里具体能看到什么（配合前面讲的）：

```
  /sys/kernel/debug/remoteproc/remoteproc0/
      resource_table       ← 打印 .resource_table 的内容（就是 ELF loader 找到的那张表）
      carveout_memories    ← 共享内存区域
      trace0/trace1...     ← 远程核的日志缓冲（RSC_TRACE 那项）
      coredump             ← 崩溃转储
      crash                ← 写一下手动触发崩溃（调试用）
```

注意 `resource_table` 这个 debugfs 文件——它输出的就是你在 elf_loader 里找到的说明书。也就是说：ELF loader 找到表 → core 解析表 → debugfs 把表内容打印给你看，形成闭环。

---

## 五、把三块用一个完整例子串起来

假设 ADSP 固件 adsp.mbn 启动：

```
  1. 主 CPU 拿到 adsp.mbn（ELF 文件）
          │
  2. 【ELF loader】sanity_check：验货，是合法 ARM64 ELF ✅
          │
  3. 【ELF loader】load_segments：
          .text  → 拷到 0x1000000（代码）
          .data  → 拷到 0x2000000（数据）
          .bss   → 0x3000000 清零 16MB
          │
  4. 【ELF loader】find_table：在 section header 里找到 .resource_table
          │
  5. 【core】解析表，发现里面有 2 个 vdev（sensor、audio）
          │
  6. 【virtio】给 vdev #1 注册 virtio 设备 → rpmsg 通道 A（sensor 用）
             给 vdev #2 注册 virtio 设备 → rpmsg 通道 B（audio 用）
          │
  7. 【core】rproc_start：ops->start() 释放 ADSP 复位
          │
  8. ADSP 跑起来，通过 vring + notify 和主 CPU 收发消息
          │
  9. 如果 ADSP 崩了：
     【coredump】把内存打包 → debugfs/coredump
     【core】rproc_trigger_recovery → 重新 load_segments → 重启
```

---

## 六、一张表总结

| 文件 | 一句话 | 关键函数 | 产出 |
|---|---|---|---|
| `remoteproc_elf_loader.c` | 拆快递：验货、搬段、找说明书 | sanity_check / load_segments / find_table | 固件段进入内存 + cached_table |
| `remoteproc_virtio.c` | 把说明书里的信箱变成真的 | rproc_virtio_probe / find_vqs / notify | virtio 设备 → rpmsg 通道 |
| `remoteproc_coredump.c` | 出事故拍现场 | rproc_coredump / add_segment | ELF core 文件 |
| `remoteproc_debugfs.c` | 调试后门 | rproc_create_debug_dir | `/sys/kernel/debug/.../trace`、`resource_table` 等 |
| `remoteproc_sysfs.c` | 稳定控制面板 | rproc_attrs | `/sys/class/remoteproc/remoteprocN/*` |

记住这条主线：

> ELF loader 把固件搬进内存并找到"说明书"（resource table）→ core 按说明书分配资源 → virtio 把说明书里的"信箱"变成真的通信通道 → coredump/debugfs 负责出事时留证据、平时给你看内部状态。