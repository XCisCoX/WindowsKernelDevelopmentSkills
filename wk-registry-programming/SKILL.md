---
name: wk-registry-programming
description: Use when a Windows driver must read or persist configuration - registry layout and hives, ZwCreateKey/ZwOpenKey/ZwQueryValueKey/ZwSetValueKey/ZwEnumerateKey with the exact KEY_* information classes, RtlQueryRegistryValues and its query table flags, the service Parameters key and IoOpenDeviceRegistryKey, REG_SZ/REG_DWORD/REG_BINARY/REG_MULTI_SZ handling, safe enumeration, and cleanup on every path.
---

# Registry access from kernel mode

The registry is the configuration store a driver is expected to use, and the kernel-side API is a handle-based object interface with its own rules: keys are objects, values are typed blobs, access is checked against the calling thread's effective token, and every buffer has a size that the configuration manager reports only after a failed first call. The mental model that keeps this code correct: a key handle behaves like a file handle (open, query, close, and free everything on the way out), and a value read is a two-call protocol (ask for the size, allocate, ask again). The rule that prevents most disasters: never trust the size, type, or termination of registry data, and never open configuration in an arbitrary client thread, because the token there is not the token `DriverEntry` ran with.

## When to use

- A driver needs tunables at load: timeouts, buffer sizes, feature flags, protected identifier lists, log paths.
- A driver must persist state at unload or on demand: counters, last run time, last known configuration.
- Reading the service `Parameters` key that an INF or `reg add` populated.
- A PnP driver must read or write its device or driver key through `IoOpenDeviceRegistryKey`.
- Enumerating values or subkeys to discover what an operator configured, or to log unknown values.
- Converting between `REG_SZ`, `REG_EXPAND_SZ`, `REG_MULTI_SZ`, `REG_DWORD`, `REG_QWORD`, `REG_BINARY` and kernel data.
- Diagnosing `STATUS_OBJECT_NAME_NOT_FOUND`, `STATUS_BUFFER_TOO_SMALL`, `STATUS_OBJECT_TYPE_MISMATCH`, or a driver that reads a value the user "definitely set".

Do not use for: intercepting or blocking registry operations, which needs a callback (`wk-registry-callbacks`); general object and handle mechanics (`wk-object-manager`); safe string construction and `UNICODE_STRING` sizing (`wk-kernel-strings`); the IOCTL protocol that might carry configuration instead (`wk-ioctl-design`); pool allocation rules (`wk-kernel-memory-pools`).

## Prerequisites and related skills

- Kernel C, WDK 10.0.22621 or later, x64. `<ntddk.h>` or `<ntifs.h>`; `<ntstrsafe.h>` for `RtlStringCchCopyW`/`RtlStringCchCatW`.
- **Every routine here runs at PASSIVE_LEVEL.** `ZwCreateKey`, `ZwQueryValueKey`, `ZwSetValueKey`, `ZwEnumerateKey`, `ZwFlushKey`, and `RtlQueryRegistryValues` may block on hive I/O and may allocate.
- Handles created with `OBJ_KERNEL_HANDLE` live in the system handle table; without it, a handle created while a user thread is current lands in that process's table and is reachable from user mode.
- Related skills, and why: `wk-object-manager` for `OBJECT_ATTRIBUTES` and handle semantics; `wk-kernel-strings` for `UNICODE_STRING` and counted-string pitfalls; `wk-kernel-memory-pools` for `ExAllocatePool2` tags and cleanup; `wk-registry-callbacks` when operations must be filtered rather than performed; `wk-driver-anatomy` for the driver skeleton that calls this code.

## Mental model

### Two names for the same tree

User-mode APIs expose `HKEY_*` roots, and the configuration manager translates them into object-manager paths under `\Registry`. Only the kernel paths exist as objects; `HKEY_LOCAL_MACHINE` is a user-mode alias.

```
\Registry
+-- Machine                      <- HKEY_LOCAL_MACHINE
|   +-- System
|   |   +-- CurrentControlSet --symlink--> ControlSet001
|   |   |   +-- Services
|   |   |       +-- RegDemo                   <- DriverEntry RegistryPath
|   |   |           +-- Parameters            <- operator tunables live here
|   |   |           +-- LastRun               <- state written at unload
|   |   +-- ControlSet001
|   |   +-- Select
|   +-- Software
|   |   +-- Classes                           <- machine half of HKEY_CLASSES_ROOT
|   +-- Security                              <- protected, needs SYSTEM
|   +-- Sam
|   +-- Hardware                              <- volatile, rebuilt at boot
+-- User
|   +-- S-1-5-21-...-1001                     <- HKEY_CURRENT_USER for that user
+-- A                                         <- application hives
+-- WC                                        <- Windows containers
```

| User-facing root | Kernel path | Backing store |
|---|---|---|
| `HKEY_LOCAL_MACHINE\SYSTEM` | `\Registry\Machine\System` | `%SystemRoot%\System32\config\SYSTEM` |
| `HKEY_LOCAL_MACHINE\SOFTWARE` | `\Registry\Machine\Software` | `...\config\SOFTWARE` |
| `HKEY_LOCAL_MACHINE\SECURITY` | `\Registry\Machine\Security` | `...\config\SECURITY` |
| `HKEY_LOCAL_MACHINE\SAM` | `\Registry\Machine\Sam` | `...\config\SAM` |
| `HKEY_LOCAL_MACHINE\HARDWARE` | `\Registry\Machine\Hardware` | volatile, built at boot, no file |
| `HKEY_LOCAL_MACHINE\BCD00000000` | `\Registry\Machine\BCD00000000` | `\Boot\BCD` |
| `HKEY_USERS\<SID>` | `\Registry\User\<SID>` | `Users\<user>\NTUSER.DAT` |
| `HKEY_USERS\.DEFAULT` | `\Registry\User\.Default` | `...\config\DEFAULT` |
| `HKEY_CURRENT_USER` | `\Registry\User\<caller's SID>` | that user's `NTUSER.DAT` |
| `HKEY_CLASSES_ROOT` | **no single key** | merged view: machine `Software\Classes`, then the user's |
| — | `\Registry\A` | application hives loaded with `NtLoadKeyEx` |
| — | `\Registry\WC` | container and silo keys, Windows 10 1809 and later |

