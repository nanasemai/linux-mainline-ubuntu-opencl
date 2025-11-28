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

你需要修改手机/设备内核源码中的两个文件。

### 文件 1: `drivers/gpu/drm/msm/adreno/adreno_gpu.c` (或 `msm_drv.c`)

找到 `adreno_get_param` 函数（部分旧内核为 `msm_ioctl_get_param`），修改以下 `case` 逻辑：

```c
/* ==================================================================
 *  Replace logic in adreno_get_param / msm_ioctl_get_param
 * ================================================================== */

case MSM_PARAM_CHIP_ID: {
    uint32_t raw_id = adreno_gpu->chip_id; // Or gpu->chip_id
    
    /* --- Adreno 6xx Compatibility Map --- */
    
    /* Adreno 630 (SD845): Map Rev 2/3 to v1 */
    if ((raw_id & 0xFFFF0000) == 0x06030000) {
        *value = 0x06030001; 
    }
    /* Adreno 640 (SD855): Map all variants to v1 */
    else if ((raw_id & 0xFFFF0000) == 0x06040000) {
        *value = 0x06040001; 
    }
    /* Adreno 650 (SD865): Map 865+ to v2 */
    else if ((raw_id & 0xFFFF0000) == 0x06050000) {
        *value = 0x06050002; 
    }
    /* Adreno 660 (SD888): Map v2 to v1 */
    else if ((raw_id & 0xFFFFFF00) == 0x06060000) {
        *value = 0x06060001; 
    }
    /* Fallback: Keep original ID */
    else {
        *value = raw_id;
    }
    
    /* Append speedbin if necessary */
    if (!adreno_gpu->info->revn)
        *value |= ((uint64_t) adreno_gpu->speedbin) << 32;
    return 0;
}

case MSM_PARAM_PRIORITIES:
    /* 
     * CRITICAL FIX: Force Single Priority.
     * Prevents driver form requesting multiple hardware rings, 
     * solving the "submit success but result is 0" sync issue.
     */
    *value = 1;
    return 0;

case MSM_PARAM_MAX_FREQ:
    *value = adreno_gpu->base.fast_rate;
    /* Cosmetic fix for clinfo showing 1MHz */
    if (*value < 1000000) *value = 710000000; 
    return 0;
```

### 文件 2: `drivers/gpu/drm/msm/msm_submitqueue.c`

找到 `msm_submitqueue_create` 函数，修改优先级检查逻辑：

```c
/* ==================================================================
 *  Modify msm_submitqueue_create
 * ================================================================== */

int msm_submitqueue_create(...) {
    /* ... code ... */

    /* ORIGINAL CODE:
    if (prio >= gpu->nr_rings)
        return -EINVAL;
    */

    /* NEW CODE: Limited Downgrade (模仿成功案例的行为) */
    if (prio >= priv->gpu->nr_rings) {
        /* 
         * 策略：
         * 1. 允许 Prio 1（映射到 Ring 0），因为驱动似乎强制需要至少一个非0优先级。
         * 2. 拒绝 Prio 2 及以上。这会迫使驱动停止贪婪探测，接受当前的配置。
         */
        if (prio == 1) {
            prio = 0; /* 允许 Prio 1，悄悄降级到 0 */
        } else {
            return -EINVAL; /* 拒绝 Prio 2, 3, 4, 5 */
        }
    }

    /* ... continue execution ... */
}
```

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

## 📥 驱动安装 (Driver Installation)

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

**Credits:**
Research, Debugging & Implementation by [nanasemai] & DeepSeek AI.

**特别感谢:** 所有参与测试和提供反馈的开发者，正是你们的贡献让这个解决方案更加完善。
