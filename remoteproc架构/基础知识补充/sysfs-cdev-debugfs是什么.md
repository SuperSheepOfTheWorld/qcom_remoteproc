## 一、三者都是 Linux 的"内核?用户态"接口，但定位不同

| | sysfs | cdev（字符设备） | debugfs |
|---|---|---|---|
| 挂载点 | `/sys/...` | `/dev/...` | `/sys/kernel/debug/...` |
| 访问方式 | cat / echo（读写文本属性） | open/ioctl/read/write（程序接口） | cat/echo/read |
| 本质 | 内核对象的属性文件 | 设备节点 + 文件操作表 | 调试用文件系统 |
| ABI 稳定性 | 稳定 ABI（不许随意改） | 稳定（ioctl 号是契约） | 不稳定（随时可改/可删） |
| 默认可用性 | 一直有 | 一直有 | 可能没挂载 / 需 root |
| 典型使用者 | 人、shell 脚本、init | 用户态驱动/HAL | 内核开发者 |

### sysfs 是什么

内核把每个 `struct device`（remoteproc 里就是 `struct rproc` 的 `dev`）映射成一个目录，对象里的每个属性对应一个文件。读文件 = 调 show 回调，写文件 = 调 store 回调。

remoteproc 暴露的（remoteproc_sysfs.c）：

```
  /sys/class/remoteproc/remoteprocN/
      state      ← 读: offline/running/crashed...  写: start/stop
      firmware   ← 读/写: 固件文件名（如 qcom/adsp.mbn）
      name       ← 读: adsp / cdsp / modem（只读）
      recovery   ← 读/写: 崩溃后是否自动恢复
      coredump   ← 读/写: 崩溃时是否 dump
```

### cdev 是什么

字符设备：一个设备节点 + `file_operations`，用户态通过 `open()` 拿到 fd，再 `ioctl()` 下发命令。适合结构化的、程序化的、需要会话状态的控制。

remoteproc 的（remoteproc_cdev.c）：

```
  /dev/remoteprocN
      ioctl(RPROC_SET_SHUTDOWN_ON_RELEASE)  ← 设置"关闭 fd 时是否自动 stop"
      ioctl(RPROC_GET_SHUTDOWN_ON_RELEASE)
```

它就是一个"句柄"：打开它 → 保持引用 → 关掉时按策略决定要不要关远程核。这种"生命周期绑定到 fd"的语义，sysfs 的文本读写表达不了。

### debugfs 是什么

专门给调试用的虚拟文件系统，没有任何稳定性保证（内核版本一变就可能改/删），所以绝不能拿它当正式接口。

remoteproc 的（remoteproc_debugfs.c）：

```
  /sys/kernel/debug/remoteproc/remoteprocN/
      name                ← 名字
      recovery            ← 是否允许恢复
      crash               ← 写一下就手动触发一次 crash（调试用）
      resource_table      ← 打印固件里的资源表（共享内存/vring 布局）
      carveout_memories   ← 预留内存区域
      coredump            ← 触发/查看 coredump
      <trace 文件们>       ← 远程核的 trace 缓冲（rproc->traces）
```

## 二、为什么 remoteproc 要同时做三种

因为它们的使用者、稳定性要求、数据形态完全不同：

### 1. sysfs —— 给"人和简单脚本"的稳定开关/观测

- 谁用：init.rc、vendor 脚本、cat/echo 调试、工厂测试。你之前验证 FF5 补丁就是 `cat .../ff5_update_status`。
- 为什么必须稳定：这是对外 ABI，用户态（含 Android 的 init、vendor 二进制）依赖它，不能随便改。
- 为什么是文本：人可读、shell 可拼装，`echo start > state` 就够了。
- 局限：只能传字符串、没有会话概念、不适合复杂/二进制/长事务。

### 2. cdev —— 给"用户态驱动/HAL"的程序化控制

- 谁用：Android 的 remoteproc HAL、固件加载守护进程。
- 为什么需要它：很多操作 sysfs 表达不了，比如：
    - 引用/生命周期语义：打开 `/dev/remoteprocN` 就等于"我要它活着"，关 fd 按策略决定是否 stop（`RPROC_SET_SHUTDOWN_ON_RELEASE`）。
    - ioctl 传递结构化参数：多个字段、二进制 buffer。
    - 独占/权限：设备节点权限 + fd 独占，比"谁都能 echo"更可控。
- 一句话：sysfs 是"给人按的按钮"，cdev 是"给程序用的 API"。

### 3. debugfs —— 给"内核/驱动开发者"的调试后门