Facts that follow:

- `CurrentControlSet` is a **symbolic link** to one `ControlSet00N`, chosen at boot. Open through it, never hard-code `ControlSet001`.
- `HKEY_CLASSES_ROOT` and `HKEY_CURRENT_USER` are conveniences, not kernel keys. A driver that needs per-user data must resolve the SID and open `\Registry\User\<SID>` explicitly; it must not assume the calling thread's token is the user it wants.
- **No WOW64 redirection and no UAC virtualization applies to kernel callers.** A 32-bit installer that wrote `HKLM\Software\Vendor` without `KEY_WOW64_64KEY` wrote to `HKLM\Software\WOW6432Node\Vendor`, and the driver reading `\Registry\Machine\Software\Vendor` will not see it.
- `REG_OPTION_VOLATILE` keys exist only in memory and disappear at shutdown; they are useful for counters that must not wear the hive.

### Keys are objects, values are typed blobs

A key handle is a real object handle: reference counted, access-masked, and required for every value operation. Values have no handles; they are named within a key and carry an explicit type (`REG_DWORD` is a 4-byte little-endian blob with a type tag, not an integer type in the API).

### Reads are a two-call protocol

`ZwQueryValueKey` returns `STATUS_BUFFER_TOO_SMALL` and fills `ResultLength` when the buffer is short. The canonical sequence is: call with a stack buffer sized for the common case, and only on a size failure allocate exactly `ResultLength` bytes and call again. `KEY_VALUE_PARTIAL_INFORMATION` puts the data at a fixed offset; `KEY_VALUE_FULL_INFORMATION` puts the name first and the data at `DataOffset`, which is aligned and **must not be computed by hand**.

## Key APIs and structures

| API | Purpose | IRQL | Buffer ownership |
|---|---|---|---|
| `ZwCreateKey` | create or open, returns handle and disposition | PASSIVE | caller closes; `Disposition` says created vs opened |
| `ZwOpenKey` | open existing key | PASSIVE | caller closes |
| `ZwQueryKey` | class, subkey/value counts, max sizes | PASSIVE | caller-allocated, size-probed |
| `ZwSetValueKey` | create or replace a value | PASSIVE | API copies the data before returning |
| `ZwQueryValueKey` | read a value, typed | PASSIVE | caller-allocated, size-probed |
| `ZwEnumerateKey` | subkey by index | PASSIVE | caller-allocated, may be short |
| `ZwEnumerateValueKey` | value by index | PASSIVE | caller-allocated, may be short |
| `ZwDeleteKey` | delete an empty key | PASSIVE | handle invalid after success |
| `ZwDeleteValueKey` | delete one value | PASSIVE | — |
| `ZwFlushKey` | force the hive to disk | PASSIVE, slow | blocks until the log is written |
| `ZwClose` | release the handle | PASSIVE | every path |
| `RtlQueryRegistryValues` | read many named values in one call | PASSIVE | callback receives data, or direct copy |
| `IoOpenDeviceRegistryKey` | PnP device or driver key | PASSIVE | caller closes; needs a PDO |

```c
NTSTATUS ZwCreateKey(PHANDLE KeyHandle, ACCESS_MASK DesiredAccess,
                     POBJECT_ATTRIBUTES ObjectAttributes, ULONG TitleIndex,
                     PUNICODE_STRING Class, ULONG CreateOptions, PULONG Disposition);
NTSTATUS ZwOpenKey(PHANDLE KeyHandle, ACCESS_MASK DesiredAccess,
                   POBJECT_ATTRIBUTES ObjectAttributes);
NTSTATUS ZwSetValueKey(HANDLE KeyHandle, PUNICODE_STRING ValueName, ULONG TitleIndex,
                       ULONG Type, PVOID Data, ULONG DataSize);
NTSTATUS ZwQueryValueKey(HANDLE KeyHandle, PUNICODE_STRING ValueName,
                         KEY_VALUE_INFORMATION_CLASS KeyValueInformationClass,
                         PVOID KeyValueInformation, ULONG Length, PULONG ResultLength);
NTSTATUS ZwEnumerateKey(HANDLE KeyHandle, ULONG Index,
                        KEY_INFORMATION_CLASS KeyInformationClass,
                        PVOID KeyInformation, ULONG Length, PULONG ResultLength);
NTSTATUS ZwEnumerateValueKey(HANDLE KeyHandle, ULONG Index,
                             KEY_VALUE_INFORMATION_CLASS KeyValueInformationClass,
                             PVOID KeyValueInformation, ULONG Length, PULONG ResultLength);
```

### Information classes

| `KEY_INFORMATION_CLASS` | Value | Structure | Use |
|---|---|---|---|
| `KeyBasicInformation` | 0 | `KEY_BASIC_INFORMATION` | name and last-write time |
| `KeyNodeInformation` | 1 | `KEY_NODE_INFORMATION` | name plus class name |
| `KeyFullInformation` | 2 | `KEY_FULL_INFORMATION` | counts and maximum sizes; size enumeration buffers |
| `KeyNameInformation` | 3 | `KEY_NAME_INFORMATION` | name only |
| `KeyCachedInformation` | 4 | `KEY_CACHED_INFORMATION` | cached summary |
| `KeyFlagsInformation` | 5 | `KEY_FLAGS_INFORMATION` | key flags |
| `KeyVirtualizationInformation` | 6 | `KEY_VIRTUALIZATION_INFORMATION` | user-mode virtualization state |
| `KeyHandleTagsInformation` | 7 | `KEY_HANDLE_TAGS_INFORMATION` | handle tags |
| `KeyTrustInformation` | 8 | `KEY_TRUST_INFORMATION` | trust state |
| `KeyLayerInformation` | 9 | `KEY_LAYER_INFORMATION` | layered-key information |

