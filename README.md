# tick-vulkan

Vulkan binding layer for the [Tick programming language](https://github.com/Rounak-Paul/tick).

## Files

| File | Purpose |
|------|---------|
| `vulkan.tick` | Binding layer — structs, externs, constants, helpers |
| `sandbox.tick` | Demo — device enumeration, logical device, capabilities |

---

## Build

### Prerequisites

1. **Tick compiler** built from source:
   ```sh
   git clone https://github.com/Rounak-Paul/tick
   cd tick
   cmake -B build && cmake --build build
   # add ./build/tick to your PATH or use the absolute path
   ```

2. **Vulkan SDK / MoltenVK** (macOS):
   ```sh
   # Download from https://vulkan.lunarg.com/sdk/home
   source ~/VulkanSDK/<version>/setup-env.sh
   ```
   Or via Homebrew:
   ```sh
   brew install molten-vk vulkan-headers
   ```

### Compile

**macOS / MoltenVK:**
```sh
tick sandbox.tick -DMACOS -o sandbox
```

**Linux / Windows:**
```sh
tick sandbox.tick -o sandbox
```

The `-DMACOS` flag activates `@if(MACOS)` blocks in both files, enabling
the MoltenVK portability extensions and flags.  Without it, the extension
arrays are empty and the portability flag is 0 — appropriate for Desktop
Linux/Windows Vulkan drivers.

The Tick pipeline compiles `sandbox.tick` → intermediary C → GCC binary.
`link "-lvulkan"` and `link "-Wl,-rpath,/usr/local/lib"` are embedded in
`vulkan.tick`, so no extra linker flags are needed on the command line.

### Run

```sh
./sandbox
```

Expected output (abbreviated):
```
╔══════════════════════════════════╗
║   tick-vulkan sandbox             ║
╚══════════════════════════════════╝

[OK] VkInstance created.

── Enumerating physical devices ──────────────────────────
Found 1 device(s).

Device [0]
───────────────────────────────────────────────────────
  Name        : Apple M3 Pro
  Type        : Integrated GPU
  API version : 1.2.323
  ...
  Memory heaps (1):
    Heap [0]: 18 GiB 0 MiB  [DEVICE_LOCAL]
  Memory types (3):
    Type [0] heap=0: DEVICE_LOCAL
    ...
  Features (selection):
    geometryShader      : no
    tessellationShader  : yes
    samplerAnisotropy   : yes
    ...

── Selected device [0]: Apple M3 Pro
Graphics queue family index: 0

── Creating logical device ───────────────────────────────
[OK] VkDevice (logical device) created.
[OK] VkQueue (graphics) handle retrieved.

── Final capabilities summary ────────────────────────────
Device          : Apple M3 Pro
Type            : Integrated GPU
Vulkan API      : 1.2.323
Graphics family : 0
Queue priority  : 1.0 (max)

── Cleanup ───────────────────────────────────────────────
[OK] VkDevice destroyed.
[OK] VkInstance destroyed.

Done.
```

---

## Platform notes

### macOS / MoltenVK

Compile with `-DMACOS`.  Under the hood, `@if(MACOS)` blocks activate:

| Symbol | Value |
|--------|-------|
| `VK_EXT_PORTABILITY_ENUMERATION` | `"VK_KHR_portability_enumeration"` |
| `VK_EXT_GET_PHYS_DEV_PROPERTIES2` | `"VK_KHR_get_physical_device_properties2"` |
| `VK_DEV_EXT_PORTABILITY_SUBSET` | `"VK_KHR_portability_subset"` |
| `PLATFORM_INSTANCE_FLAGS` | `1` (VK_INSTANCE_CREATE_ENUMERATE_PORTABILITY_BIT_KHR) |
| `PLATFORM_INSTANCE_EXT_COUNT` | `2` |
| `PLATFORM_DEVICE_EXT_COUNT` | `1` |

### Linux / Windows

Compile without `-DMACOS`.  The portability constants resolve to empty
strings / zero counts, and the extension arrays are left empty — compatible
with standard Vulkan 1.x drivers.

---

## Known Language Limitations

These are restrictions of the Tick language (not design choices of this
binding) that affect how Vulkan can be expressed.

### 1. `VkPhysicalDeviceProperties` requires a raw byte buffer

The struct embeds `VkPhysicalDeviceLimits` (121+ mixed fields, ~524 bytes),
making a complete `@dataclass` declaration impractical.  The first six
fields (`apiVersion` … `deviceName[256]`) are read at known byte offsets
using the `vk_read_*` helpers:

```tick
var props : ptr = malloc(VK_PHYSICAL_DEVICE_PROPERTIES_SIZE);
memset(props, 0, VK_PHYSICAL_DEVICE_PROPERTIES_SIZE);
vkGetPhysicalDeviceProperties(device, props);
var name : str = vk_read_str(props, VK_PROPS_OFFSET_DEVICE_NAME);
free(props);
```

All other Vulkan structs in this binding — including
`VkPhysicalDeviceFeatures` (55 fields) and
`VkPhysicalDeviceMemoryProperties` (with `VkMemoryType[32]` and
`VkMemoryHeap[16]`) — are proper `@dataclass` types.

### 2. No C preprocessor macros

Tick does not expose `#define` / `#if defined(...)`.  `VK_MAKE_API_VERSION`
is provided as `vk_make_api_version()`.  All version constants are
pre-computed `const` values:

```tick
const VK_API_VERSION_1_2 : u32 = 4202496;   /* (1 << 22) | (2 << 12) */
```

### 3. `const char* const*` is approximated as `ptr`

Vulkan's `ppEnabledExtensionNames` is typed `const char* const*`.  Tick's
only compatible type is `ptr` (void*):

```tick
var exts : str[] = ["VK_KHR_my_extension"];
create_info.ppEnabledExtensionNames = cast(addr(exts[0]), ptr);
```

Safe as long as `exts` remains live through the Vulkan call.

### 4. `-D` flags do not propagate to imported modules

`@if(SYMBOL)` blocks in `vulkan.tick` always resolve to the `@else` branch when
the file is compiled as an import, even if the parent file was compiled with
`-DSYMBOL`.  As a result, all platform-specific constants (extension name strings,
feature flags, counts) must be declared with `@if` blocks in the **application
file** (`sandbox.tick`), not in the binding module.  This is why the
`VK_EXT_PORTABILITY_*` constants live in `sandbox.tick` rather than `vulkan.tick`.

### 5. Avoid C standard-library function names

The Tick runtime links `-lm`.  Any Tick function whose name matches a C
standard-library symbol (e.g. `log`, `pow`, `yn`) will produce a
"conflicting types" GCC error.  This binding uses `bool_str` instead of
the shorter `yn` for exactly this reason.

---

## Binding layer API reference

### Platform constants (set by `-DMACOS`)

| Name | macOS value | non-macOS value |
|------|-------------|-----------------|
| `VK_EXT_PORTABILITY_ENUMERATION` | `"VK_KHR_portability_enumeration"` | `""` |
| `VK_EXT_GET_PHYS_DEV_PROPERTIES2` | `"VK_KHR_get_physical_device_properties2"` | `""` |
| `VK_DEV_EXT_PORTABILITY_SUBSET` | `"VK_KHR_portability_subset"` | `""` |
| `PLATFORM_INSTANCE_FLAGS` | `1` | `0` |
| `PLATFORM_INSTANCE_EXT_COUNT` | `2` | `0` |
| `PLATFORM_DEVICE_EXT_COUNT` | `1` | `0` |

### Result / type constants

| Name | Type | Value / purpose |
|------|------|-----------------|
| `VK_SUCCESS` | i32 | 0 |
| `VK_API_VERSION_1_0 … 1_3` | u32 | Packed Vulkan API version constants |
| `VK_PHYSICAL_DEVICE_PROPERTIES_SIZE` | u64 | Safe `malloc` size for raw props buffer |
| `VK_PROPS_OFFSET_DEVICE_NAME` | i32 | 20 — `deviceName` byte offset |
| `VK_QUEUE_GRAPHICS_BIT` | u32 | 1 |
| `VK_QUEUE_COMPUTE_BIT` | u32 | 2 |
| `VK_QUEUE_TRANSFER_BIT` | u32 | 4 |

(See `vulkan.tick` for the complete listing.)

### Structs (`@dataclass`)

```
VkApplicationInfo
VkInstanceCreateInfo
VkExtent3D
VkQueueFamilyProperties
VkDeviceQueueCreateInfo
VkDeviceCreateInfo
VkPhysicalDeviceFeatures          ← all 55 VkBool32 fields
VkMemoryType
VkMemoryHeap
VkPhysicalDeviceMemoryProperties  ← VkMemoryType[32] + VkMemoryHeap[16]
```

### Extern functions

```
vkCreateInstance / vkDestroyInstance
vkEnumeratePhysicalDevices
vkGetPhysicalDeviceProperties         ← takes ptr (raw buffer)
vkGetPhysicalDeviceFeatures           ← takes ptr<VkPhysicalDeviceFeatures>
vkGetPhysicalDeviceMemoryProperties   ← takes ptr<VkPhysicalDeviceMemoryProperties>
vkGetPhysicalDeviceQueueFamilyProperties
vkCreateDevice / vkDestroyDevice
vkGetDeviceQueue
vkEnumerateInstanceExtensionProperties
vkEnumerateInstanceLayerProperties
```

### Helper functions

| Function | Returns | Purpose |
|----------|---------|---------|
| `vk_read_u32(mem, offset)` | u32 | Read u32 from raw buffer at byte offset |
| `vk_read_u64(mem, offset)` | u64 | Read u64 from raw buffer at byte offset |
| `vk_read_str(mem, offset)` | str | Read char* from raw buffer at byte offset |
| `vk_api_major/minor/patch(v)` | u32 | Decode packed version field |
| `vk_make_api_version(M, m, p)` | u32 | Pack version fields |
| `vk_device_type_str(t)` | str | Human-readable device type name |
| `vk_result_str(r)` | str | Human-readable VkResult name |
| `vk_print_queue_flags(f)` | void | Print queue capability flags |
