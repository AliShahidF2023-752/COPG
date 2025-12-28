# Hardware/SoC Property Spoofing Guide

## Overview

COPG now supports comprehensive hardware detection spoofing to bypass advanced app checks. This includes spoofing system properties (`ro.*` values) and filesystem content (`/sys/*` and `/proc/*` paths).

## New Configuration Fields

### Hardware/SoC Properties

Add these fields to your device profiles in `COPG.json`:

```json
{
  "PACKAGES_YOUR_DEVICE_DEVICE": {
    "BRAND": "Samsung",
    "MODEL": "SM-S928B",
    "MANUFACTURER": "samsung",
    "DEVICE": "e3q",
    "PRODUCT": "e3qxxx",
    "FINGERPRINT": "samsung/e3qxxx/e3q:14/UP1A.231005.007/S928BXXS1AXXX:user/release-keys",
    
    // NEW: Hardware/SoC properties (all optional)
    "HARDWARE": "qcom",
    "HARDWARE_CHIPNAME": "SM8650",
    "SOC_MANUFACTURER": "Qualcomm",
    "SOC_MODEL": "SM8650",
    "ARCH": "arm64",
    "BOARD_PLATFORM": "kalama",
    "PRODUCT_BOARD": "kalama",
    "REVISION": "0",
    "BUILD_CHANGELIST": "12345678",
    "BUILD_FLAVOR": "e3qxxx-user",
    "CHIPNAME": "SM8650",
    "MEDIATEK_PLATFORM": "",
    "CPU_CORES": 8
  }
}
```

### Field Descriptions

| Field | System Property | Purpose | Example |
|-------|----------------|---------|---------|
| `HARDWARE` | `ro.hardware` | Hardware platform name | "qcom", "exynos", "mt6895" |
| `HARDWARE_CHIPNAME` | `ro.hardware.chipname` | SoC chip identifier | "SM8650", "Exynos2400" |
| `SOC_MANUFACTURER` | `ro.soc.manufacturer` | SoC maker | "Qualcomm", "Samsung", "MediaTek" |
| `SOC_MODEL` | `ro.soc.model` | SoC model number | "SM8650", "Exynos 2400" |
| `ARCH` | `ro.arch` | CPU architecture | "arm64", "armv8-a" |
| `BOARD_PLATFORM` | `ro.board.platform` | Board platform | "kalama", "pineapple" |
| `PRODUCT_BOARD` | `ro.product.board` | Product board name | "kalama", "mt6895" |
| `REVISION` | `ro.revision` | Hardware revision | "0", "1.0", "11" |
| `BUILD_CHANGELIST` | `ro.build.changelist` | Build changelist ID | "12345678" |
| `BUILD_FLAVOR` | `ro.build.flavor` | Build flavor | "e3qxxx-user", "PJZ110-user" |
| `CHIPNAME` | `ro.chipname` | Alternative chip name | "SM8650", "Dimensity9300" |
| `MEDIATEK_PLATFORM` | `ro.mediatek.platform` | MediaTek platform (if applicable) | "MT6895", "MT6989" |
| `CPU_CORES` | N/A (future use) | Number of CPU cores | 8, 12, 16 |

## How It Works

### System Properties
When a package with device spoofing is loaded, COPG sets the specified system properties using `resetprop`. Properties are only set if the corresponding field has a non-empty value in the configuration.

### File Spoofing
When CPU spoofing is active for a package, COPG also mounts spoofed files at:
- `/sys/devices/system/cpu/possible` → Reports "0-7" (8 cores)
- `/sys/devices/system/cpu/present` → Reports "0-7" (8 cores)
- `/sys/devices/system/cpu/online` → Reports "0-7" (8 cores)
- `/sys/devices/system/cpu/kernel_max` → Reports "7"
- `/proc/stat` → Spoofed CPU statistics
- `/proc/meminfo` → Spoofed memory information (8GB default)