| `KEY_VALUE_INFORMATION_CLASS` | Value | Structure | Use |
|---|---|---|---|
| `KeyValueBasicInformation` | 0 | `KEY_VALUE_BASIC_INFORMATION` | name and type, no data |
| `KeyValueFullInformation` | 1 | `KEY_VALUE_FULL_INFORMATION` | name, type, data at `DataOffset` |
| `KeyValuePartialInformation` | 2 | `KEY_VALUE_PARTIAL_INFORMATION` | type and data, no name |
| `KeyValueFullInformationAlign64` | 3 | as full | 64-bit values aligned |
| `KeyValuePartialInformationAlign64` | 4 | as partial | use for `REG_QWORD` |
| `KeyValueLayerInformation` | 5 | `KEY_VALUE_LAYER_INFORMATION` | layered values |

```c
/* KEY_BASIC_INFORMATION: LastWriteTime, TitleIndex, NameLength, Name at 0x10. */
typedef struct _KEY_FULL_INFORMATION {       /* header 0x2C, Class at 0x2C */
    LARGE_INTEGER LastWriteTime;
    ULONG TitleIndex;
    ULONG ClassOffset;
    ULONG ClassLength;
    ULONG SubKeys;
    ULONG MaxNameLen;
    ULONG MaxClassLen;
    ULONG Values;
    ULONG MaxValueNameLen;
    ULONG MaxValueDataLen;
    WCHAR Class[1];
} KEY_FULL_INFORMATION, *PKEY_FULL_INFORMATION;

typedef struct _KEY_VALUE_PARTIAL_INFORMATION {   /* header 0x0C, Data at 0x0C */
    ULONG TitleIndex;
    ULONG Type;
    ULONG DataLength;
    UCHAR Data[1];
} KEY_VALUE_PARTIAL_INFORMATION, *PKEY_VALUE_PARTIAL_INFORMATION;

typedef struct _KEY_VALUE_FULL_INFORMATION {      /* header 0x14, Name at 0x14 */
    ULONG TitleIndex;
    ULONG Type;
    ULONG DataOffset;            /* data starts here, aligned: never assume Name end */
    ULONG DataLength;
    ULONG NameLength;
    WCHAR Name[1];
} KEY_VALUE_FULL_INFORMATION, *PKEY_VALUE_FULL_INFORMATION;

/* KEY_VALUE_BASIC_INFORMATION: TitleIndex, Type, NameLength, Name at 0x0C. */
```

### Value types

| Type | Value | Storage | Kernel handling |
|---|---|---|---|
| `REG_NONE` | 0 | undefined | treat as opaque bytes |
| `REG_SZ` | 1 | UTF-16, NUL-terminated | `DataLength` includes the terminator; validate it |
| `REG_EXPAND_SZ` | 2 | UTF-16 with `%VAR%` | `RtlQueryRegistryValues` expands unless `RTL_QUERY_REGISTRY_NOEXPAND`; raw `ZwQueryValueKey` never expands |
| `REG_BINARY` | 3 | arbitrary bytes | most robust for structures; add a version field |
| `REG_DWORD` | 4 | 4 bytes little-endian | `DataLength` must be exactly 4 |
| `REG_DWORD_BIG_ENDIAN` | 5 | 4 bytes big-endian | convert explicitly |
| `REG_LINK` | 6 | symbolic link target | key links only |
| `REG_MULTI_SZ` | 7 | strings, each NUL, then an empty string | double-NUL; parse with an explicit bound |
| `REG_RESOURCE_LIST` | 8 | hardware resources | not a driver setting |
| `REG_FULL_RESOURCE_DESCRIPTOR` | 9 | hardware resources | not a driver setting |
| `REG_QWORD` | 11 | 8 bytes little-endian | read with the `Align64` class |

| Access mask | Value | Notes |
|---|---|---|
| `KEY_QUERY_VALUE` | `0x0001` | read one value |
| `KEY_SET_VALUE` | `0x0002` | write or delete a value |
| `KEY_CREATE_SUB_KEY` | `0x0004` | create below this key |
| `KEY_ENUMERATE_SUB_KEYS` | `0x0008` | list subkeys |
| `KEY_NOTIFY` | `0x0010` | change notification |
| `KEY_ALL_ACCESS` | `0xF003F` | everything; avoid in drivers |

### RtlQueryRegistryValues

```c
typedef NTSTATUS (*PRTL_QUERY_REGISTRY_ROUTINE)(
    PWSTR ValueName, ULONG ValueType, PVOID ValueData, ULONG ValueLength,
    PVOID Context, PVOID EntryContext);

typedef struct _RTL_QUERY_REGISTRY_TABLE {
    PRTL_QUERY_REGISTRY_ROUTINE QueryRoutine;   /* NULL + DIRECT copies instead */
    ULONG Flags;
    PWSTR Name;
    PVOID EntryContext;
    ULONG DefaultType;                          /* REG_NONE: no default */
    PVOID DefaultData;
    ULONG DefaultLength;
} RTL_QUERY_REGISTRY_TABLE, *PRTL_QUERY_REGISTRY_TABLE;

NTSTATUS RtlQueryRegistryValues(ULONG RelativeTo, PCWSTR Path,
                                PRTL_QUERY_REGISTRY_TABLE QueryTable,
                                PVOID Context, PVOID Environment);
```

