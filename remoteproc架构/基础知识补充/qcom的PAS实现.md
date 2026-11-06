# Qualcomm ADSP 的 PAS 实现（qcom_q6v5_pas.c）

**核心认知：ADSP 的固件不是 Linux 直接加载的，是 TrustZone 帮忙加载和认证的**

普通 remoteproc（比如 TI 的 DSP）是：Linux 自己把固件 memcpy 到内存 → 自己写复位寄存器启动。

但 Qualcomm 的 ADSP 不是这样，它是 **PAS（Peripheral Authentication Service）** 模式：

```
  Linux（AP）          只负责：把固件搬到"指定中转站" + 通知 TZ
  TrustZone（TZ）      负责：验签（认证固件）→ 搬进 ADSP 内存 → 释放复位 → 开门
  ADSP                 被 TZ 放行后开始跑
```

Linux 不能直接碰 ADSP 的复位/启动寄存器，一切通过 SCM（Secure Channel，`qcom_scm_*`）请求 TZ 代劳。这是这层最反直觉、也最关键的一点。

用比喻：

- ADSP = 一间独立办公室（有自己的 CPU、内存）
- mem_region = 门口指定的"家具暂存区"（物理地址在设备树里写死）
- adsp_load = 把家具（固件段）搬进暂存区
- `qcom_scm_pas_auth_and_reset` = 请**保安（TZ）**来：检查家具合不合法（验签）→ 搬进办公室 → 打开门禁（释放复位）
- adsp_stop = 请保安关门

---

## 一、adsp_probe() —— 开机时准备好一切

设备树里有这个节点（`qcom,seraph-adsp-pas`，就是你 FF5 补丁匹配的那个），probe 时做：

```c
  static int adsp_probe(struct platform_device *pdev)
  {
      adsp_alloc_memory_region(adsp);      // ① 拿到"家具暂存区"
      adsp_init_clock(adsp);               // ② 拿时钟（xo、aggre2）
      adsp_init_regulator(adsp);           // ③ 拿电源（cx、px）
      adsp_pds_attach(adsp, ...);          // ④ 拿电源域（Power Domain）
      qcom_q6v5_init(&adsp->q6v5, ...);    // ⑤ 注册中断（wdt/fatal/ready/handover/sysmon）
      rproc = rproc_alloc(...);            // ⑥ 分配 struct rproc
      rproc->ops = &adsp_ops;              // ⑦ 挂上平台回调
      rproc_add(rproc);                    // ⑧ 注册（← 你的 FF5 probe_add 也在这里被调用）
  }
```

### ① adsp_alloc_memory_region() —— 拿到"家具暂存区"

```c
  node = of_parse_phandle(adsp->dev->of_node, "memory-region", 0);  // 从 DT 拿物理地址
  of_address_to_resource(node, 0, &r);
  adsp->mem_phys = adsp->mem_reloc = r.start;
  adsp->mem_size = resource_size(&r);
  adsp->mem_region = devm_ioremap_wc(adsp->dev, adsp->mem_phys, adsp->mem_size);  // 映射到 CPU
```

关键：这块内存的物理地址是设备树里写死的（ADSP 固件必须放在这个固定位置，TZ 才知道去哪找）。`devm_ioremap_wc` 把它映射成 CPU 能写的地址（`mem_region`）。

### ④⑤ 电源域和中断

- 电源域（PD）：ADSP 要用电，得先开对应的电源域（`proxy_pds`），否则上电也没反应。
- 中断：`qcom_q6v5_init` 注册一堆中断：
    - `wdt`：看门狗（ADSP 卡死）
    - `fatal`：致命错误
    - `ready`：ADSP 启动完成信号（后面 `wait_for_start` 等的就是它）
    - `handover`：ADSP 说"我可以独立运行了，你放手吧"
    - `sysmon`：系统监控（panic/ssr 通知）

---

## 二、adsp_load() —— 把固件搬到中转站

