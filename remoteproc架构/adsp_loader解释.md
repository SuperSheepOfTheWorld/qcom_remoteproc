# adsp-loader 驱动解读

## 一、它是什么

adsp-loader 是 audio-kernel 里的一个 platform driver（`qcom,adsp-loader`），职责只有一句话：

> 在音频要用 ADSP 之前，把 ADSP "拉起来"（boot），并提供一个 sysfs 开关让用户态控制它。

它自己不加载固件（固件是 remoteproc/PAS 的事），它只是调用 remoteproc 的 API 投一票。

---

## 二、数据结构：注意它"没有"什么

```c
  struct adsp_loader_private {
      void *pil_h;              // ← 其实是 struct rproc *（rproc_get_by_phandle 拿到的）
      struct kobject *boot_adsp_obj;
      struct attribute_group *attr_group;
      char *adsp_fw_name;       // 从 fuse/DT 读到的固件变体名
      bool ssr_triggered;       // SSR 标志
  };                            // ★ 注意：没有 vote_held！
```

还有两个全局变量（不是 priv 里的）：

```c
  static struct work_struct adsp_ldr_work;      // 全局 work
  static struct platform_device *adsp_private;   // 全局 pdev
```

关键：整个结构体里没有"我是否已经投过票"的字段——这就是后面所有问题的根源。

---

## 三、四个入口（用户态怎么操作它）

它建了 `/sys/kernel/boot_adsp/` 下的两个节点（`adsp_loader_init_sysfs`）：

```
  /sys/kernel/boot_adsp/boot    写 1 = 启动；写 0 = 停止
  /sys/kernel/boot_adsp/ssr     写 1 = 触发 SSR（重启 ADSP）
```

### 入口 1：adsp_boot_store（写 boot）

```c
  if (boot == BOOT_CMD)            // 写 1
      adsp_loader_do(adsp_private);   // → schedule_work(&adsp_ldr_work)
  else if (boot == IMAGE_UNLOAD_CMD)  // 写 0
      adsp_loader_unload(adsp_private);  // → rproc_shutdown()
```

注意：写 1 是异步的（丢给 workqueue），写 0 是同步的。

### 入口 2：adsp_ssr_store（写 ssr）

```c
  priv->ssr_triggered = true;
  rproc_shutdown(adsp_dev);      // ← 无条件 shutdown！
  adsp_loader_do(adsp_private);  // 再排 work 重新 boot
```

### 入口 3：adsp_load_fw（work 线程，真正干活）

这是核心，写 1 后由 workqueue 异步执行：

```c
  static void adsp_load_fw(struct work_struct *adsp_ldr_work)
  {
      // 1. 读 DT 的 qcom,adsp-state
      of_property_read_u32(..., "qcom,adsp-state", &adsp_state);

      // 2. SSR 后清标志
      if (priv->ssr_triggered) { priv->ssr_triggered = false; ... }

      // 3. 如果有 qcom,proc-img-to-load 且是 modem，按状态 boot
      if (!strcmp(rproc->name, "modem")) {
          if (adsp_state == SPF_SUBSYS_DOWN)
              rproc_boot(priv->pil_h);      // ← 投票
      }

  load_adsp:
      // 4. 等 APM 就绪（最多 10 秒）
      adsp_state = spf_core_is_apm_ready(ADSP_LOADER_APM_TIMEOUT_MS);  // 10000ms

      if (adsp_state == SPF_SUBSYS_DOWN) {
          if (priv->adsp_fw_name)
              rproc_set_firmware(priv->pil_h, priv->adsp_fw_name);  // 用变体固件名
          rc = rproc_boot(priv->pil_h);     // ← 投票（power++）
          if (rc) goto fail;
      } else if (adsp_state == SPF_SUBSYS_LOADED) {
          // 已经加载了，什么都不做
      }
      return;
  fail:
      dev_err(...);   // 只打印，没记录
  }
```

### 入口 4：adsp_loader_unload / adsp_loader_remove

```c
  // boot 写 0 时
  static void adsp_loader_unload(struct platform_device *pdev)
  {
      if (priv->pil_h)
          rproc_shutdown(priv->pil_h);      // ← 无条件 shutdown
  }

  // 模块卸载时
  static int adsp_loader_remove(struct platform_device *pdev)
  {
      if (priv->pil_h) {
          rproc_shutdown(priv->pil_h);      // ← 无条件 shutdown
          priv->pil_h = NULL;
      }
      // 删 sysfs
      // ★ 没有 rproc_put()！★ 没有 cancel_work_sync()！
  }
```