| Flag | Value | Meaning |
|---|---|---|
| `RTL_QUERY_REGISTRY_SUBKEY` | `0x00000001` | `Name` is a subkey: descend, do not read a value |
| `RTL_QUERY_REGISTRY_TOPKEY` | `0x00000002` | restore the original key |
| `RTL_QUERY_REGISTRY_REQUIRED` | `0x00000004` | missing value fails the whole call |
| `RTL_QUERY_REGISTRY_NOVALUE` | `0x00000008` | do not call the routine, only descend |
| `RTL_QUERY_REGISTRY_NOEXPAND` | `0x00000010` | leave `REG_EXPAND_SZ` unexpanded |
| `RTL_QUERY_REGISTRY_DIRECT` | `0x00000020` | copy into `EntryContext` without a routine |
| `RTL_QUERY_REGISTRY_DELETE` | `0x00000040` | delete the value after reading it |

`RelativeTo` accepts `RTL_REGISTRY_ABSOLUTE` (`0`), `RTL_REGISTRY_SERVICES` (`1`), `RTL_REGISTRY_CONTROL` (`2`), `RTL_REGISTRY_WINDOWS_NT` (`3`), `RTL_REGISTRY_DEVICEMAP` (`4`), `RTL_REGISTRY_USER` (`5`), `RTL_REGISTRY_HANDLE` (`0x40000000`), optionally ORed with `RTL_REGISTRY_OPTIONAL` (`0x80000000`). The table is terminated by an all-zero entry.

### Where a driver's configuration lives

| Key | Opened with | Typical content |
|---|---|---|
| `\Registry\Machine\System\CurrentControlSet\Services\<name>` | `DriverEntry`'s `RegistryPath` | service metadata, and the `Parameters` subkey |
| `...\Services\<name>\Parameters` | `ZwOpenKey` on the path above plus `\Parameters` | operator tunables; INF `AddReg` or `reg add` |
| Device hardware key | `IoOpenDeviceRegistryKey(DeviceObject, PLUGPLAY_REGKEY_DEVICE, ...)` | instance data, `\Enum\<bus>\<id>\<instance>` |
| Device driver key | `IoOpenDeviceRegistryKey(DeviceObject, PLUGPLAY_REGKEY_DRIVER, ...)` | the `Control\Class\{GUID}\####` software key |
| Device interface key | `IoOpenDeviceInterfaceRegistryKey` or `PLUGPLAY_REGKEY_INTERFACE` | per-interface settings |

`PLUGPLAY_REGKEY_DEVICE` is `1`, `PLUGPLAY_REGKEY_DRIVER` is `2`, `PLUGPLAY_REGKEY_INTERFACE` is `3`. `IoOpenDeviceRegistryKey` requires the **physical device object** of a PnP device; on a driver-created (non-PnP) device object it fails with `STATUS_INVALID_PARAMETER`. A legacy driver uses `RegistryPath` instead.

## Step-by-step

1. Decide the storage location: tunables under `...\Services\<name>\Parameters`, state under the same service key, per-device data under the device key. Do not put configuration in `\Registry\Machine\Software\<Vendor>` unless a user-mode installer owns it.
2. Copy `RegistryPath` into driver-owned storage during `DriverEntry`; the string is not guaranteed to outlive the call. Bound the copy and NUL-terminate it.
3. Build the `Parameters` path by appending `\Parameters` with a bounded string function, and keep a separate `WCHAR` buffer for it.
4. Open with `InitializeObjectAttributes(... OBJ_CASE_INSENSITIVE | OBJ_KERNEL_HANDLE ...)` and the narrowest access mask (`KEY_QUERY_VALUE` to read, `KEY_SET_VALUE` to write). Check the status; a missing key is `STATUS_OBJECT_NAME_NOT_FOUND`.
5. Read each value with the two-call protocol: stack buffer first, then an exact-size paged allocation. Validate `Type` and `DataLength` before touching `Data`.
6. Treat every value as untrusted input: clamp numbers into range, reject unknown types, validate `REG_SZ` termination, and bound `REG_MULTI_SZ` parsing by `DataLength`.
7. Apply the configuration, keeping defaults for anything missing or malformed, and log which values were used.
8. On unload, write state with a versioned `REG_BINARY` structure, call `ZwFlushKey` once, and close the handle.
9. Close every handle on every path, in reverse order of opening, and free every pool block with its tag. Check by reading the routine end to end looking for a `return` that skips a close.

## Complete example

A kernel service that reads `Parameters\IntervalSeconds`, `Parameters\LogPath`, and `Parameters\FilterList`, enumerates everything under `Parameters`, and writes a `LastRun` binary record at unload.

```ini
; RegDemo.inf - fragment that creates the service Parameters key
[RegDemo_Service]
ServiceType    = 1
StartType      = 3
ErrorControl   = 1
ServiceBinary  = %12%\RegDemo.sys
AddReg         = RegDemo_Params

[RegDemo_Params]
HKR,Parameters,IntervalSeconds,0x00010001,30
HKR,Parameters,LogPath,0x00000000,"\SystemRoot\Temp\regdemo.log"
HKR,Parameters,FilterList,0x00010000,"tcpip","http","dns"
HKR,Parameters,Verbose,0x00010001,0
```

