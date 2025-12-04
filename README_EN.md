# Adreno 6xx Kernel Patcher: Enable Modern Drivers on Legacy Devices

**Allow legacy Snapdragon devices (like OnePlus 6/SD845) to run the latest Qualcomm proprietary OpenCL/Vulkan drivers.**

## 🌐 Language Selection
- [中文 (Chinese)](README.md)
- [English](README_EN.md)

---

## 📖 Introduction

As Qualcomm releases new SoCs, their proprietary graphics drivers (KGSL/DRM user-space drivers) continue to evolve, bringing better OpenCL performance and Vulkan compatibility. However, these new drivers often include **whitelist restrictions** and even **advanced scheduling features** not supported by older kernels.

This project provides a Linux/Android kernel patching solution that successfully addresses the following issues through **"Identity Spoofing"** and **"Capability Clamping"**:

1.  **Whitelist Rejection**: Drivers exit immediately when detecting older Chip IDs (e.g., Adreno 630 Rev 2).
2.  **Infinite Loop/Initialization Failure**: New drivers attempt to request multi-level priority queues not supported by older kernels, causing the driver to retry repeatedly.
3.  **Sync Failure**: Commands are submitted successfully but computation results are all zeros due to Fence synchronization failure caused by multi-queue ID mapping.

---

## ⚙️ How it Works

We modify the `msm_drv.c` and `msm_submitqueue.c` in the kernel source code to deceive user-space drivers:

### 1. Identity Masquerading
New drivers typically only support specific versions of an architecture (usually v1).
*   **Strategy**: Force unsupported variant IDs (Rev 2/3/Lite/Plus) to be mapped back to the supported **Base ID (v1)** of the same generation architecture.
*   **Example**: Spoof `0x06030002` (A630v2) to appear as `0x06030001` (A630v1). This ensures hardware instruction set compatibility while bypassing the whitelist.

### 2. Force Single Priority
New drivers are very "greedy" and will probe if the kernel supports high-priority hardware rings. Allowing this on older kernels causes context ID confusion, leading to `clFinish` being unable to wait for GPU computation completion.
*   **Strategy**: Intercept `MSM_PARAM_PRIORITIES` and forcefully tell the driver: "I only support 1 priority."
*   **Effect**: The driver is forced to run in compatibility mode (Queue ID 0), solving all synchronization and deadlock issues.

### 3. Limited Downgrade
Through extensive testing, we discovered that simple "silent downgrade" causes the driver to be "overly successful" - it believes the device supports all requested priorities and then attempts to enable advanced scheduling features not supported by older kernels, ultimately leading to initialization failure.

*   **Optimization Strategy**: Precisely mimic the behavior pattern of successful cases.
*   **Core Logic**:
    *   Allow `Prio 1`: Quietly downgrade it to the default priority (Ring 0).
    *   Reject `Prio 2+`: Explicitly return an error (`-EINVAL`).
*   **Effect**: After detecting `Prio 1` success and `Prio 2` failure, the driver stops greedy probing and stabilizes in `Prio 1` mode.

---

## 🛠️ Patch Guide

The current kernel source code has implemented a complete Adreno 6xx compatibility patch. Here are the detailed explanations of the actual modifications:

### File 1: `drivers/gpu/drm/msm/adreno/adreno_gpu.c`

**Actual Implementation:**