### 入口 5：adsp_loader_probe（开机时）

```c
  static int adsp_loader_probe(struct platform_device *pdev)
  {
      // 1. 拿 rproc 对象（对象引用 +1）
      adsp = rproc_get_by_phandle(rproc_phandle);
      if (!adsp) return -EPROBE_DEFER;

      // 2. 建 sysfs 节点（/sys/kernel/boot_adsp/{boot,ssr}）
      adsp_loader_init_sysfs(pdev);
      priv->pil_h = adsp;

      // 3. 读 nvmem fuse / DT，决定固件变体名 adsp_fw_name
      //    （adsp_variant fuse → adsp-fw-names 里选一个）

  wqueue:
      INIT_WORK(&adsp_ldr_work, adsp_load_fw);   // ★ 注意位置：最后才初始化 work
      return 0;
  }
```

注意：probe 不 boot ADSP，只是"准备好"（拿对象、建节点、初始化 work）。真正 boot 要等用户态写 `boot=1`。

---

## 四、正常启动流程（时序图）

```
  开机
   │
   ├─ adsp_loader_probe()
   │    ├─ rproc_get_by_phandle()  → pil_h      （objrefs = 1）
   │    ├─ 建 /sys/kernel/boot_adsp/{boot,ssr}
   │    └─ INIT_WORK(adsp_load_fw)
   │
   │  (音频 HAL 需要 ADSP)
   ▼
  写 /sys/kernel/boot_adsp/boot = 1
   │
   ├─ adsp_boot_store() → schedule_work
   ▼
  adsp_load_fw()（work 线程）
   ├─ spf_core_is_apm_ready(10s)
   ├─ (可选) rproc_set_firmware(adsp_fw_name)
   └─ rproc_boot()  →  rproc->power = 1   ★ ADSP 起来了
   │
   ▼
  audio 通过 APR/GLINK 使用 ADSP（另一条通道，不是这个文件）
   │
   │  (音频用完)
   ▼
  写 /sys/kernel/boot_adsp/boot = 0
   │
   └─ adsp_loader_unload() → rproc_shutdown() → power = 0
```

---

## 五、它和 remoteproc 的接口（4 个 API）

| API | 作用 | 对应你学的概念 |
|---|---|---|
| `rproc_get_by_phandle()` | 拿 `struct rproc *` 对象 | 对象引用（objrefs++） |
| `rproc_set_firmware()` | 换固件名（变体） | 改 `rproc->firmware` |
| `rproc_boot()` | 投运行票 | power++ |
| `rproc_shutdown()` | 撤运行票 | power-- |
| `rproc_put()` | 放对象引用 | objrefs--（★ 这版从来没调） |

---

## 六、这版代码的 5 个坑（正好是下个提交修的）

现在你再看你的提交说明，就全对上了：

| # | 坑 | 代码位置 | 后果 |
|---|---|---|---|
| 1 | 没有 vote_held，不知道自己有没有票 | 整个结构体 | 后面的重复投票/误释放无从判断 |
| 2 | 重复投票：APM 10s 未就绪就再 rproc_boot | adsp_load_fw 的 `load_adsp:` | power 变成 2，关不掉 |
| 3 | 误释放：ssr/unload/remove 都无条件 rproc_shutdown | adsp_ssr_store / adsp_loader_unload / adsp_loader_remove | 减掉别人的票，或误停 ADSP |
| 4 | 对象引用泄漏：probe 里 rproc_get_by_phandle，remove 里从不 rproc_put | adsp_loader_remove | objrefs 永远不归零 |
| 5 | work 竞态：INIT_WORK 在 adsp_loader_init_sysfs（建节点）之后，且 remove 没 cancel_work_sync | probe 的 `wqueue:` / remove | 节点刚出现被写时排了未初始化的 work；remove 后 work 还在跑 |
| 6 | 错误被忽略 | `goto fail` 只打印 | 出问题查不到 |

用一句话概括这版的设计缺陷：

> 它把"投票"当成无状态的操作——每次需要就 `rproc_boot()`，不需要就 `rproc_shutdown()`，从不记录自己到底有没有票。这在单用户、正常路径下能跑，但一旦有"重复请求"（APM 超时）、"多个入口"（ssr/boot/unload/remove）、"并发"（work vs sysfs），power 就乱了。