```c
/* RegDemo.c - load configuration, enumerate it, persist state at unload */
#include <ntddk.h>
#include <ntstrsafe.h>

#define REGDEMO_TAG            'meDR'
#define REGDEMO_MAX_PATH       260
#define REGDEMO_MAX_FILTERS    8
#define REGDEMO_FILTER_CHARS   48
#define REGDEMO_MIN_INTERVAL   1
#define REGDEMO_MAX_INTERVAL   3600

typedef struct _REGDEMO_CONFIG {
    ULONG  IntervalSeconds;
    ULONG  Verbose;
    WCHAR  LogPath[REGDEMO_MAX_PATH];
    WCHAR  Filters[REGDEMO_MAX_FILTERS][REGDEMO_FILTER_CHARS];
    ULONG  FilterCount;
} REGDEMO_CONFIG;

typedef struct _REGDEMO_STATE {
    WCHAR          ServicePath[300];
    WCHAR          ParametersPath[320];
    REGDEMO_CONFIG Config;
    ULONG          LoadCount;
} REGDEMO_STATE;

static REGDEMO_STATE g_State;

static NTSTATUS BuildParametersPath(VOID)
{
    NTSTATUS status;
    status = RtlStringCchCopyW(g_State.ParametersPath,
                               ARRAYSIZE(g_State.ParametersPath),
                               g_State.ServicePath);
    if (!NT_SUCCESS(status))
        return status;
    return RtlStringCchCatW(g_State.ParametersPath,
                            ARRAYSIZE(g_State.ParametersPath), L"\\Parameters");
}

static NTSTATUS OpenParameters(ACCESS_MASK Access, PHANDLE Key)
{
    OBJECT_ATTRIBUTES attributes;
    UNICODE_STRING path;
    RtlInitUnicodeString(&path, g_State.ParametersPath);
    InitializeObjectAttributes(&attributes, &path,
                               OBJ_CASE_INSENSITIVE | OBJ_KERNEL_HANDLE,
                               NULL, NULL);
    return ZwOpenKey(Key, Access, &attributes);
}

/* Two-call read for a DWORD: stack buffer, then type and length checks. */
static NTSTATUS ReadDwordValue(HANDLE Key, PCWSTR ValueName, ULONG Default, PULONG Value)
{
    UCHAR buffer[sizeof(KEY_VALUE_PARTIAL_INFORMATION) + sizeof(ULONG)];
    PKEY_VALUE_PARTIAL_INFORMATION info = (PKEY_VALUE_PARTIAL_INFORMATION)buffer;
    UNICODE_STRING name;
    ULONG length = 0;
    NTSTATUS status;
    RtlInitUnicodeString(&name, ValueName);
    status = ZwQueryValueKey(Key, &name, KeyValuePartialInformation,
                             info, sizeof(buffer), &length);
    if (status == STATUS_OBJECT_NAME_NOT_FOUND) {
        *Value = Default;
        return STATUS_SUCCESS;
    }
    if (!NT_SUCCESS(status))
        return status;
    if (info->Type != REG_DWORD || info->DataLength != sizeof(ULONG))
        return STATUS_OBJECT_TYPE_MISMATCH;
    RtlCopyMemory(Value, info->Data, sizeof(ULONG));
    return STATUS_SUCCESS;
}

/* Size probe, exact allocation, bounded copy, explicit terminator. */
static NTSTATUS ReadStringValue(HANDLE Key, PCWSTR ValueName, PWSTR Output, ULONG OutputChars)
{
    UNICODE_STRING name;
    PKEY_VALUE_PARTIAL_INFORMATION info;
    NTSTATUS status;
    ULONG length = 0;
    ULONG chars;
    RtlInitUnicodeString(&name, ValueName);
    status = ZwQueryValueKey(Key, &name, KeyValuePartialInformation, NULL, 0, &length);
    if (status != STATUS_BUFFER_TOO_SMALL && status != STATUS_BUFFER_OVERFLOW &&
        !NT_SUCCESS(status))
        return status;
    if (length < sizeof(KEY_VALUE_PARTIAL_INFORMATION) + sizeof(WCHAR))
        return STATUS_INVALID_PARAMETER;
    info = (PKEY_VALUE_PARTIAL_INFORMATION)ExAllocatePool2(POOL_FLAG_PAGED, length, REGDEMO_TAG);
    if (info == NULL)
        return STATUS_INSUFFICIENT_RESOURCES;
    status = ZwQueryValueKey(Key, &name, KeyValuePartialInformation, info, length, &length);
    if (!NT_SUCCESS(status)) {
        ExFreePoolWithTag(info, REGDEMO_TAG);
        return status;
    }
    if (info->Type != REG_SZ || info->DataLength < sizeof(WCHAR)) {
        ExFreePoolWithTag(info, REGDEMO_TAG);
        return STATUS_OBJECT_TYPE_MISMATCH;
    }
    chars = info->DataLength / sizeof(WCHAR);
    while (chars > 0 && ((PWSTR)info->Data)[chars - 1] == L'\0')
        chars--;                                  /* DataLength counts the terminator */
    if (chars >= OutputChars) {
        ExFreePoolWithTag(info, REGDEMO_TAG);
        return STATUS_BUFFER_TOO_SMALL;
    }
    RtlCopyMemory(Output, info->Data, chars * sizeof(WCHAR));
    Output[chars] = L'\0';
    ExFreePoolWithTag(info, REGDEMO_TAG);
    return STATUS_SUCCESS;
}

/* Bounded MULTI_SZ parse: never call wcslen on registry data. */
static VOID ReadFilterList(HANDLE Key, PCWSTR ValueName, REGDEMO_CONFIG *Config)
{
    UNICODE_STRING name;
    PKEY_VALUE_PARTIAL_INFORMATION info;
    NTSTATUS status;
    ULONG length = 0, chars, index = 0, start, itemLen, copyLen;
    RtlInitUnicodeString(&name, ValueName);
    status = ZwQueryValueKey(Key, &name, KeyValuePartialInformation, NULL, 0, &length);
    if (status != STATUS_BUFFER_TOO_SMALL && status != STATUS_BUFFER_OVERFLOW &&
        !NT_SUCCESS(status))
        return;
    if (length < sizeof(KEY_VALUE_PARTIAL_INFORMATION))
        return;
    info = (PKEY_VALUE_PARTIAL_INFORMATION)ExAllocatePool2(POOL_FLAG_PAGED, length, REGDEMO_TAG);
    if (info == NULL)
        return;
    status = ZwQueryValueKey(Key, &name, KeyValuePartialInformation, info, length, &length);
    if (NT_SUCCESS(status) && info->Type == REG_MULTI_SZ) {
        PWSTR data = (PWSTR)info->Data;
        chars = info->DataLength / sizeof(WCHAR);
        while (index < chars && Config->FilterCount < REGDEMO_MAX_FILTERS) {
            if (data[index] == L'\0')
                break;                            /* double NUL: end of list */
            start = index;
            itemLen = 0;
            while (index + itemLen < chars && data[index + itemLen] != L'\0')
                itemLen++;
            if (index + itemLen >= chars)
                break;                            /* unterminated: reject the tail */
            copyLen = min(itemLen, REGDEMO_FILTER_CHARS - 1);
            RtlCopyMemory(Config->Filters[Config->FilterCount], data + start,
                          copyLen * sizeof(WCHAR));
            Config->Filters[Config->FilterCount][copyLen] = L'\0';
            Config->FilterCount++;
            index += itemLen + 1;
        }
    }
    ExFreePoolWithTag(info, REGDEMO_TAG);
}

static VOID LoadConfig(VOID)
{
    HANDLE key = NULL;
    NTSTATUS status;
    ULONG value;
    g_State.Config.IntervalSeconds = 30;
    g_State.Config.Verbose = 0;
    g_State.Config.FilterCount = 0;
    RtlStringCchCopyW(g_State.Config.LogPath, ARRAYSIZE(g_State.Config.LogPath),
                      L"\\SystemRoot\\Temp\\regdemo.log");
    status = OpenParameters(KEY_QUERY_VALUE, &key);
    if (!NT_SUCCESS(status)) {
        KdPrint(("RegDemo: Parameters missing (0x%08X), using defaults\n", status));
        return;
    }
    if (NT_SUCCESS(ReadDwordValue(key, L"IntervalSeconds", 30, &value))) {
        if (value >= REGDEMO_MIN_INTERVAL && value <= REGDEMO_MAX_INTERVAL)
            g_State.Config.IntervalSeconds = value;
        else
            KdPrint(("RegDemo: IntervalSeconds %lu out of range, clamped\n", value));
    }
    ReadDwordValue(key, L"Verbose", 0, &g_State.Config.Verbose);      /* best effort */
    ReadStringValue(key, L"LogPath", g_State.Config.LogPath,
                    ARRAYSIZE(g_State.Config.LogPath));               /* best effort */
    ReadFilterList(key, L"FilterList", &g_State.Config);
    ZwClose(key);
    KdPrint(("RegDemo: interval=%lu verbose=%lu filters=%lu log=%ws\n",
             g_State.Config.IntervalSeconds, g_State.Config.Verbose,
             g_State.Config.FilterCount, g_State.Config.LogPath));
}

static VOID EnumerateParameters(VOID)
{
    OBJECT_ATTRIBUTES attributes;
    UNICODE_STRING path, name;
    PKEY_VALUE_FULL_INFORMATION info;
    HANDLE key = NULL;
    NTSTATUS status;
    ULONG index = 0, length = 0, size = 512;
    RtlInitUnicodeString(&path, g_State.ParametersPath);
    InitializeObjectAttributes(&attributes, &path,
                               OBJ_CASE_INSENSITIVE | OBJ_KERNEL_HANDLE, NULL, NULL);
    status = ZwOpenKey(&key, KEY_QUERY_VALUE, &attributes);
    if (!NT_SUCCESS(status))
        return;
    info = (PKEY_VALUE_FULL_INFORMATION)ExAllocatePool2(POOL_FLAG_PAGED, size, REGDEMO_TAG);
    if (info == NULL) {
        ZwClose(key);
        return;
    }
    for (;;) {
        status = ZwEnumerateValueKey(key, index, KeyValueFullInformation, info, size, &length);
        if (status == STATUS_BUFFER_OVERFLOW || status == STATUS_BUFFER_TOO_SMALL) {
            ExFreePoolWithTag(info, REGDEMO_TAG);
            size = length;
            info = (PKEY_VALUE_FULL_INFORMATION)ExAllocatePool2(POOL_FLAG_PAGED, size, REGDEMO_TAG);
            if (info == NULL) {
                ZwClose(key);
                return;
            }
            continue;                 /* same index, bigger buffer */
        }
        if (status == STATUS_NO_MORE_ENTRIES)
            break;                    /* warning status: NT_SUCCESS is TRUE here */
        if (!NT_SUCCESS(status)) {
            KdPrint(("RegDemo: enumerate failed 0x%08X at index %lu\n", status, index));
            break;
        }
        name.Buffer = (PWSTR)((PUCHAR)info + FIELD_OFFSET(KEY_VALUE_FULL_INFORMATION, Name));
        name.Length = (USHORT)info->NameLength;
        name.MaximumLength = (USHORT)info->NameLength;
        if (info->Type == REG_DWORD && info->DataLength == sizeof(ULONG))
            KdPrint(("RegDemo: value %wZ = %lu\n", &name,
                     *(PULONG)((PUCHAR)info + info->DataOffset)));
        else
            KdPrint(("RegDemo: value %wZ type %lu size %lu\n", &name,
                     info->Type, info->DataLength));
        index++;
    }
    ExFreePoolWithTag(info, REGDEMO_TAG);
    ZwClose(key);
}

static VOID SaveState(VOID)
{
    OBJECT_ATTRIBUTES attributes;
    UNICODE_STRING path, valueName;
    HANDLE key = NULL;
    NTSTATUS status;
    LARGE_INTEGER now;
    struct {
        ULONG Version;
        ULONG LoadCount;
        ULONG IntervalSeconds;
        ULONG FilterCount;
        LARGE_INTEGER Timestamp;
    } record;
    RtlInitUnicodeString(&path, g_State.ParametersPath);
    InitializeObjectAttributes(&attributes, &path,
                               OBJ_CASE_INSENSITIVE | OBJ_KERNEL_HANDLE, NULL, NULL);
    status = ZwOpenKey(&key, KEY_SET_VALUE, &attributes);
    if (!NT_SUCCESS(status)) {
        KdPrint(("RegDemo: cannot open Parameters for write (0x%08X)\n", status));
        return;
    }
    KeQuerySystemTimePrecise(&now);
    record.Version = 1;
    record.LoadCount = g_State.LoadCount;
    record.IntervalSeconds = g_State.Config.IntervalSeconds;
    record.FilterCount = g_State.Config.FilterCount;
    record.Timestamp = now;
    RtlInitUnicodeString(&valueName, L"LastRun");
    status = ZwSetValueKey(key, &valueName, 0, REG_BINARY, &record, sizeof(record));
    if (!NT_SUCCESS(status))
        KdPrint(("RegDemo: ZwSetValueKey failed 0x%08X\n", status));
    else if (!NT_SUCCESS(ZwFlushKey(key)))
        KdPrint(("RegDemo: ZwFlushKey failed\n"));
    ZwClose(key);
}

static VOID RegDemoUnload(PDRIVER_OBJECT DriverObject)
{
    UNREFERENCED_PARAMETER(DriverObject);
    SaveState();
    KdPrint(("RegDemo: unloaded\n"));
}

NTSTATUS DriverEntry(PDRIVER_OBJECT DriverObject, PUNICODE_STRING RegistryPath)
{
    if (RegistryPath == NULL || RegistryPath->Length == 0 ||
        RegistryPath->Length >= sizeof(g_State.ServicePath))
        return STATUS_INVALID_PARAMETER;
    RtlCopyMemory(g_State.ServicePath, RegistryPath->Buffer, RegistryPath->Length);
    g_State.ServicePath[RegistryPath->Length / sizeof(WCHAR)] = L'\0';
    if (!NT_SUCCESS(BuildParametersPath()))
        return STATUS_INVALID_PARAMETER;
    g_State.LoadCount = 1;
    LoadConfig();
    EnumerateParameters();
    DriverObject->DriverUnload = RegDemoUnload;
    return STATUS_SUCCESS;
}
```