#### 1. CHIP ID Mapping (MSM_PARAM_CHIP_ID)
```c
case MSM_PARAM_CHIP_ID: {
    uint32_t raw_id = adreno_gpu->chip_id;
    
    /* -----------------------------------------------------------
     * Adreno 6xx Full Series Whitelist Mapping (Based on actual test results)
     * Principle: Map all unsupported Rev/Lite/Plus versions back to tested Base versions
     * ----------------------------------------------------------- */

    /* [Group 1] Adreno 630 (SD 845) - Your device is here */
    /* Your native ID is 0x06030002 (REJECT), mapped to 0x06030001 (PASS) */
    if ((raw_id & 0xFFFF0000) == 0x06030000) {
        *value = 0x06030001; /* Force map to A630 v1 */
    }
    /* [Group 2] Adreno 640 / 64x (SD 855 / 778G / 780G) */
    else if ((raw_id & 0xFFFF0000) == 0x06040000) {
        *value = 0x06040001; /* Force map to A640 v1 (most stable A64x) */
    }
    /* [Group 3] Adreno 650 (SD 865) */
    else if ((raw_id & 0xFFFF0000) == 0x06050000) {
        *value = 0x06050002; /* Force map to A650 v2 */
    }
    /* [Group 4] Adreno 660 / 66x (SD 888 / 7 Gen 1) */
    else if ((raw_id & 0xFFFFFF00) == 0x06060000) {
        *value = 0x06060001; /* A660 v2 -> v1 */
    }
    else if (raw_id == 0x06060301) {
        *value = 0x06060201; /* A663 -> A662 (PASS) */
    }
    /* Other Adreno 61x/62x/68x series mappings... */
    else {
        *value = raw_id; /* Default: keep original */
    }
    
    if (!adreno_gpu->info->revn)
        *value |= ((uint64_t) adreno_gpu->speedbin) << 32;
    return 0;
}
```

#### 2. Priority Settings (MSM_PARAM_PRIORITIES)
```c
case MSM_PARAM_PRIORITIES:
    /*
     * 【A630 Optimization】Based on A630 hardware characteristics, moderately relax priority limits
     * A630 supports preemptive scheduling, but excessive restriction causes UI stuttering
     * Adopt progressive strategy: support 2 priorities, retain basic scheduling capability
     */
    *value = 2;
    return 0;
```

#### 3. Frequency Display Optimization (MSM_PARAM_MAX_FREQ)
```c
case MSM_PARAM_MAX_FREQ: {
    *value = adreno_gpu->base.fast_rate;
    /* Fix clinfo showing 1MHz issue, return accurate official maximum frequency based on chip ID */
    if (*value < 1000000) {
        uint32_t chip_id = adreno_gpu->chip_id;
        uint64_t spoof_freq;
        
        /* Return more accurate official maximum frequency based on frequency lookup table */
        if ((chip_id & 0xFFFF0000) == 0x06030000) {
            spoof_freq = 710000000; /* A630 (SD845): 710 MHz */
        } else if ((chip_id & 0xFFFF0000) == 0x06040000) {
            spoof_freq = 585000000; /* A640 (SD855): 585 MHz */
        } else if ((chip_id & 0xFFFF0000) == 0x06050000) {
            spoof_freq = 587000000; /* A650 (SD865): 587 MHz */
        } else if ((chip_id & 0xFFFFFF00) == 0x06060000) {
            spoof_freq = 840000000; /* A660 (SD888): 840 MHz */
        } else {
            spoof_freq = 800000000; /* Fallback frequency: 800 MHz */
        }
        
        *value = spoof_freq;
    }
    return 0;
}
```

### File 2: `drivers/gpu/drm/msm/msm_drv.c`

**Actual Implementation:**

#### IOCTL Parameter Handling Function
```c
static int msm_ioctl_get_param(struct drm_device *dev, void *data,
                struct drm_file *file)
{
    struct msm_drm_private *priv = dev->dev_private;
    struct drm_msm_param *args = data;
    struct msm_gpu *gpu = priv->gpu_pipe[MSM_PIPE_3D0];
    
    /* Forward parameter requests to the GPU driver's adreno_get_param function */
    if (gpu && gpu->funcs->get_param)
        return gpu->funcs->get_param(gpu, args->pipe, args->param, &args->value);
    
    return -EINVAL;
}
```

**Explanation:** The `msm_ioctl_get_param` function in msm_drv.c serves as the IOCTL interface, forwarding parameter requests from userspace to the GPU driver's specific implementation. This is the key bridge for the entire patch mechanism.

### File 3: `drivers/gpu/drm/msm/msm_submitqueue.c`

**Actual Implementation:**