```c
  static int adsp_load(struct rproc *rproc, const struct firmware *fw)
  {
      // ① 把固件的"元数据"（段表、hash）登记给 SCM，让 TZ 知道后面要验什么
      qcom_mdt_pas_init(adsp->dev, fw, rproc->firmware, adsp->pas_id,
                        adsp->mem_phys, &adsp->pas_metadata);

      // ② 把固件的各个段 memcpy 到 mem_region（中转站）
      qcom_mdt_load_no_init(adsp->dev, fw, rproc->firmware, adsp->pas_id,
                            adsp->mem_region, adsp->mem_phys, adsp->mem_size,
                            &adsp->mem_reloc);

      // ③ 记录"固件加载在哪"，给 ramdump 用
      qcom_pil_info_store(adsp->info_name, adsp->mem_phys, adsp->mem_size);
  }
```

`qcom_mdt_load_no_init` 最终调 `__qcom_mdt_load`，就是你在上一节学的 ELF 段加载：

```c
  for (每个 program header) {
      offset = phdr->p_paddr - mem_reloc;          // 相对偏移
      ptr = mem_region + offset;                    // 目标地址
      memcpy(ptr, fw->data + phdr->p_offset, phdr->p_filesz);   // 搬数据
      if (memsz > filesz)
          memset(ptr + filesz, 0, memsz - filesz);  // BSS 清零
  }
```

注意：这里和通用 remoteproc 的 ELF loader 几乎一样，但目的地不是"翻译后的地址"，而是固定的 `mem_region`（中转站）。因为 TZ 只认这个固定位置。

> 还有个小细节：`.mdt` 固件可能是分片的（`.b00`、`.b01`...）。`mdt_load_split_segment()` 会按需加载分片——这就是为什么 `adsp_load` 里有个 `else if (phdr->p_filesz)` 分支。

---

## 三、adsp_start() —— 上电 + 请保安开门（最关键）

```c
  static int adsp_start(struct rproc *rproc)
  {
      qcom_q6v5_prepare(&adsp->q6v5);                              // ① 打开中断
      adsp_pds_enable(adsp, adsp->proxy_pds, ...);                 // ② 开电源域
      clk_prepare_enable(adsp->xo);                                // ③ 开 XO 时钟
      clk_prepare_enable(adsp->aggre2_clk);                        // ④ 开 aggre2 时钟
      regulator_enable(adsp->cx_supply);                           // ⑤ 开 CX 电源
      regulator_enable(adsp->px_supply);                           // ⑥ 开 PX 电源

      qcom_scm_pas_auth_and_reset(adsp->pas_id);                   // ⑦ ★ 请 TZ 验签并释放复位

      qcom_q6v5_wait_for_start(&adsp->q6v5, msecs_to_jiffies(5000));  // ⑧ 等 ADSP 发 ready
      qcom_scm_pas_metadata_release(&adsp->pas_metadata);          // ⑨ 释放元数据
  }
```

### ⑦ 才是真正的"启动"：qcom_scm_pas_auth_and_reset

```c
  int qcom_scm_pas_auth_and_reset(u32 peripheral)
  {
      struct qcom_scm_desc desc = {
          .svc  = QCOM_SCM_SVC_PIL,                    // 服务：PIL
          .cmd  = QCOM_SCM_PIL_PAS_AUTH_AND_RESET,     // 命令：认证 + 复位
          .args[0] = peripheral,                       // 哪个外设（ADSP 的 pas_id）
      };
      qcom_scm_call(__scm->dev, &desc, &res);          // 陷入 TZ
  }
```

这一步进去之后，TZ 做三件事：

1. 从 `mem_phys` 读固件，验证签名（用设备里烧的 root key）
2. 验签通过 → 把固件"搬进" ADSP 的真实内存
3. 释放 ADSP 的复位（ADSP 开始执行）

**如果验签失败，直接返回错误，ADSP 不会启动。** 这就是为什么 Qualcomm 设备的固件不能随便替换——必须签名。

### ⑧ 等 ADSP 回话

`qcom_q6v5_wait_for_start` 就是在等 ready 中断：