## Variations and advanced use

**Read several values with one `RtlQueryRegistryValues` call.** One trip through the configuration manager, defaults built in, no manual size probing:

```c
static NTSTATUS QueryInterval(PWSTR ValueName, ULONG ValueType, PVOID ValueData,
                              ULONG ValueLength, PVOID Context, PVOID EntryContext)
{
    PULONG target = (PULONG)EntryContext;
    UNREFERENCED_PARAMETER(ValueName); UNREFERENCED_PARAMETER(Context);
    if (ValueType != REG_DWORD || ValueLength != sizeof(ULONG))
        return STATUS_SUCCESS;                  /* keep the default */
    *target = *(PULONG)ValueData;
    return STATUS_SUCCESS;
}

static VOID LoadWithQueryTable(VOID)
{
    static const ULONG defaultInterval = 30;
    RTL_QUERY_REGISTRY_TABLE table[3];
    NTSTATUS status;
    RtlZeroMemory(table, sizeof(table));
    table[0].QueryRoutine = QueryInterval;
    table[0].Name = L"IntervalSeconds";
    table[0].EntryContext = &g_State.Config.IntervalSeconds;
    table[0].DefaultType = REG_DWORD;           /* supplies the default when absent */
    table[0].DefaultData = (PVOID)&defaultInterval;
    table[0].DefaultLength = sizeof(ULONG);
    table[1].Flags = RTL_QUERY_REGISTRY_DIRECT | RTL_QUERY_REGISTRY_NOEXPAND;
    table[1].Name = L"LogPath";
    table[1].EntryContext = g_State.Config.LogPath;
    table[1].DefaultType = REG_SZ;
    table[1].DefaultData = L"\\SystemRoot\\Temp\\regdemo.log";
    table[1].DefaultLength = sizeof(L"\\SystemRoot\\Temp\\regdemo.log");
    status = RtlQueryRegistryValues(RTL_REGISTRY_ABSOLUTE, g_State.ParametersPath,
                                    table, NULL, NULL);
    if (!NT_SUCCESS(status))
        KdPrint(("RegDemo: query table failed 0x%08X\n", status));
}
```