- 谁用：开发调试、问题定位。
- 为什么单独一套：这些内容
    - 数据量大/格式内部（trace 缓冲、resource_table 原始结构、carveout 布局）；
    - 可能泄露敏感信息（内存地址、固件布局）；
    - 语义不稳定（跟内核版本强绑定）。

  这些绝不能进稳定 ABI，所以放 debugfs，且默认生产环境可能不挂载。

## 三、三种接口的分层设计哲学（一句话记住）

```
  sysfs    = 稳定的"控制面板"     （人/脚本，文本，ABI 保证）
  cdev     = 稳定的"程序 API"      （HAL/守护进程，ioctl，fd 生命周期）
  debugfs  = 不稳定的"调试后门"    （开发者，原始/大量/敏感数据）
```

这就是为什么 remoteproc 不是"只做一个"，而是按"稳定性 × 使用者 × 数据形态"分三层：该稳定的（sysfs/cdev）给正式功能，该灵活的（debugfs）给调试，互不污染。

## 四、对应到你的任务

这也解释了你补丁里的设计选择：

- 你加的 `ff5_update_status` / `ff5_update_gate_probe` 放在 sysfs（`/sys/class/remoteproc/remoteprocN/`），因为：
    - 需要"简单 cat 就能看"→ 用 sysfs；
    - 需要明确权限（你用了 `0400` + `capable(CAP_SYS_ADMIN)`）→ sysfs 属性天然支持 mode；
    - 是诊断接口，不该污染 debugfs（debugfs 生产可能没挂载，不能作为验收依据）。
- 如果以后做正式的"在线更新 ADSP 固件"接口，更合适的是 cdev（`/dev/remoteprocN` + 新 ioctl），因为更新是长事务、需要会话/互斥/错误码，正是 cdev 擅长、sysfs 不擅长的。
- `resource_table`、`coredump`、`trace` 这种调试用、大数据、格式内部的东西，继续留在 debugfs。

一句话总结：sysfs 面向"稳定观测/开关"，cdev 面向"程序化控制/生命周期"，debugfs 面向"调试"——remoteproc 三种都做，是为了让不同稳定性要求的接口各归其位，不把调试用的东西混进 ABI。


**ABI 稳定性**指的是：**内核暴露给用户态的“二进制接口契约”不能随便改**，否则已经编译好/写好的用户态程序、HAL、脚本会直接坏掉。

### 五、ABI
### 1. ABI 是什么

ABI = **Application Binary Interface（应用二进制接口）**。  
它不是源码层面的 API，而是“二进制层面怎么对接”的约定，包括：

- 系统调用号、参数寄存器/栈布局
- `ioctl` 的命令号、参数结构体布局、字段大小/对齐
- 设备节点路径、权限
- sysfs 属性文件名、读写格式、取值含义
- netlink 消息格式
- 结构体字段顺序、大小、位域布局

只要这些变了，用户态二进制不重新编译就可能出错。

### 2. “稳定 ABI”是什么意思

意思是：

> 这个接口一旦发布，就承诺保持兼容：属性名不改、格式不改、ioctl 号不改、语义不改；只能新增，不能随意删改已有部分。

比如 remoteproc sysfs：

```text
/sys/class/remoteproc/remoteprocN/state
```

读出来是 `offline/running/crashed`，写 `start/stop`。  
如果内核某天改成：

```text
/sys/class/remoteproc/remoteprocN/status
```

或者写 `begin/end`，那所有依赖它的 init.rc、vendor 脚本、测试脚本都会坏。  
所以 sysfs 属于**稳定 ABI**，不能随意改。

cdev 的 ioctl 号也是契约：

```c
ioctl(fd, RPROC_SET_SHUTDOWN_ON_RELEASE, ...)
```

如果内核把 `RPROC_SET_SHUTDOWN_ON_RELEASE` 的数值改了，已经编译好的 HAL 调过去就是错的命令，所以 ioctl 号也是稳定 ABI。

### 3. debugfs 为什么“不稳定”

debugfs 明确不承诺 ABI：

```text
/sys/kernel/debug/remoteproc/remoteprocN/resource_table
```

文件名、内容格式、甚至整个文件，内核版本一变就可能改/删。  
因为它只给开发者调试用，不视为正式用户态接口。生产环境甚至可能不挂载 debugfs。

### 4. 一句话类比

- **API 稳定**：源码能编译过。
- **ABI 稳定**：已经编译好的二进制不用重编也能继续跑。

内核说的“稳定 ABI”主要是后者：**用户态二进制依赖的接口契约不能破坏**。  
所以：

```text
sysfs    = 稳定 ABI：路径、属性名、文本格式是契约
cdev     = 稳定 ABI：设备节点、ioctl 号、参数结构是契约
debugfs  = 不稳定：调试接口，随时可改可删
```

这就是前面表格里“ABI 稳定性”那一行的含义。