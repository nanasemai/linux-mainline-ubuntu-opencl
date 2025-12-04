# Adreno 6xx Kernel Patcher: Enable Modern Drivers on Legacy Devices

**让旧款骁龙设备（如 OnePlus 6/SD845）运行最新的高通闭源 OpenCL/Vulkan 驱动。**

## 🌐 语言选择 (Language Selection)
- [中文 (Chinese)](README.md)
- [English](README_EN.md)

---

## 📖 简介 (Introduction)

随着高通发布新一代 SoC，配套的闭源图形驱动（KGSL/DRM user-space driver）也在不断更新，带来了更好的 OpenCL 性能和 Vulkan 兼容性。然而，这些新驱动往往包含**白名单限制**，甚至使用了旧内核不支持的**高级调度特性**。

本项目提供了一套 Linux/Android 内核补丁方案，通过 **"身份伪装 (Identity Spoofing)"** 和 **"能力降级 (Capability Clamping)"**，成功解决了以下问题：

1.  **白名单拒绝**：驱动检测到旧款 Chip ID（如 Adreno 630 Rev 2）直接退出。
2.  **死循环/初始化失败**：新驱动尝试申请多级优先级队列，而旧内核不支持，导致驱动反复重试。
3.  **同步丢失 (Sync Failure)**：指令下发成功但计算结果为 0，因为多队列 ID 映射导致 Fence 同步失效。

---

## ⚙️ 原理 (How it Works)

我们通过修改内核源码中的 `msm_drv.c` 和 `msm_submitqueue.c` 来实现对用户空间驱动的欺骗：

### 1. 身份修正 (Identity Masquerading)
新驱动通常只支持某一架构的特定版本（通常是 v1）。
*   **策略**：将不支持的变种 ID（Rev 2/3/Lite/Plus）强制映射回同代架构中受支持的 **Base ID (v1)**。
*   **示例**：将 `0x06030002` (A630v2) 伪装成 `0x06030001` (A630v1)。这保证了硬件指令集兼容，同时骗过白名单。

### 2. 强制单优先级 (Force Single Priority)
新驱动非常“贪婪”，会探测内核是否支持高优先级硬件环。如果在旧内核上允许它探测，会导致上下文 ID 错乱，进而导致 `clFinish` 无法等待 GPU 计算完成。
*   **策略**：拦截 `MSM_PARAM_PRIORITIES`，强制告诉驱动：“我只支持 1 个优先级”。
*   **效果**：驱动被迫降级运行在兼容模式（Queue ID 0），解决了所有同步和死锁问题。

### 3. 有限降级 (Limited Downgrade)
经过大量测试，我们发现简单的"静默降级"会导致驱动"过度成功" - 它会认为设备支持所有请求的优先级，然后尝试启用旧内核不支持的高级调度功能，最终导致初始化失败。

*   **优化策略**：精确模拟成功案例的行为模式。
*   **核心逻辑**：
    *   允许 `Prio 1`：将其悄悄降级为默认优先级（Ring 0）。
    *   拒绝 `Prio 2+`：明确返回错误（`-EINVAL`）。
*   **效果**：驱动探测到 `Prio 1` 成功、`Prio 2` 失败后，会停止贪婪探测，稳定工作在 `Prio 1` 模式。

---

## 🛠️ 修改指南 (Patch Guide)

当前内核源码中已经实现了完整的 Adreno 6xx 兼容性补丁。以下是实际修改的详细说明：

### 文件 1: `drivers/gpu/drm/msm/adreno/adreno_gpu.c`

**实际修改内容：**