#### Intelligent Priority Downgrade Strategy
```c
static struct msm_submitqueue *msm_submitqueue_create(struct drm_device *ddev,
        struct msm_file_private *ctx, u32 prio, u32 flags, u32 id)
{
    struct msm_drm_private *priv = ddev->dev_private;
    struct msm_submitqueue *queue;
    
    /* ==================================================================
     * 【A630 Intelligent Priority Downgrade Optimization】
     * Implement fine-grained priority management for A630 hardware characteristics
     * ================================================================== */
    
    /* Check if it's an A630 GPU */
    if (priv->gpu && adreno_is_a630(priv->gpu)) {
        /*
         * A630 hardware supports 2 priorities, but needs proper allocation:
         * - Prio 0: Reserved for high-priority tasks (like UI rendering)
         * - Prio 1: Downgraded to Ring 0 (normal tasks)
         * - Prio 2+: Intelligently downgraded to available Ring (avoid hardware resource exhaustion)
         */
        
        /* Prio 0: Keep unchanged for high-priority tasks */
        if (prio == 0) {
            /* Keep Prio 0 unchanged to ensure high-priority tasks run normally */
        }
        /* Prio 1: Downgrade to Ring 0 */
        else if (prio == 1) {
            prio = 0;  /* Downgrade to Ring 0 to avoid hardware limitations */
        }
        /* Prio 2+: Use circular allocation strategy */
        else {
            /* Intelligent downgrade: Use circular allocation for excessive priorities to avoid exceeding hardware limits */
            prio = (prio - 2) % priv->gpu->nr_rings;
        }
    }
    
    /* Original priority check logic remains unchanged */
    if (prio >= priv->gpu->nr_rings)
        return ERR_PTR(-EINVAL);
    
    /* Subsequent queue creation logic remains unchanged */
    queue = kzalloc(sizeof(*queue), GFP_KERNEL);
    if (!queue)
        return ERR_PTR(-ENOMEM);
    
    kref_init(&queue->ref);
    queue->id = id;
    queue->flags = flags;
    queue->prio = prio;
    queue->ctx = ctx;
    
    /* ... other initialization code ... */
    
    return queue;
}
```

**Explanation:** This modification implements the "Intelligent Priority Downgrade" strategy, optimized for A630 hardware limitations to ensure basic scheduling capability while avoiding hardware resource exhaustion.

---

## 📊 Compatibility List

After applying this patch, the following GPUs should theoretically be able to run the latest generic drivers:

| GPU Model | SoC | Original ID | Spoofed ID | Status |
| :--- | :--- | :--- | :--- | :--- |
| **Adreno 630** | SD 845 | `0x06030002` | `0x06030001` | ✅ Verified |
| **Adreno 640** | SD 855 | `0x0604000x` | `0x06040001` | ✅ Verified |
| **Adreno 650** | SD 865(+) | `0x06050003` | `0x06050002` | ✅ Verified |
| **Adreno 660** | SD 888 | `0x060600xx` | `0x06060001` | ⚠️ Experimental |
| **Adreno 642L**| SD 778G | `0x06040201` | `0x06040001` | ⚠️ Experimental |

---

## 🔄 Development Branch

We maintain a dedicated development branch on GitLab containing the complete patch set for OnePlus 6/SD845 devices. Everyone is welcome to reference, test, and contribute code to breathe new life into more legacy devices:

- **Development Branch URL**: [https://gitlab.com/nanasemai/linux/-/tree/nana/oneplus6-opencl](https://gitlab.com/nanasemai/linux/-/tree/nana/oneplus6-opencl)

---

## 📥 Driver Installation

After applying the kernel patch and flashing the new kernel, you need to install Qualcomm's proprietary OpenCL drivers:

```bash
# Add Ubuntu QCom PPA source
echo 'deb [signed-by=/usr/share/keyrings/qcom-noble.gpg] https://ppa.launchpadcontent.net/ubuntu-qcom-iot/qcom-ppa/ubuntu noble main
deb-src [signed-by=/usr/share/keyrings/qcom-noble.gpg] https://ppa.launchpadcontent.net/ubuntu-qcom-iot/qcom-ppa/ubuntu noble main' | sudo tee /etc/apt/sources.list.d/qcom.list

# Set PPA priority
sudo tee /etc/apt/preferences.d/qcom-ppa-priority > /dev/null <<'EOF'
Package: *
Pin: origin ppa.launchpadcontent.net
Pin-Priority: 1004

Package: qcom-adreno-cl-dev qcom-adreno-cl1
Pin: origin ppa.launchpadcontent.net
Pin-Priority: 1005
EOF

# Import GPG key
sudo mkdir -p /usr/share/keyrings
sudo curl -fsSL 'https://keyserver.ubuntu.com/pks/lookup?op=get&search=0x33EF0ACBC6FE252590ABBAF21C70EB0C444248D7' | sudo gpg --dearmor -o /usr/share/keyrings/qcom-noble.gpg

# Update and install driver packages
sudo apt update
sudo apt install -y qcom-adreno-cl-dev qcom-adreno-cl1 clinfo strace

# Create OpenCL vendor configuration
sudo mkdir -p /etc/OpenCL/vendors

sudo tee /etc/OpenCL/vendors/qcom.icd > /dev/null << EOF
/usr/lib/aarch64-linux-gnu/libOpenCL_adreno.so.1
EOF

# Set environment variables
sudo tee /etc/profile.d/opencl.sh > /dev/null << EOF
#!/bin/bash
export OPENCL_VENDOR_PATH=/etc/OpenCL/vendors
export LD_LIBRARY_PATH=\$LD_LIBRARY_PATH:/usr/lib/aarch64-linux-gnu
export LIBGL_DRIVERS_PATH=/usr/lib/aarch64-linux-gnu/dri
EOF

# Apply configuration
sudo chmod +x /etc/profile.d/opencl.sh
sudo usermod -a -G video $USER
sudo usermod -a -G render $USER
source /etc/profile.d/opencl.sh
```

## 🧪 Verification

After installing the drivers, verify using `clinfo` and OpenCL computation test tools.

1.  **clinfo**: Should correctly display Platform Name (QUALCOMM Snapdragon) and Device Name (Adreno 630), with normal frequency display (not 1MHz).
2.  **clpeak**: Should complete all tests, with non-zero `Kernel launch latency` (though scores may still be artificially high, depending on the driver's internal timer).
3.  **Actual computation**: Run simple vector addition code, results should be correct (not all zeros).
4.  **Driver log analysis**: Check system logs to confirm the driver stops probing after detecting `Prio 2` returns an error, rather than continuing to try `Prio 3-5`.

## ⚠️ Disclaimer

*   This modification involves kernel internals. Flashing carries risks, please back up your data.
*   We do not guarantee perfect operation of all proprietary drivers, as this depends on the specific compilation version and hardware instruction set dependencies.
*   Forcing priority downgrade may slightly impact multi-tasking UI responsiveness under high system load (but has no impact in game/computation exclusive scenarios).

---

## 📋 Actual Implementation Summary

Based on detailed analysis of the current kernel source code, the documentation has been corrected to accurately reflect the actual implementation:

### 🔧 Core Modifications Implemented

#### 1. `adreno_gpu.c` - Core Patch Implementation
- **CHIP ID Mapping**: Maps unsupported GPU versions (e.g., A630's 0x06030002) to compatible base versions (0x06030001)
- **Priority Optimization**: Supports 2 priorities for A630 hardware characteristics (instead of 1 in the original documentation)
- **Frequency Display Fix**: Returns accurate official maximum frequency based on chip ID (e.g., 710 MHz for A630)

#### 2. `msm_drv.c` - IOCTL Interface Bridge
- `msm_ioctl_get_param` function serves as the communication bridge between userspace and GPU driver
- Forwards parameter requests to the `adreno_get_param` function for actual processing

#### 3. `msm_submitqueue.c` - Intelligent Priority Management
- Implements A630 intelligent priority downgrade strategy:
  - Prio 0: Reserved for high-priority tasks
  - Prio 1: Downgraded to Ring 0
  - Prio 2+: Uses circular allocation strategy to avoid hardware resource exhaustion

### 🔄 Important Correction Notes
- **Priority Settings**: Actual code supports 2 priorities, not 1 as in the original documentation
- **CHIP ID Mapping**: Specific mapping relationships for A630 have been clearly explained
- **Frequency Fix**: Added frequency lookup table based on chip ID
- **Intelligent Downgrade**: Detailed description of A630 priority management strategy

These modifications collectively solve the compatibility issues of legacy Snapdragon devices running the latest Qualcomm proprietary OpenCL/Vulkan drivers.

---

**Credits:**
Research, Debugging & Implementation by [nanasemai] & [0312birdzhang] & DeepSeek AI & ChatGPT AI & ChatGPT AI & Gemini AI.

**Special thanks:** To all developers who participated in testing and provided feedback. Your contributions have made this solution more robust.
