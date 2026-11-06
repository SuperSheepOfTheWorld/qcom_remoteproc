common/ 和 msm-kernel/ 是两棵独立的内核源码树（不同 git 仓库、不同版本）。构建时走 Kleaf 的 mixed build：builtin 代码（进 vmlinux 的）从 common/ 编，.ko 模块从 msm-kernel/ 编。remoteproc core 是 builtin，所以必须改 common/。

### 两棵树的区别

| | common/ | msm-kernel/ |
|---|---|---|
| 定位 | 内核基线（GKI/平台通用内核） | 设备/vendor 内核树（Qualcomm 设备定制） |
| git HEAD | bdad6207b608（你的 FF5 补丁提交） | 94824260e802f（System_wakeup 等） |
| 提供什么 | vmlinux 的 builtin 部分 | vendor 模块 .ko |
| remoteproc 文件数 | 35 个 .c | 37 个 .c |

msm-kernel 独有（common 没有）：
- qcom_spss.c、qcom_tracepoints.c、subsystem_notif_virt.c

Makefile 差异（msm-kernel 更新）：

```makefile
  # msm-kernel 把 qcom_common 拆成带 tracepoints 的版本
  rproc_qcom_common-y := qcom_common.o qcom_tracepoints.o
  obj-$(CONFIG_MSM_QUIN_SUBSYSTEM_NOTIF_VIRT) += subsystem_notif_virt.o
  obj-$(CONFIG_QCOM_SPSS) += qcom_spss.o
```

内容差异（说明是不同版本分支）：

```
  remoteproc_core.c     39 行差异（基本是你的 FF5 补丁）
  remoteproc_virtio.c   相同
  qcom_q6v5_pas.c       2551 行差异  ← 版本差很大
  qcom_q6v5.c           285 行差异
  qcom_common.c         462 行差异
  qcom_sysmon.c         428 行差异
```

### 为什么构建用 common 而不是 msm-kernel

#### 1. 构建配置指定了 base_kernel = common

msm-kernel/meta_kernel_la.bzl：

```python
  if variant == "consolidate":
      base_kernel = "//common:kernel_aarch64_consolidate"   # ← 关键
```

#### 2. Kleaf 的 mixed build 机制

kernel_build.bzl 的 _create_kbuild_mixed_tree()：

```bash
  # 把 base_kernel(common) 的所有文件拷到 KBUILD_MIXED_TREE
  export KBUILD_MIXED_TREE=$(realpath ..._kbuild_mixed_tree)
  cp -a -t ${KBUILD_MIXED_TREE} <common 的文件...>
  ...
  --srcdir ${KBUILD_MIXED_TREE}     # ← 内核源码目录 = common
```

也就是说：编译时的源码根（srcdir）是 common/，msm-kernel/ 作为"当前目录"只负责按 Makefile 编出 .ko。

#### 3. 决定性证据（实测）

- System.map / vmlinux 里 remoteproc.ko 出现在 modules.builtin（builtin），且 vmlinux 的 DWARF comp_dir = /android14-6.1/common
- 之前把补丁放 msm-kernel/ → vmlinux 没有 rproc_ff5_* 符号
- 改到 common/ → vmlinux 有 rproc_ff5_* 符号 ?

#### 4. 模块列表印证

meta_seraph_kernel.bzl 里 in_tree_modules（从 msm-kernel 编的 .ko）：

```
  drivers/remoteproc/qcom_pil_info.ko
  drivers/remoteproc/qcom_q6v5.ko
  drivers/remoteproc/qcom_q6v5_pas.ko      ← 模块，msm-kernel
  drivers/remoteproc/qcom_sysmon.ko
  drivers/remoteproc/rproc_qcom_common.ko
```

注意：没有 remoteproc.ko（core）——因为 core 是 CONFIG_REMOTEPROC=y 的 builtin，不在模块列表里，所以从 common 编。

### 怎么判断改哪棵树（实用规则）

```
  CONFIG_xxx=y  (builtin，进 vmlinux)   → 改 common/
  CONFIG_xxx=m  (模块 .ko)              → 改 msm-kernel/
```

具体到 remoteproc：

| 文件 | 类型 | 改哪里 |
|---|---|---|
| remoteproc_core.c（框架/生命周期） | CONFIG_REMOTEPROC=y | common/ |
| remoteproc_internal.h | builtin | common/ |
| remoteproc_virtio.c / elf_loader.c | builtin | common/ |
| qcom_q6v5_pas.c（ADSP 驱动） | CONFIG_QCOM_Q6V5_PAS=m | msm-kernel/ |
| qcom_q6v5.c / qcom_common.c / qcom_sysmon.c | =m | msm-kernel/ |

> 注意：common/ 里也有 qcom_q6v5_pas.c 等（因为 common 不是纯 AOSP GKI，而是 Qualcomm 的基线树），但由于构建时这些是模块、且模块源码从 msm-kernel 取，所以 模块相关改动要改 msm-kernel。这也是最坑的地方——同一个文件两处都有。

### 对应到你的任务

你的 FF5 补丁改的是 remoteproc core 的生命周期入口（rproc_boot/shutdown/detach/set_firmware/trigger_recovery）和新增诊断节点，全部是 CONFIG_REMOTEPROC=y 的 builtin 代码：

- ? 放 common/drivers/remoteproc/ —— 进 vmlinux（boot.img）
- ? 放 msm-kernel/drivers/remoteproc/ —— 不参与 vmlinux 编译，等于白改

如果后续你要改 ADSP 具体的 PAS 加载逻辑（比如让 adsp_load 走不同路径、或改 qcom_q6v5_pas 的启动流程），那部分在 msm-kernel/drivers/remoteproc/qcom_q6v5_pas.c（模块 .ko，最终打进 vendor_dlkm 或 vendor_boot）。