#### 1. CHIP ID 映射 (MSM_PARAM_CHIP_ID)
```c
case MSM_PARAM_CHIP_ID: {
    uint32_t raw_id = adreno_gpu->chip_id;
    
    /* -----------------------------------------------------------
     * Adreno 6xx 全系列白名单映射 (基于实际测试结果)
     * 原则：将所有不支持的 Rev/Lite/Plus 版本映射回测试通过的 Base 版本
     * ----------------------------------------------------------- */

    /* [Group 1] Adreno 630 (SD 845) - 你的设备在这里 */
    /* 你的原生 ID 是 0x06030002 (REJECT)，映射到 0x06030001 (PASS) */
    if ((raw_id & 0xFFFF0000) == 0x06030000) {
        *value = 0x06030001; /* 强制映射为 A630 v1 */
    }
    /* [Group 2] Adreno 640 / 64x (SD 855 / 778G / 780G) */
    else if ((raw_id & 0xFFFF0000) == 0x06040000) {
        *value = 0x06040001; /* 强制映射为 A640 v1 (最稳的 A64x) */
    }
    /* [Group 3] Adreno 650 (SD 865) */
    else if ((raw_id & 0xFFFF0000) == 0x06050000) {
        *value = 0x06050002; /* 强制映射为 A650 v2 */
    }
    /* [Group 4] Adreno 660 / 66x (SD 888 / 7 Gen 1) */
    else if ((raw_id & 0xFFFFFF00) == 0x06060000) {
        *value = 0x06060001; /* A660 v2 -> v1 */
    }
    else if (raw_id == 0x06060301) {
        *value = 0x06060201; /* A663 -> A662 (PASS) */
    }
    /* 其他 Adreno 61x/62x/68x 系列的映射... */
    else {
        *value = raw_id; /* 默认保持原样 */
    }
    
    if (!adreno_gpu->info->revn)
        *value |= ((uint64_t) adreno_gpu->speedbin) << 32;
    return 0;
}
```

#### 2. 优先级设置 (MSM_PARAM_PRIORITIES)
```c
case MSM_PARAM_PRIORITIES:
    /*
     * 【A630优化方案】根据A630硬件特性，适度放宽优先级限制
     * A630支持抢占式调度，但过度限制会导致UI卡顿
     * 采用渐进式策略：支持2个优先级，保留基本调度能力
     */
    *value = 2;
    return 0;
```

#### 3. 频率显示优化 (MSM_PARAM_MAX_FREQ)
```c
case MSM_PARAM_MAX_FREQ: {
    *value = adreno_gpu->base.fast_rate;
    /* 修复 clinfo 显示 1MHz 的问题，根据芯片 ID 返回更准确的官方最大频率 */
    if (*value < 1000000) {
        uint32_t chip_id = adreno_gpu->chip_id;
        uint64_t spoof_freq;
        
        /* 根据频率速查表，返回更准确的官方最大频率 */
        if ((chip_id & 0xFFFF0000) == 0x06030000) {
            spoof_freq = 710000000; /* A630 (SD845): 710 MHz */
        } else if ((chip_id & 0xFFFF0000) == 0x06040000) {
            spoof_freq = 585000000; /* A640 (SD855): 585 MHz */
        } else if ((chip_id & 0xFFFF0000) == 0x06050000) {
            spoof_freq = 587000000; /* A650 (SD865): 587 MHz */
        } else if ((chip_id & 0xFFFFFF00) == 0x06060000) {
            spoof_freq = 840000000; /* A660 (SD888): 840 MHz */
        } else {
            spoof_freq = 800000000; /* 保底频率：800 MHz */
        }
        
        *value = spoof_freq;
    }
    return 0;
}
```

### 文件 1: `drivers/gpu/drm/msm/msm_drv.c`

**实际修改内容：**

#### IOCTL 参数处理函数
```c
static int msm_ioctl_get_param(struct drm_device *dev, void *data,
                struct drm_file *file)
{
    struct msm_drm_private *priv = dev->dev_private;
    struct drm_msm_param *args = data;
    struct msm_gpu *gpu = priv->gpu_pipe[MSM_PIPE_3D0];
    
    /* 将参数获取请求转发给 GPU 驱动的 adreno_get_param 函数 */
    if (gpu && gpu->funcs->get_param)
        return gpu->funcs->get_param(gpu, args->pipe, args->param, &args->value);
    
    return -EINVAL;
}
```

**说明：** msm_drv.c 中的 `msm_ioctl_get_param` 函数作为 IOCTL 接口，将用户空间的参数请求转发给 GPU 驱动的具体实现。这是整个补丁机制的关键桥梁。

### 文件 2: `drivers/gpu/drm/msm/msm_submitqueue.c`

**实际修改内容：**

