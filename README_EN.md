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

You need to modify two files in your phone/device's kernel source code.

### File 1: `drivers/gpu/drm/msm/adreno/adreno_gpu.c` (or `msm_drv.c`)

Find the `adreno_get_param` function (in some older kernels: `msm_ioctl_get_param`) and modify the following `case` logic:

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

### File 2: `drivers/gpu/drm/msm/msm_submitqueue.c`

Find the `msm_submitqueue_create` function and modify the priority check logic:

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

    /* NEW CODE: Limited Downgrade (Mimicking successful cases) */
    if (prio >= priv->gpu->nr_rings) {
        /* 
         * Strategy:
         * 1. Allow Prio 1 (mapped to Ring 0), as the driver seems to forcefully require at least one non-zero priority.
         * 2. Reject Prio 2 and above. This forces the driver to stop greedy probing and accept the current configuration.
         */
        if (prio == 1) {
            prio = 0; /* Allow Prio 1, quietly downgrade to 0 */
        } else {
            return -EINVAL; /* Reject Prio 2, 3, 4, 5 */
        }
    }

    /* ... continue execution ... */
}
```

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

**Credits:**
Research, Debugging & Implementation by [nanasemai] & DeepSeek AI.

**Special thanks:** To all developers who participated in testing and provided feedback. Your contributions have made this solution more robust.