而这恰恰是在线更新 ADSP 固件的致命前提：更新要求 power 能干净地归零。所以下个提交（`4bcbf1aa`）才要给它加 `vote_held` + `vote_lock`，把它从"无状态"变成"有状态、只撤自己的票"。

## Patch

### 02B:把 adsp-loader 从"无状态"变成"有状态"

这次改动把 adsp-loader 从"无状态"（每次需要就 `rproc_boot`，不需要就 `rproc_shutdown`，从不记录自己有没有票）变成"有状态"（用 `vote_held` 记录自己是否持票，用 `vote_lock` 串行化所有决策）。

---

### 一、数据结构：加了三个字段

```c
   struct adsp_loader_private {
       void *pil_h;
       struct kobject *boot_adsp_obj;
       struct attribute_group *attr_group;
       char *adsp_fw_name;
       bool ssr_triggered;
  +    struct mutex vote_lock;   // ← 串行化所有 boot/shutdown 决策
  +    bool vote_held;           // ← 我到底有没有票（核心！）
  +    int  last_err;            // ← 最后一次错误码（可观测）
   };
```

还新增了一个只读节点：

```c
  + static struct kobj_attribute adsp_ff5_state_attribute =
  +     __ATTR(ff5_state, 0400, adsp_ff5_state_show, NULL);
```

```
  /sys/kernel/boot_adsp/ff5_state   →  patch=FF5_LOADER_02B vote=1 last_err=0
```

`vote_held` 就是这次改动的灵魂：之前没有它，所以"我该不该 shutdown"无从判断。

---

### 二、逐函数对比

#### 1. adsp_load_fw（work 线程，投票的地方）

##### 修复前

```c
  if (adsp_state == SPF_SUBSYS_DOWN) {
      if (priv->adsp_fw_name)
          rproc_set_firmware(...);
      rc = rproc_boot(priv->pil_h);      // ← 每次都 boot，不管之前 boot 过没
      if (rc) goto fail;                 // 失败只打印
  }
```

##### 修复后

```c
  if (adsp_state == SPF_SUBSYS_DOWN) {
      mutex_lock(&priv->vote_lock);
      if (priv->vote_held) {                      // ★ 已经持票
          dev_warn(dev, "loader already holds its ADSP boot vote");
          mutex_unlock(&priv->vote_lock);
          return;                                 // ★ 不再重复 boot
      }
      if (priv->adsp_fw_name) {
          rc = rproc_set_firmware(...);
          if (rc) {
              priv->last_err = rc;                // ★ 记错误
              mutex_unlock(&priv->vote_lock);
              goto fail;
          }
      }
      rc = rproc_boot(priv->pil_h);
      if (rc) {
          priv->last_err = rc;                    // ★ 记错误
          mutex_unlock(&priv->vote_lock);
          goto fail;
      }
      priv->vote_held = true;                     // ★ 成功了才标记持票
      mutex_unlock(&priv->vote_lock);
  }
```

行为差异（场景：APM 10 秒未就绪）：

```
  修复前：boot 一次(power=1) → 10s 后 APM 还没好 → 又 rproc_boot(power=2)  ❌
  修复后：boot 一次(power=1) → 10s 后 vote_held=true → warn + return       ✅ power 保持 1
```

另外 `ssr_triggered` 的读取/清零也挪进了锁里（用局部变量 `ssr_triggered` 承接），避免和 sysfs 路径竞争。

#### 2. adsp_ssr_store（用户态写 ssr 触发重启）

##### 修复前

```c
  priv->ssr_triggered = true;
  rproc_shutdown(adsp_dev);      // ← 无条件 shutdown！
  adsp_loader_do(adsp_private);  // 再排 work 重启
```

##### 修复后

```c
  mutex_lock(&priv->vote_lock);
  if (priv->vote_held) {                       // ★ 只有自己持票才撤
      rc = rproc_shutdown(adsp_dev);
      if (rc) {
          priv->last_err = rc;
          mutex_unlock(&priv->vote_lock);
          return rc;                           // ★ 失败：不排重启、票保留
      }
      priv->vote_held = false;
  }
  priv->ssr_triggered = true;
  mutex_unlock(&priv->vote_lock);
  adsp_loader_do(adsp_private);
```

行为差异（场景：audio 自己没票，别人（sensor）在用的 ADSP）：