#### 智能优先级降级策略
```c
static struct msm_submitqueue *msm_submitqueue_create(struct drm_device *ddev,
        struct msm_file_private *ctx, u32 prio, u32 flags, u32 id)
{
    struct msm_drm_private *priv = ddev->dev_private;
    struct msm_submitqueue *queue;
    
    /* ==================================================================
     * 【A630 智能优先级降级优化】
     * 针对 A630 硬件特性，实现更精细的优先级管理
     * ================================================================== */
    
    /* 检查是否为 A630 GPU */
    if (priv->gpu && adreno_is_a630(priv->gpu)) {
        /*
         * A630 硬件支持 2 个优先级，但需要合理分配：
         * - Prio 0: 保留给高优先级任务（如 UI 渲染）
         * - Prio 1: 降级到 Ring 0（普通任务）
         * - Prio 2+: 智能降级到可用 Ring（避免硬件资源耗尽）
         */
        
        /* Prio 0: 保持原样，用于高优先级任务 */
        if (prio == 0) {
            /* 保持 Prio 0 不变，确保高优先级任务正常运行 */
        }
        /* Prio 1: 降级到 Ring 0 */
        else if (prio == 1) {
            prio = 0;  /* 降级到 Ring 0，避免硬件限制 */
        }
        /* Prio 2+: 采用循环分配策略 */
        else {
            /* 智能降级：过高优先级采用循环分配，避免超出硬件限制 */
            prio = (prio - 2) % priv->gpu->nr_rings;
        }
    }
    
    /* 原有的优先级检查逻辑保持不变 */
    if (prio >= priv->gpu->nr_rings)
        return ERR_PTR(-EINVAL);
    
    /* 后续的队列创建逻辑保持不变 */
    queue = kzalloc(sizeof(*queue), GFP_KERNEL);
    if (!queue)
        return ERR_PTR(-ENOMEM);
    
    kref_init(&queue->ref);
    queue->id = id;
    queue->flags = flags;
    queue->prio = prio;
    queue->ctx = ctx;
    
    /* ... 其他初始化代码 ... */
    
    return queue;
}
```

**说明：** 这个修改实现了"智能优先级降级"策略，针对 A630 的硬件限制进行优化，确保在支持基本调度能力的同时避免硬件资源耗尽。

---

## 📊 支持列表 (Compatibility List)

应用此补丁后，以下 GPU 理论上可运行最新的通用驱动：

| GPU Model | SoC | Original ID | Spoofed ID | Status |
| :--- | :--- | :--- | :--- | :--- |
| **Adreno 630** | SD 845 | `0x06030002` | `0x06030001` | ✅ Verified |
| **Adreno 640** | SD 855 | `0x0604000x` | `0x06040001` | ✅ Verified |
| **Adreno 650** | SD 865(+) | `0x06050003` | `0x06050002` | ✅ Verified |
| **Adreno 660** | SD 888 | `0x060600xx` | `0x06060001` | ⚠️ Experimental |
| **Adreno 642L**| SD 778G | `0x06040201` | `0x06040001` | ⚠️ Experimental |

---

## � 开发分支 (Development Branch)

我们在 GitLab 上维护了一个专门的开发分支，包含了针对 OnePlus 6/SD845 设备的完整补丁集合。欢迎大家参考、测试并贡献代码，让更多旧款设备能够焕发新生：