- `RTL_QUERY_REGISTRY_DIRECT` with a `REG_SZ` needs an `EntryContext` large enough for `DefaultLength`; `RTL_QUERY_REGISTRY_REQUIRED` turns a missing value into a failing status, which is the right choice for a mandatory setting.
- **`ZwCreateKey` when the key may not exist yet:** pass `REG_OPTION_NON_VOLATILE` (`0`) and inspect `Disposition`. `REG_CREATED_NEW_KEY` is `1` and `REG_OPENED_EXISTING_KEY` is `2`; a create that "succeeds" on an existing key has not changed its security or class.
- **Device key instead of the service key:** `IoOpenDeviceRegistryKey(pdo, PLUGPLAY_REGKEY_DRIVER, KEY_READ | KEY_WRITE, &key)` opens the software key an INF addresses as `HKR`. It needs the PDO, so call it after `AddDevice`/`StartDevice`, not in `DriverEntry`.
- **Volatile counters:** create `...\Services\<name>\State` with `REG_OPTION_VOLATILE` (`0x1`) for values that change often; nothing is written to disk and nothing survives a reboot, which is exactly what a statistics counter wants.
- **Do not write on every change.** `ZwSetValueKey` dirties hive pages and `ZwFlushKey` waits for the log write; a driver that flushes per IOCTL turns a 20-microsecond operation into milliseconds and wears the disk. Flush at unload or on an explicit save command.