```
  修复前：无条件 rproc_shutdown() → 减掉 sensor 的票 → ADSP 被误关  ❌
  修复后：vote_held=false → 跳过 shutdown → 不碰别人的票           ✅
```

还有一个细节：shutdown 失败时不排重启（直接 `return rc`），因为票还在，重启没意义；只有成功撤票才排重启。

#### 3. adsp_loader_unload（用户态写 boot=0）

##### 修复前

```c
  if (priv->pil_h) {
      rproc_shutdown(priv->pil_h);   // ← 无条件
  }
```

##### 修复后

```c
  mutex_lock(&priv->vote_lock);
  if (priv->pil_h && priv->vote_held) {      // ★ 持票才撤
      rc = rproc_shutdown(priv->pil_h);
      if (rc) {
          priv->last_err = rc;               // 失败：票保留
      } else {
          priv->vote_held = false;
      }
  }
  mutex_unlock(&priv->vote_lock);
```

同样：只撤自己的票。

#### 4. adsp_loader_remove（模块卸载）—— 改动最大

##### 修复前

```c
  if (priv->pil_h) {
      rproc_shutdown(priv->pil_h);   // ← 无条件
      priv->pil_h = NULL;
  }
  if (priv->boot_adsp_obj) {
      sysfs_remove_group(...);
      kobject_del(...);
  }
  // ★ 没有 cancel_work_sync
  // ★ 从不 rproc_put（probe 里 get 的对象引用泄漏）
```

##### 修复后

```c
  /* 1. 先删 sysfs 组（等进行中的写入结束，新写入进不来） */
  if (priv->boot_adsp_obj) {
      sysfs_remove_group(priv->boot_adsp_obj, priv->attr_group);
      kobject_del(priv->boot_adsp_obj);
      priv->boot_adsp_obj = NULL;
  }

  /* 2. 等已排队的 work 跑完（修 work 竞态） */
  cancel_work_sync(&adsp_ldr_work);

  /* 3. 持票才 shutdown，票释放后才 rproc_put */
  if (priv->pil_h) {
      mutex_lock(&priv->vote_lock);
      if (priv->vote_held && !rproc_shutdown(priv->pil_h))
          priv->vote_held = false;

      if (!priv->vote_held)
          rproc_put(priv->pil_h);          // ★ 配对 probe 的 get
      else
          dev_err(dev, "ADSP vote still held, keep rproc ref");

      mutex_unlock(&priv->vote_lock);
      priv->pil_h = NULL;
  }
```

三个关键点：

1. **顺序**：先删 sysfs → 再 `cancel_work_sync` → 最后 shutdown + put。这样保证"没有新的 sysfs 写入、没有正在跑的 work"再动 ADSP。
2. **`rproc_put` 补上了**：修复对象引用泄漏（objrefs 永远不归零的问题）。
3. **shutdown 失败时保留 `rproc_put`（即不 put）**：因为票还在、ADSP 还在跑，如果此时放掉对象引用，PAS 驱动可能被卸载，ADSP 就失控了。所以宁可"钉住"它（注释写 "keep the PAS driver pinned"）。

#### 5. adsp_loader_probe —— 只挪了一行

##### 修复前

```c
  adsp_loader_init_sysfs(pdev);   // ← 先建 sysfs 节点
  ...
  wqueue:
      INIT_WORK(&adsp_ldr_work, adsp_load_fw);   // ← 最后才初始化 work
```

##### 修复后

```c
  INIT_WORK(&adsp_ldr_work, adsp_load_fw);   // ★ 提前到 sysfs 之前
  ret = adsp_loader_init_sysfs(pdev);
  ...
  wqueue:
      // （原来那行删掉了）
```

为什么：`adsp_loader_init_sysfs` 建完节点后，用户态可能立刻写 `boot=1` → `schedule_work` → 但此时 `adsp_ldr_work` 还没 `INIT_WORK` → 操作未初始化的 work（可能 crash）。提前初始化就消除了这个窗口。

#### 6. adsp_loader_init_sysfs —— 加一行

```c
  platform_set_drvdata(pdev, priv);
  + mutex_init(&priv->vote_lock);   // ★ 初始化锁
  priv->pil_h = NULL;
```

#### 7. 新增 adsp_ff5_state_show

```c
  static ssize_t adsp_ff5_state_show(...)
  {
      ...
      mutex_lock(&priv->vote_lock);
      ret = sysfs_emit(buf, "patch=FF5_LOADER_02B vote=%d last_err=%d\n",
                       priv->vote_held, priv->last_err);
      mutex_unlock(&priv->vote_lock);
      return ret;
  }
```