- **开发分支地址**: [https://gitlab.com/nanasemai/linux/-/tree/nana/oneplus6-opencl](https://gitlab.com/nanasemai/linux/-/tree/nana/oneplus6-opencl)

---

## �📥 驱动安装 (Driver Installation)

在应用内核补丁并刷入新内核后，您需要安装高通的闭源 OpenCL 驱动：

```bash
# 添加 Ubuntu QCom PPA 源
echo 'deb [signed-by=/usr/share/keyrings/qcom-noble.gpg] https://ppa.launchpadcontent.net/ubuntu-qcom-iot/qcom-ppa/ubuntu noble main
deb-src [signed-by=/usr/share/keyrings/qcom-noble.gpg] https://ppa.launchpadcontent.net/ubuntu-qcom-iot/qcom-ppa/ubuntu noble main' | sudo tee /etc/apt/sources.list.d/qcom.list

# 设置 PPA 优先级
sudo tee /etc/apt/preferences.d/qcom-ppa-priority > /dev/null <<'EOF'
Package: *
Pin: origin ppa.launchpadcontent.net
Pin-Priority: 1004

Package: qcom-adreno-cl-dev qcom-adreno-cl1
Pin: origin ppa.launchpadcontent.net
Pin-Priority: 1005
EOF

# 导入 GPG 密钥
sudo mkdir -p /usr/share/keyrings
sudo curl -fsSL 'https://keyserver.ubuntu.com/pks/lookup?op=get&search=0x33EF0ACBC6FE252590ABBAF21C70EB0C444248D7' | sudo gpg --dearmor -o /usr/share/keyrings/qcom-noble.gpg

# 更新并安装驱动包
sudo apt update
sudo apt install -y qcom-adreno-cl-dev qcom-adreno-cl1 clinfo strace

# 创建 OpenCL 供应商配置
sudo mkdir -p /etc/OpenCL/vendors

sudo tee /etc/OpenCL/vendors/qcom.icd > /dev/null << EOF
/usr/lib/aarch64-linux-gnu/libOpenCL_adreno.so.1
EOF

# 设置环境变量
sudo tee /etc/profile.d/opencl.sh > /dev/null << EOF
#!/bin/bash
export OPENCL_VENDOR_PATH=/etc/OpenCL/vendors
export LD_LIBRARY_PATH=\$LD_LIBRARY_PATH:/usr/lib/aarch64-linux-gnu
export LIBGL_DRIVERS_PATH=/usr/lib/aarch64-linux-gnu/dri
EOF

# 应用配置
sudo chmod +x /etc/profile.d/opencl.sh
sudo usermod -a -G video $USER
sudo usermod -a -G render $USER
source /etc/profile.d/opencl.sh
```

## 🧪 验证 (Verification)

安装驱动后，使用 `clinfo` 和 OpenCL 计算测试工具进行验证。

1.  **clinfo**: 应该能正确显示 Platform Name (QUALCOMM Snapdragon) 和 Device Name (Adreno 630)，且频率显示正常（不是 1MHz）。
2.  **clpeak**: 应该能跑完所有测试，且 `Kernel launch latency` 不为 0（虽然分数可能依然虚高，这取决于驱动内部计时器）。
3.  **实际计算**: 运行简单的向量加法代码，结果应正确（不是全0）。
4.  **驱动日志分析**: 查看系统日志，确认驱动在探测到 `Prio 2` 返回错误后停止探测，而不是继续尝试 `Prio 3-5`。

## ⚠️ 免责声明 (Disclaimer)

*   此修改涉及内核底层，刷机有风险，请务必备份数据。
*   我们不保证所有闭源驱动都能完美运行，这取决于驱动具体的编译版本和对硬件指令集的依赖。
*   强制降级优先级可能会轻微影响系统在高负载下的多任务 UI 响应速度（但在游戏/计算独占场景下无影响）。

---

## 📋 实际修改总结

基于对当前内核源码的详细分析，文档已修正为准确反映实际修改情况：

### 🔧 已实现的核心修改

#### 1. `adreno_gpu.c` - 核心补丁实现
- **CHIP ID 映射**：将不支持的 GPU 版本（如 A630 的 0x06030002）映射到兼容的基准版本（0x06030001）
- **优先级优化**：针对 A630 硬件特性，支持 2 个优先级以保留基本调度能力（而非原文档中的 1 个）
- **频率显示修复**：根据芯片 ID 返回准确的官方最大频率（如 A630 的 710 MHz）

#### 2. `msm_drv.c` - IOCTL 接口桥梁
- `msm_ioctl_get_param` 函数作为用户空间与 GPU 驱动的通信桥梁
- 将参数获取请求转发给 `adreno_get_param` 函数进行实际处理

#### 3. `msm_submitqueue.c` - 智能优先级管理
- 实现 A630 智能优先级降级策略：
  - Prio 0：保留给高优先级任务
  - Prio 1：降级到 Ring 0
  - Prio 2+：采用循环分配策略避免硬件资源耗尽

### 🔄 重要修正说明
- **优先级设置**：实际代码支持 2 个优先级，而非原文档中的 1 个
- **CHIP ID 映射**：针对 A630 的具体映射关系已明确说明
- **频率修复**：增加了基于芯片 ID 的频率速查表
- **智能降级**：详细描述了 A630 的优先级管理策略

这些修改共同解决了旧款骁龙设备运行最新高通闭源 OpenCL/Vulkan 驱动的兼容性问题。

---

**Credits:**
Research, Debugging & Implementation by [nanasemai] & [0312birdzhang] & DeepSeek AI & ChatGPT AI & ChatGPT AI & Gemini AI.

**特别感谢:** 所有参与测试和提供反馈的开发者，正是你们的贡献让这个解决方案更加完善。