## Failure modes and diagnosis

| Symptom | Likely cause | Fix / first command |
|---|---|---|
| `STATUS_OBJECT_NAME_NOT_FOUND` (`0xC0000034`) on open | key absent: the INF never ran, or the path was built wrong | `reg query "HKLM\SYSTEM\CurrentControlSet\Services\RegDemo\Parameters"` |
| `0xC0000023` (`STATUS_BUFFER_TOO_SMALL`) on a value read | first call used a fixed buffer, or the value grew between calls | use `ResultLength`, retry once, then give up |
| `0xC000000D` (`STATUS_INVALID_PARAMETER`) | `OBJECT_ATTRIBUTES` not initialized, or an empty `UNICODE_STRING` name | check `InitializeObjectAttributes` and `RtlInitUnicodeString` |
| `0xC0000022` (`STATUS_ACCESS_DENIED`) | open issued in a client thread, or a protected key (`SAM`, `SECURITY`) | open at `DriverEntry` or from a system thread; narrow the mask |
| `STATUS_OBJECT_TYPE_MISMATCH` (`0xC0000024`) | value written as `REG_SZ` but read as `REG_DWORD` | `reg query <key> /v <name>` and compare the type |
| Value reads as all zeros or a truncated string | `REG_EXPAND_SZ` read raw, or `DataLength` treated as character count | divide by `sizeof(WCHAR)`; expand explicitly if needed |
| Enumeration returns one item then `STATUS_NO_MORE_ENTRIES` immediately | loop treats the warning status as failure, or index not incremented | compare against `STATUS_NO_MORE_ENTRIES` before `NT_SUCCESS` |
| Driver reads defaults although the value exists | 32-bit tool wrote under `WOW6432Node` | `reg query "HKLM\SOFTWARE\WOW6432Node\..."` |
| Handle leak: `!handle` count grows per load | early `return` between open and close | audit each path; close in reverse order |
| Bugcheck `0xA`/`0xD1` calling `ZwQueryValueKey` | called at `DISPATCH_LEVEL` | move the call to PASSIVE work, such as a work item |
| `ZwFlushKey` returns `STATUS_ACCESS_DENIED` | opened with `KEY_QUERY_VALUE` only | reopen with `KEY_SET_VALUE` |
| State lost after reboot | wrote to a volatile key, or never flushed | `REG_OPTION_NON_VOLATILE` plus `ZwFlushKey` |

## Safety, IRQL, and cleanup checklist

- All registry calls at PASSIVE_LEVEL; defer from DPC or completion routines with `IoQueueWorkItem` or a system thread.
- `OBJ_KERNEL_HANDLE` on every open, so no handle ever lands in a user process's table.
- Narrow access masks: `KEY_QUERY_VALUE` to read, `KEY_SET_VALUE` to write, never `KEY_ALL_ACCESS`.
- One `ZwClose` per successful open on every path, including every failure branch.
- Every `ExAllocatePool2` has a matching `ExFreePoolWithTag` with the same tag, including inside loops that `continue`.
- Validate `Type` and `DataLength` before reading `Data`; treat all registry content as attacker-controlled.
- Bound `REG_MULTI_SZ` parsing by `DataLength`; never call `wcslen` on a buffer that may not be terminated.
- Bound string copies with `RtlStringCchCopyW`/`RtlStringCchCatW` and size destination arrays with `ARRAYSIZE`.
- Copy `RegistryPath` in `DriverEntry`; do not cache the pointer.
- Add a version field to every `REG_BINARY` structure that may outlive the driver version that wrote it.
- Flush once at unload, not per write.
- Assume concurrent modification: another process can change or delete the value between the size probe and the read.

## Verification

1. Build: `msbuild RegDemo.vcxproj /p:Configuration=Release /p:Platform=x64`. Expected: no warnings at `/W4`, `RegDemo.sys` produced.
2. Create the configuration exactly as the INF would:
   `reg add "HKLM\SYSTEM\CurrentControlSet\Services\RegDemo\Parameters" /v IntervalSeconds /t REG_DWORD /d 45 /f`
   `reg add "...\Parameters" /v LogPath /t REG_SZ /d "\SystemRoot\Temp\a.log" /f`
   `reg add "...\Parameters" /v FilterList /t REG_MULTI_SZ /d "tcpip\0http" /f`
3. `sc create RegDemo type= kernel binPath= C:\Drivers\RegDemo.sys` then `sc start RegDemo`. DebugView must show `interval=45 ... filters=2`, proving the type-correct read of all three values.
4. The enumeration line per value proves `ZwEnumerateValueKey` and `DataOffset` handling: `value IntervalSeconds = 45`, `value LogPath type 1 size ...`.
5. `sc stop RegDemo`, then `reg query "...\Parameters" /v LastRun`. Expected: `REG_BINARY` with 24 bytes; `reg query ... /v LastRun /t REG_BINARY` shows `01 00 00 00` first (version 1).
6. Delete `Parameters` entirely and start again: expected `Parameters missing (0xC0000034), using defaults` and `interval=30`, with the driver still loading.
7. Set `IntervalSeconds` to `0` and to `999999`: expected the clamp log line and a value of `30`.
8. Verify no leak: `!handle 0 0` before and after ten load/unload cycles shows no growth in handle count, and `!poolused 2 RegD` shows no outstanding `meDR` allocations.
