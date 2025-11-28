# Adreno 6xx Kernel Patcher: Enable Modern Drivers on Legacy Devices

**让旧款骁龙设备（如 OnePlus 6/SD845）运行最新的高通闭源 OpenCL/Vulkan 驱动。**

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

### 3. 静默降级 (Silent Downgrade)
作为安全网，如果驱动强行请求高优先级，内核不再返回错误（`-EINVAL`），而是悄悄将其降级为默认优先级。

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

    /* NEW CODE: Silent Downgrade */
    /* If requested priority is too high, force it to default (0) */
    if (prio >= priv->gpu->nr_rings) {
        prio = 0;
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

## 🧪 验证 (Verification)

编译并刷入新内核后，使用 `clinfo` 和 OpenCL 计算测试工具进行验证。

1.  **clinfo**: 应该能正确显示 Platform Name (QUALCOMM Snapdragon) 和 Device Name (Adreno 630)，且频率显示正常。
2.  **clpeak**: 应该能跑完所有测试，且 `Kernel launch latency` 不为 0（虽然分数可能依然虚高，这取决于驱动内部计时器）。
3.  **实际计算**: 运行简单的向量加法代码，结果应正确。

## ⚠️ 免责声明 (Disclaimer)

*   此修改涉及内核底层，刷机有风险，请务必备份数据。
*   我们不保证所有闭源驱动都能完美运行，这取决于驱动具体的编译版本和对硬件指令集的依赖。
*   强制降级优先级可能会轻微影响系统在高负载下的多任务 UI 响应速度（但在游戏/计算独占场景下无影响）。

---

**Credits:**
Research & Debugging by [nanasemai] & DeepSeek AI.