These spoofed files are created during module installation in:
- `/data/adb/modules/COPG/sys_spoof/`
- `/data/adb/modules/COPG/proc_spoof/`

## Examples

### Qualcomm Snapdragon Device (OnePlus 13)
```json
"PACKAGES_ONEPLUS_13_DEVICE": {
  "BRAND": "OnePlus",
  "MODEL": "PJZ110",
  "HARDWARE": "qcom",
  "HARDWARE_CHIPNAME": "SM8750",
  "SOC_MANUFACTURER": "Qualcomm",
  "SOC_MODEL": "SM8750",
  "ARCH": "arm64",
  "BOARD_PLATFORM": "pineapple",
  "PRODUCT_BOARD": "pineapple",
  "CPU_CORES": 8
}
```

### Samsung Exynos Device
```json
"PACKAGES_SAMSUNG_S24_DEVICE": {
  "BRAND": "samsung",
  "MODEL": "SM-S921B",
  "HARDWARE": "exynos2400",
  "HARDWARE_CHIPNAME": "Exynos2400",
  "SOC_MANUFACTURER": "Samsung",
  "SOC_MODEL": "Exynos 2400",
  "ARCH": "arm64",
  "BOARD_PLATFORM": "s5e9945",
  "PRODUCT_BOARD": "s5e9945",
  "CPU_CORES": 10
}
```

### MediaTek Dimensity Device
```json
"PACKAGES_VIVO_X100_DEVICE": {
  "BRAND": "vivo",
  "MODEL": "V2309A",
  "HARDWARE": "mt6989",
  "HARDWARE_CHIPNAME": "MT6989",
  "SOC_MANUFACTURER": "MediaTek",
  "SOC_MODEL": "Dimensity 9300",
  "MEDIATEK_PLATFORM": "MT6989",
  "ARCH": "arm64",
  "BOARD_PLATFORM": "mt6989",
  "PRODUCT_BOARD": "mt6989",
  "CPU_CORES": 8
}
```

## Customizing Spoofed Files

You can customize the spoofed file content by editing files in:
- `/data/adb/modules/COPG/sys_spoof/`
- `/data/adb/modules/COPG/proc_spoof/`

After editing, reboot your device for changes to take effect.

### Example: Changing CPU Core Count
Edit `/data/adb/modules/COPG/sys_spoof/cpu_possible`:
```
0-11
```
This will report 12 CPU cores instead of 8.

Similarly update `cpu_present`, `cpu_online`, and `kernel_max` (set to 11 for 12 cores).

## Backward Compatibility

All new fields are **optional**. Existing device profiles without these fields will continue to work exactly as before. Only add these fields when you need to spoof hardware detection for specific apps.

## Troubleshooting

### Properties Not Being Set
1. Verify the field names are correct (case-sensitive)
2. Check that values are non-empty strings
3. Review logs: `adb logcat | grep COPGModule`

### File Mounts Not Working
1. Ensure CPU spoofing is enabled for the package (`:with_cpu` tag or in `cpu_only_packages`)
2. Check that spoof files exist: `ls -la /data/adb/modules/COPG/sys_spoof/`
3. Verify mounts: `mount | grep COPG`

### App Still Detecting Real Hardware
Some apps may use additional detection methods not covered by this implementation:
- `/proc/self/exe` and `/proc/self/auxv` (requires PLT hooking - not implemented)
- Hardware-specific files in `/sys/devices/`
- Direct hardware queries via kernel interfaces

## Technical Notes

- Mount-based approach is used for file spoofing (cleaner than PLT hooking)
- Mounts are per-namespace, only affecting the spoofed app
- Original system files remain unchanged
- No performance impact when not in use

## Detected By These Libraries

This spoofing addresses detection methods in:
- libliteavsdk (TikTok, live streaming apps)
- libonnxruntime (AI/ML apps)
- libncnn (Neural network apps)
- libopencv_java4 (Computer vision apps)
- libGCloudVoice (Gaming voice chat)
- libBugly_Native (Crash reporting, analytics)