```c
  int qcom_q6v5_wait_for_start(struct qcom_q6v5 *q6v5, int timeout)
  {
      ret = wait_for_completion_timeout(&q6v5->start_done, timeout);  // 等 ready 中断唤醒
      return !ret ? -ETIMEDOUT : 0;
  }
```

ADSP 固件跑起来后，会发一个"我 ready 了"的中断，这里被唤醒 → 返回成功 → `rproc->state = RUNNING`。

**超时（5 秒）怎么办？** 代码里会 `qcom_scm_pas_shutdown` 回滚，避免 ADSP 卡在半启动状态。

---

## 四、adsp_stop() —— 请保安关门

```c
  static int adsp_stop(struct rproc *rproc)
  {
      qcom_q6v5_request_stop(&adsp->q6v5, adsp->sysmon);   // ① 礼貌地请求停（等 stop-ack）
      qcom_scm_pas_shutdown(adsp->pas_id);                 // ② ★ 请 TZ 关掉 ADSP
      handover = qcom_q6v5_unprepare(&adsp->q6v5);         // ③ 关中断
      if (handover)
          qcom_pas_handover(&adsp->q6v5);                  // ④ handover 情况下重新开
  }
```

- ① 先发"请你停"的消息（通过 sysmon/rpmsg），等 ADSP 回应 stop-ack。
- ② `qcom_scm_pas_shutdown` 又是进 TZ——真正关电/复位也是 TZ 做。
- ③④ 清理中断，处理 handover（ADSP 说"我自己能跑，你别管"的场景）。

注意：`adsp_stop` 里没有 `regulator_disable` / `clk_disable`！因为 `adsp_stop` 之后 core 还会调 `ops->unprepare`（`adsp_unprepare`），那才是关时钟/电源的地方。这是分层的：

```
  rproc_shutdown()
    ├─ rproc_stop() → ops->stop()      → adsp_stop()      （请 TZ 关门）
    ├─ ops->unprepare()                → adsp_unprepare() （关时钟/电源/PD）
    └─ rproc_resource_cleanup()
```

---

## 五、adsp_da_to_va() —— 地址翻译

```c
  static void *adsp_da_to_va(struct rproc *rproc, u64 da, size_t len, bool *is_iomem)
  {
      offset = da - adsp->mem_reloc;                    // 相对中转站的偏移
      if (offset < 0 || offset + len > adsp->mem_size)
          return NULL;                                  // 越界
      *is_iomem = true;                                 // 标记：这是 IO 内存
      return adsp->mem_region + offset;                 // CPU 虚拟地址
  }
```

作用：把"ADSP 眼里的地址（da）"翻译成"CPU 能写的地址（`mem_region + offset`）"。你在上一节学的 ELF loader 里 `rproc_da_to_va()` 最终调的就是它。

注意 `is_iomem = true`：因为 `mem_region` 是 `devm_ioremap_wc` 映射的（不是普通 RAM），所以后面拷贝要用 `memcpy_toio` 而不是 `memcpy`。

---

## 六、adsp_ops —— 平台回调清单

```c
  static const struct rproc_ops adsp_ops = {
      .unprepare = adsp_unprepare,   // 关时钟/电源
      .start     = adsp_start,       // 上电 + SCM 认证 + 等 ready
      .stop      = adsp_stop,        // SCM shutdown
      .da_to_va  = adsp_da_to_va,    // 地址翻译
      .parse_fw  = qcom_register_dump_segments,  // 解析固件（登记 dump 段）
      .load      = adsp_load,        // 搬固件到中转站
      .panic     = adsp_panic,       // 崩溃时读 TZ 的 panic 寄存器
  };
```

对比通用 remoteproc：它没有实现 `sanity_check`、`find_loaded_rsc_table` 等——因为固件格式是 `.mdt`（Qualcomm 私有），走的是 `qcom_mdt_*` 而不是通用 ELF loader。

---

## 七、几个驱动的分工