就是你验收时看到的 `vote=1 last_err=0`——让你能直接看到 loader 自己认为"持票了没"，和 remoteproc 的 `ff5_update_votes` 账本对照。

---

### 三、锁的设计（这段很关键，别踩坑）

提交说明里写了锁的顺序，我展开讲：

```
  锁顺序：vote_lock  →  rproc->lock（rproc_boot/shutdown 内部拿）
  方向：单向，不会反过来
```

为什么不会死锁：

- `adsp_load_fw` 持 `vote_lock` 跨越 `rproc_boot()`（内部会拿 `rproc->lock`）→ 锁序 `vote_lock → rproc->lock`。
- remoteproc core 不会回调 loader（没有反向获取 `vote_lock`），所以不会形成环。
- 等 APM 就绪（最长 10 秒）在锁外——避免持锁等 10 秒把其他操作（ssr/unload/ff5_state）全堵死。

work 持锁跨越 `rproc_boot` 是有意的串行化：期间 ssr/unload/ff5_state 会等待，保证同一时刻只有一个决策者。

remove 不持锁等待：`sysfs_remove_group` 和 `cancel_work_sync` 都在不持 `vote_lock` 的情况下做，等它们完成后再拿锁。这样避免 "remove 持锁等 work，work 又在等锁" 的死锁。

---

### 四、用一张表总结

| 函数 | 修复前 | 修复后 |
|---|---|---|
| `adsp_load_fw` | 每次 DOWN 都 `rproc_boot`（可能重复投票） | 已持票则 warn+return；成功才置 `vote_held` |
| `adsp_ssr_store` | 无条件 `rproc_shutdown` | 仅持票才 shutdown；失败不排重启、票保留 |
| `adsp_loader_unload` | 无条件 `rproc_shutdown` | 仅持票才 shutdown；失败记 `last_err` |
| `adsp_loader_remove` | 无条件 shutdown、无 `rproc_put`、无 `cancel_work_sync` | 先删 sysfs → `cancel_work_sync` → 持票才 shutdown → 票释放才 `rproc_put`；失败保留引用 |
| `adsp_loader_probe` | `INIT_WORK` 在 sysfs 之后 | `INIT_WORK` 提前到 sysfs 之前 |
| `adsp_loader_init_sysfs` | — | 加 `mutex_init(&vote_lock)` |
| 新增节点 | — | `ff5_state` 只读，显示 `vote`/`last_err` |

---

### 五、修复前后对比（三个场景演一遍）

#### 场景 A：APM 超时（重复投票）

```
  修复前：boot → power=1 → 10s 后 APM 没好 → boot → power=2  ❌（关不掉）
  修复后：boot → power=1 → 10s 后 vote_held=1 → warn+return  ✅（power=1）
```

#### 场景 B：audio 没票时被要求 SSR

```
  修复前：无条件 shutdown → 减掉 sensor 的票 → ADSP 误关  ❌
  修复后：vote_held=0 → 跳过 → 不动别人的票             ✅
```

#### 场景 C：模块卸载

```
  修复前：shutdown（无条件）+ 从不 rproc_put
          → objrefs 永远不归零 + work 可能在卸载后还在跑  ❌
  修复后：删 sysfs → cancel_work_sync → 持票才 shutdown → 票清了才 rproc_put
          → objrefs 正确递减，work 不会跑飞                ✅
```

---

### 六、边界（保持你提交里的口径）

- 这次只验证了正常路径（`vote=1`、`balanced=1`、`boot ok=1`）。
- 没有真正触发重复 boot、SSR、`boot=0`/unload、remove、在线更新——这些异常/停止路径要等第 05 步受控 STOP/START 验证。
- `objrefs=2`（`audio_notify_probe` + `adsp_loader_probe` 各一次 get）是正常的，因为还在用；对象引用 ≠ 运行票。
- 没有改 core、没有导出新符号、没有改 PAS、没有改 `q6_notifier`（它的引用只用于通知，不投运行票）。

---

### 一句话收尾

> 这次提交不是"加功能"，而是"给投票加账本"——让 loader 从"盲目 boot/shutdown"变成"只投自己的票、只撤自己的票"，并把"我持不持票"暴露成 `ff5_state`。这是让 `rproc->power` 能安全归零、进而实现在线更新 ADSP 固件的前提。