| 驱动 | 对应硬件 | 特点 |
|---|---|---|
| `qcom_q6v5_pas.c` | ADSP / CDSP / MPSS（新平台） | PAS 模式，TZ 加载认证 |
| `qcom_q6v5_adsp.c` | ADSP（老平台） | 不通过 PAS，直接控制 |
| `qcom_q6v5_mss.c` | Modem（MSS） | 最复杂，带 QMI/DFES |
| `qcom_q6v5_wcss.c` | WiFi（WCSS） | 带 QMI |
| `qcom_q6v5.c` | （公共库） | 中断、handover、wait_for_start |
| `qcom_common.c` | （公共库） | SSR 通知、subsystem_get |
| `qcom_sysmon.c` | （公共库） | 处理远程核的 panic/shutdown 请求 |

你的设备（seraph/blueshark）用的就是 `qcom_q6v5_pas.c`（因为 DT 里是 `qcom,seraph-adsp-pas`）。

---

## 八、对你"在线更新 ADSP 固件"的意义（重点）

这层决定了你能怎么更新、难在哪：

### 1. 固件必须签名，不能随便替换

`qcom_scm_pas_auth_and_reset` 里 TZ 会验签。所以：

- 不能直接往 `mem_region` 里塞一个改过的 `adsp.mbn` 就启动——TZ 会拒绝。
- 必须用签过名的固件（Qualcomm 或你们用私钥签的）。
- 你之前刷的 modem.img 能生效，是因为它是官方签名过的。

### 2. 更新流程就是"停 → 换 → 起"

```c
  rproc_shutdown(rproc);        // → adsp_stop() → TZ shutdown
  // 把新的 adsp.mbn 放到文件系统里
  rproc_set_firmware(rproc, "adsp.mbn");   // 换固件名（或替换固件内容）
  rproc_boot(rproc);            // → adsp_load() + adsp_start() → TZ 重新认证启动
```

这就是你 FF5 补丁要保护的那 5 个入口——因为更新期间如果有人并发 `rproc_boot/shutdown`，会乱套。

### 3. 难点（也就是你补丁要解决的）

| 难点 | 对应 |
|---|---|
| 引用计数：ADSP 上有 sensor/audio 在用，power 降不到 0，关不掉 | 你的 `ff5_update_votes` 账本 |
| 并发控制：更新时别人来 boot/shutdown | 你的 `ff5_update_gate_probe` + 准入点 |
| TZ 认证：新固件必须签名 | 只能走 PAS 流程，不能绕过 |
| 依赖重连：ADSP 重启后 rpmsg/GLINK 要重连 | 需要上层（sensor/audio 驱动）配合 |

### 4. 还有一个 Qualcomm 特有的坑：PAS metadata

`adsp_load` 里 `qcom_mdt_pas_init(..., &adsp->pas_metadata)` 会向 SCM 登记"元数据上下文"（哪些段、hash 是什么）。这个上下文在 `auth_and_reset` 成功后释放（`qcom_scm_pas_metadata_release`）。如果更新流程中途失败，要确保 `adsp_unprepare` 里也释放（代码里确实处理了）——否则下次更新会失败。这是做在线更新时容易踩的坑。

---

## 九、一句话总结

```
  qcom_q6v5_pas.c 这层做的事：
     probe  : 拿中转站内存、时钟、电源、中断
     load   : 把固件段搬到固定物理地址的中转站（mem_region）
     start  : 上电（PD/clk/regulator）→ 请 TZ 验签+释放复位 → 等 ADSP 发 ready
     stop   : 请 TZ 关掉 ADSP
     da_to_va: 把 ADSP 地址翻译成 CPU 可写地址

  核心：Linux 只负责"搬家具 + 通知保安"，真正"验货 + 开门"的是 TrustZone。
```

这就是为什么做在线更新这么难：你不能像普通 remoteproc 那样"自己写寄存器重启"，一切要过 TZ 那道门；再加上引用计数和并发控制，所以才需要你先用补丁把"能不能安全停下、会不会被抢"变成可观测的数字。