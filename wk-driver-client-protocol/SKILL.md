---
name: wk-driver-client-protocol
description: Use when writing the user-mode side of a Windows driver protocol — building the Win32 device path, CreateFileW access and share flags, decoding CTL_CODE bit fields, choosing among METHOD_BUFFERED and the direct/neither methods, validating input and output sizes, packing and versioning shared structs, running overlapped DeviceIoControl with cancellation, and mapping GetLastError back to NTSTATUS.
---

# Driver client protocol: the user-mode half of the contract

A driver is only as good as the client that talks to it. The kernel side can validate every byte and
still be unusable if the client opens the wrong path, asks for the wrong rights, or reads a structure
the driver never finished writing. This skill defines the user-mode half of the **KlogSink** protocol:
the device path, handle flags, the four IOCTL codes and their numeric values, the buffer method
choice, size validation in both directions, a reusable C++ client, and the mapping from `GetLastError`
back to the `NTSTATUS` the driver returned.

The rule that prevents most disasters: **the client validates what it received, not just whether the
call succeeded.** `DeviceIoControl` returning `TRUE` means the driver completed the IRP successfully,
not that it wrote a structure you understand. Check `lpBytesReturned`, then the struct's `Size` and
`Version`, and only then read the fields.

## When to use

- You are writing a Win32 or C++ client for a driver that exposes `IRP_MJ_CREATE`, `IRP_MJ_WRITE` and
  IOCTLs.
- You must decide between `WriteFile`, `ReadFile`, and `DeviceIoControl` for a given operation.
- You are defining `CTL_CODE` values and need the bit layout and the reserved function ranges.
- A client works on x64 but returns garbage or `ERROR_INVALID_PARAMETER` from a 32-bit build.
- You need overlapped I/O, timeouts, or cancellation for an IOCTL that can block.
- You are mapping driver status codes to actionable client errors, or debugging a call that returns
  5, 87, 122, or 1306.

Do not use for the kernel side (that is `wk-driver-anatomy`), for IOCTL interface design without a
client (`wk-ioctl-design`), or for KMDF queue modeling.

## Prerequisites and related skills

- **Headers:** `<windows.h>`, `<winioctl.h>` (defines `CTL_CODE`), and the shared `Public.h`. That
  header must compile in both kernel and user mode, so it may use only `ULONG`, `WCHAR`, `LONGLONG`
  and the `CTL_CODE`/`METHOD_*`/`FILE_*` constants.
- **Toolchain:** any MSVC that targets Windows 10; the client links against `kernel32.lib` only.
- **Driver state:** the KlogSink service must be running with its link present, or every call fails
  with 2. Build and load it per `wk-driver-anatomy`.
- **Privilege:** opening a device whose DACL grants only administrators requires an elevated client.
- Siblings: `wk-driver-anatomy` (same protocol, kernel side — change both together),
  `wk-device-objects` (symbolic links and device security), `wk-ioctl-design` (interface design),
  `wk-driver-build-deploy` (installing the service).

## Mental model

A device handle is a file handle. `CreateFileW("\\\\.\\KlogSink", ...)` resolves the DOS device name
to the symbolic link, the object manager follows the link to `\Device\KlogSink`, and the I/O manager
creates a `FILE_OBJECT` whose `FsContext` the driver fills in during `IRP_MJ_CREATE`. Every later call
on that handle carries the same `FILE_OBJECT`, which is how the driver finds its per-handle state.

```
CreateFileW("\\.\KlogSink") --> \??\KlogSink --> \Device\KlogSink --> FILE_OBJECT
   WriteFile(record)      -> IRP_MJ_WRITE          buffered: SystemBuffer
   DeviceIoControl(code)  -> IRP_MJ_DEVICE_CONTROL Parameters.DeviceIoControl
   CloseHandle            -> IRP_MJ_CLOSE          FsContext freed
```

The protocol below is the one implemented in `wk-driver-anatomy`, byte for byte. If you change a
struct or a function code here, change it there in the same commit.

| Element | Value |
|---|---|
| Device / link / client path | `\Device\KlogSink`, `\??\KlogSink`, `\\.\KlogSink` |
| Device type | `0x8A10` (`KLOG_DEVICE_TYPE`, vendor range `0x8000`-`0xFFFF`) |
| Write payload | one `KLOG_RECORD`, exactly 160 bytes, buffered I/O |
| `KLOG_IOCTL_SET_CONFIG` | `0x8A10A004`, input `KLOG_CONFIG`, no output |
| `KLOG_IOCTL_GET_CONFIG` | `0x8A106008`, no input, output `KLOG_CONFIG` |
| `KLOG_IOCTL_GET_STATS` | `0x8A10600C`, no input, output `KLOG_STATS` |
| `KLOG_IOCTL_RESET` | `0x8A10A010`, no buffers |
| Version | `KLOG_PROTOCOL_VERSION` = 1 in every struct |

### CTL_CODE bit layout

`CTL_CODE(DeviceType, Function, Method, Access)` packs four fields into one 32-bit number:

```
31              16 15   14 13                    2 1  0
+-----------------+-------+-----------------------+-----+
|   DeviceType    | Access|       Function        |Method|
|     16 bits     | 2 bit |       12 bits         |2 bit|
+-----------------+-------+-----------------------+-----+
value = (DeviceType << 16) | (Access << 14) | (Function << 2) | Method
```

| Bits | Width | Field | Range and meaning | `SET_CONFIG` |
|---|---|---|---|---|
| 31-16 | 16 | DeviceType | `0x0000`-`0x7FFF` system, `0x8000`-`0xFFFF` vendor | `0x8A10` |
| 15-14 | 2 | Access | 0 any, 1 read, 2 write, 3 read+write | 2 `FILE_WRITE_DATA` |
| 13-2 | 12 | Function | `0x000`-`0x7FF` reserved, `0x800`-`0xFFF` vendor | `0x801` |
| 1-0 | 2 | Method | 0 buffered, 1 in-direct, 2 out-direct, 3 neither | 0 |

Working the arithmetic for each code in the protocol:

| Code | DeviceType | Access | Function | Method | Numeric value |
|---|---|---|---|---|---|
| `SET_CONFIG` | `0x8A10` | 2 | `0x801` | 0 | `0x8A10A004` |
| `GET_CONFIG` | `0x8A10` | 1 | `0x802` | 0 | `0x8A106008` |
| `GET_STATS` | `0x8A10` | 1 | `0x803` | 0 | `0x8A10600C` |
| `RESET` | `0x8A10` | 2 | `0x804` | 0 | `0x8A10A010` |

Never hard-code those numbers in the client; derive them from the macros so a protocol change moves
both sides at once. They appear here only so you can recognize them in a debugger or a log.

### The four buffer methods

The method field selects who copies, and the access field selects which handle rights the call needs.
They are independent: access is checked against the granted access on the handle, method decides how
the buffers reach the driver.

| Method | Value | Input buffer | Output buffer | Driver reads via | Use for |
|---|---|---|---|---|---|
| `METHOD_BUFFERED` | 0 | copied to a system buffer before dispatch | copied back after completion | `Irp->AssociatedIrp.SystemBuffer` | small fixed structs |
| `METHOD_IN_DIRECT` | 1 | locked as an MDL, read-only | not mapped | `Irp->MdlAddress` | large input |
| `METHOD_OUT_DIRECT` | 2 | not mapped | locked as an MDL, driver writes | `Irp->MdlAddress` | large output |
| `METHOD_NEITHER` | 3 | raw user pointer in `Type3InputBuffer` | raw user pointer in `UserBuffer` | pointers after probing | almost never |

`METHOD_IN_DIRECT` and `METHOD_OUT_DIRECT` name the buffer from the caller's perspective: in-direct
maps the input buffer for the driver to read; out-direct maps the output buffer for the driver to
write. Neither method maps the other side. For the four KlogSink codes buffered is correct: every
payload is a 24- or 32-byte struct, and a copy is cheaper than an MDL.

### Packing and versioning

Every struct in `Public.h` sits inside `#pragma pack(push, 8)`, leads with `Version` and `Size`, and
contains only fixed-width integers and `WCHAR` arrays. Three rules keep the two sides agreeing:

- **Assert the size on both sides.** The driver uses `C_ASSERT(sizeof(KLOG_RECORD) == 160)`; the
  client uses `static_assert(sizeof(KLOG_RECORD) == 160, "protocol drift")`. A mismatch must break the
  build, not corrupt a transfer.
- **Reject unknown majors, tolerate longer buffers.** Require `Version == KLOG_PROTOCOL_VERSION` and
  `Size == sizeof(KLOG_CONFIG)`; bytes beyond `sizeof` in the buffer are ignored, which is how a newer
  client can append fields without breaking an older driver.
- **Zero-initialize before filling.** `KLOG_CONFIG cfg{};` sets the reserved fields to 0; the driver
  rejects nonzero reserved fields as a cheap way to catch uninitialized memory.

## Key APIs and structures

| API | Purpose | Notes |
|---|---|---|
| `CreateFileW` | open the device | `OPEN_EXISTING`; returns `INVALID_HANDLE_VALUE` on failure |
| `DeviceIoControl` | send a control code | `lpBytesReturned` receives `IoStatus.Information` |
| `WriteFile` | send one record | the driver copies `nNumberOfBytesToWrite` bytes |
| `GetOverlappedResult` | collect an async result | only for handles opened `FILE_FLAG_OVERLAPPED` |
| `CancelIoEx` | cancel one pending operation | returns 1168 if the operation already finished |
| `CloseHandle` | drop the handle | triggers `IRP_MJ_CLOSE` and frees `FsContext` |
| `GetLastError` | Win32 error | the I/O manager maps `NTSTATUS` to it before returning |

| Structure | What matters |
|---|---|
| `KLOG_RECORD` | 160 bytes: `Version`, `Size`, `Severity` 0-4, `Flags`, driver-stamped IDs |
| `KLOG_CONFIG` | 24 bytes: `Capacity` 1-64, exactly one policy flag |
| `KLOG_STATS` | 32 bytes: counters the driver fills under its lock |
| `OVERLAPPED` | `hEvent` plus an offset; must stay alive until the call completes |

### Error mapping

The I/O manager converts the driver's `NTSTATUS` to a Win32 error before `DeviceIoControl` returns
`FALSE`. The mapping is consistent but not always intuitive, so match on the numeric code, not on the
message text, which is localized.

| `NTSTATUS` returned | Typical value | `GetLastError` seen |
|---|---|---|
| `STATUS_INVALID_DEVICE_REQUEST` | `0xC0000010` | 1 `ERROR_INVALID_FUNCTION` |
| `STATUS_ACCESS_DENIED` | `0xC0000022` | 5 `ERROR_ACCESS_DENIED` |
| `STATUS_INVALID_PARAMETER` | `0xC000000D` | 87 `ERROR_INVALID_PARAMETER` |
| `STATUS_BUFFER_TOO_SMALL` | `0xC0000023` | 122 `ERROR_INSUFFICIENT_BUFFER` |
| `STATUS_BUFFER_OVERFLOW` | `0x80000005` | 234 `ERROR_MORE_DATA` |
| `STATUS_REVISION_MISMATCH` | `0xC0000059` | 1306 `ERROR_REVISION_MISMATCH` |
| `STATUS_INVALID_BUFFER_SIZE` | `0xC0000206` | 87 on most systems |
| `STATUS_OBJECT_NAME_NOT_FOUND` | `0xC0000034` | 2 `ERROR_FILE_NOT_FOUND` |
| `STATUS_DELETE_PENDING` | `0xC0000056` | 303 `ERROR_DELETE_PENDING` |

## Step-by-step

1. **Include the shared header, never a copy:** a duplicated struct definition is the top cause of 87
   on one side and success on the other. *Check:* `static_assert(sizeof(KLOG_RECORD) == 160)` compiles.
2. **Open with the minimum rights:** `GENERIC_READ | GENERIC_WRITE`, share 0 for exclusive use,
   `FILE_SHARE_READ | FILE_SHARE_WRITE` for a second client. *Check:* a second open with share 0 fails
   with 32.
3. **Pick the method before writing the call:** buffered for structs, direct for bulk, never neither.
   *Check:* the `CTL_CODE` names match the driver's switch cases.
4. **Send exactly `sizeof(struct)` bytes and declare the output size you can handle.** *Check:* the
   driver sees `InputBufferLength == 24` for a config write.
5. **Validate `*lpBytesReturned`, `Version`, and `Size` before reading any field.** *Check:* a short
   output buffer returns 122, not silent garbage.
6. **Capture `GetLastError()` immediately on `FALSE`** and return it instead of a bool. *Check:* the
   code survives to the message you print.
7. **Use `FILE_FLAG_OVERLAPPED` and a manual-reset event for blocking commands.** *Check:* the call
   returns 997 under load and the wait completes it.
8. **Always `CloseHandle`;** a leaked handle keeps `FsContext` alive and makes unload refuse. *Check:*
   `!devobj \Device\KlogSink` returns to `RefCount: 1`.
9. **Test the rejections on purpose:** stale version, unknown function, short buffer. *Check:* 1306, 1,
   122.

## Complete example

Four files: the shared header, the client class, its implementation, and a `main` that exercises every
command plus three deliberate failures.

### Public.h

```c
/* Public.h - the contract between KlogSink and its clients.
   Include after <ntddk.h> in kernel code and after <windows.h> in user code. */
#pragma once

#define KLOG_PROTOCOL_VERSION  1
#define KLOG_DEVICE_TYPE       0x8A10                /* vendor 0x8000-0xFFFF */
#define KLOG_DEVICE_NAME       L"\\Device\\KlogSink"
#define KLOG_SYMLINK_NAME      L"\\??\\KlogSink"
#define KLOG_DOS_NAME          L"\\\\.\\KlogSink"    /* user mode only */

#define KLOG_TEXT_CHARS        64
#define KLOG_MAX_RECORDS       64
#define KLOG_MAX_SEVERITY      4
#define KLOG_CFG_DROP_ON_FULL      0x00000001
#define KLOG_CFG_OVERWRITE_OLDEST  0x00000002
#define KLOG_CFG_VALID_FLAGS       (KLOG_CFG_DROP_ON_FULL | KLOG_CFG_OVERWRITE_OLDEST)
#define KLOG_RECORD_URGENT         0x00000001
#define KLOG_RECORD_VALID_FLAGS    (KLOG_RECORD_URGENT)

#define KLOG_IOCTL_SET_CONFIG  CTL_CODE(KLOG_DEVICE_TYPE, 0x801, METHOD_BUFFERED, FILE_WRITE_DATA)
#define KLOG_IOCTL_GET_CONFIG  CTL_CODE(KLOG_DEVICE_TYPE, 0x802, METHOD_BUFFERED, FILE_READ_DATA)
#define KLOG_IOCTL_GET_STATS   CTL_CODE(KLOG_DEVICE_TYPE, 0x803, METHOD_BUFFERED, FILE_READ_DATA)
#define KLOG_IOCTL_RESET       CTL_CODE(KLOG_DEVICE_TYPE, 0x804, METHOD_BUFFERED, FILE_WRITE_DATA)

#pragma pack(push, 8)

typedef struct _KLOG_RECORD {
    ULONG    Version;                 /* KLOG_PROTOCOL_VERSION                 */
    ULONG    Size;                    /* sizeof(KLOG_RECORD), exactly 160      */
    ULONG    Severity;                /* 0..KLOG_MAX_SEVERITY                  */
    ULONG    Flags;                   /* KLOG_RECORD_*                         */
    ULONG    ProcessId;               /* overwritten by the driver             */
    ULONG    ThreadId;                /* overwritten by the driver             */
    LONGLONG Timestamp;               /* overwritten by the driver, 100 ns     */
    WCHAR    Text[KLOG_TEXT_CHARS];   /* NUL-terminated on the way in          */
} KLOG_RECORD, *PKLOG_RECORD;

typedef struct _KLOG_CONFIG {
    ULONG Version;                    /* KLOG_PROTOCOL_VERSION                 */
    ULONG Size;                       /* sizeof(KLOG_CONFIG), exactly 24       */
    ULONG Capacity;                   /* 1..KLOG_MAX_RECORDS                   */
    ULONG Flags;                      /* exactly one KLOG_CFG_* bit            */
    ULONG Reserved;                   /* must be 0                             */
    ULONG Reserved2;                  /* must be 0                             */
} KLOG_CONFIG, *PKLOG_CONFIG;

typedef struct _KLOG_STATS {
    ULONG Version;
    ULONG Size;                       /* sizeof(KLOG_STATS), exactly 32        */
    ULONG Accepted;                   /* records copied into the ring          */
    ULONG Dropped;                    /* records the policy refused            */
    ULONG Capacity;                   /* current slot count                    */
    ULONG Records;                    /* slots currently occupied              */
    ULONG OpenHandles;                /* live handles on this device           */
    ULONG MaxRecordBytes;             /* sizeof(KLOG_RECORD)                   */
} KLOG_STATS, *PKLOG_STATS;

#pragma pack(pop)
```

### KlogClient.h

```cpp
// KlogClient.h - reusable user-mode client for the KlogSink protocol.
#pragma once

#include <windows.h>
#include <winioctl.h>
#include "Public.h"

static_assert(sizeof(KLOG_RECORD) == 160, "protocol drift");   // both sides, same header

class KlogClient {
public:
    KlogClient() noexcept : m_handle(INVALID_HANDLE_VALUE), m_overlapped(false) {}
    ~KlogClient() { Close(); }

    KlogClient(const KlogClient&) = delete;
    KlogClient& operator=(const KlogClient&) = delete;

    // Returns a Win32 error code; ERROR_SUCCESS on success.
    DWORD Open(const wchar_t* devicePath = KLOG_DOS_NAME,
               DWORD shareMode = 0,
               bool overlapped = false);
    void  Close() noexcept;

    bool   IsOpen() const noexcept { return m_handle != INVALID_HANDLE_VALUE; }
    bool   IsOverlapped() const noexcept { return m_overlapped; }
    HANDLE Handle() const noexcept { return m_handle; }

    // One record per call: the driver accepts exactly sizeof(KLOG_RECORD) bytes.
    DWORD WriteRecord(ULONG severity, const wchar_t* text, bool urgent,
                      DWORD* bytesWritten = nullptr);
    DWORD WriteRaw(const KLOG_RECORD& record, DWORD* bytesWritten = nullptr);

    DWORD GetConfig(KLOG_CONFIG* config);
    DWORD SetConfig(const KLOG_CONFIG& config);
    DWORD GetStats(KLOG_STATS* stats);
    DWORD Reset();
    DWORD GetStatsOverlapped(KLOG_STATS* stats, DWORD timeoutMs);

    // Escape hatch for probing the driver's validation: unknown codes, short buffers.
    DWORD RawIoctl(DWORD code, void* input, DWORD inputSize,
                   void* output, DWORD outputSize, DWORD* bytesReturned = nullptr);

private:
    DWORD Ioctl(DWORD code, void* input, DWORD inputSize,
                void* output, DWORD outputSize, DWORD* bytesReturned);

    HANDLE m_handle;
    bool   m_overlapped;
};
```

### KlogClient.cpp

```cpp
#include "KlogClient.h"
#include <cwchar>

DWORD KlogClient::Open(const wchar_t* devicePath, DWORD shareMode, bool overlapped)
{
    if (IsOpen()) {
        return ERROR_ALREADY_EXISTS;        // close first; the old handle stays usable
    }
    DWORD flags = FILE_ATTRIBUTE_NORMAL;
    if (overlapped) {
        flags |= FILE_FLAG_OVERLAPPED;
    }
    HANDLE handle = ::CreateFileW(devicePath, GENERIC_READ | GENERIC_WRITE, shareMode,
                                  nullptr, OPEN_EXISTING, flags, nullptr);
    if (handle == INVALID_HANDLE_VALUE) {
        return ::GetLastError();
    }
    m_handle = handle;
    m_overlapped = overlapped;
    return ERROR_SUCCESS;
}

void KlogClient::Close() noexcept
{
    if (IsOpen()) {
        ::CloseHandle(m_handle);            // sends IRP_MJ_CLOSE, frees the FsContext
        m_handle = INVALID_HANDLE_VALUE;
        m_overlapped = false;
    }
}

DWORD KlogClient::Ioctl(DWORD code, void* input, DWORD inputSize,
                        void* output, DWORD outputSize, DWORD* bytesReturned)
{
    if (!IsOpen()) {
        return ERROR_INVALID_HANDLE;
    }
    DWORD local = 0;
    if (bytesReturned == nullptr) {
        bytesReturned = &local;
    }
    *bytesReturned = 0;
    if (!::DeviceIoControl(m_handle, code, input, inputSize,
                           output, outputSize, bytesReturned, nullptr)) {
        return ::GetLastError();
    }
    return ERROR_SUCCESS;
}

DWORD KlogClient::WriteRecord(ULONG severity, const wchar_t* text, bool urgent,
                              DWORD* bytesWritten)
{
    KLOG_RECORD record{};                   // zero first: the driver rejects junk flags
    record.Version = KLOG_PROTOCOL_VERSION;
    record.Size = static_cast<ULONG>(sizeof(KLOG_RECORD));
    record.Severity = severity;
    record.Flags = urgent ? KLOG_RECORD_URGENT : 0u;
    if (text != nullptr) {
        ::wcsncpy_s(record.Text, KLOG_TEXT_CHARS, text, _TRUNCATE);
        record.Text[KLOG_TEXT_CHARS - 1] = L'\0';
    }
    return WriteRaw(record, bytesWritten);
}

DWORD KlogClient::WriteRaw(const KLOG_RECORD& record, DWORD* bytesWritten)
{
    if (!IsOpen()) {
        return ERROR_INVALID_HANDLE;
    }
    if (m_overlapped) {
        return ERROR_INVALID_PARAMETER;     // this helper is synchronous by design
    }
    DWORD written = 0;
    if (!::WriteFile(m_handle, &record, static_cast<DWORD>(sizeof(record)),
                     &written, nullptr)) {
        return ::GetLastError();
    }
    if (written != sizeof(record)) {
        return ERROR_WRITE_FAULT;           // short write: the driver disagrees
    }
    if (bytesWritten != nullptr) {
        *bytesWritten = written;
    }
    return ERROR_SUCCESS;
}

DWORD KlogClient::GetConfig(KLOG_CONFIG* config)
{
    if (config == nullptr) {
        return ERROR_INVALID_PARAMETER;
    }
    DWORD returned = 0;
    DWORD error = Ioctl(KLOG_IOCTL_GET_CONFIG, nullptr, 0, config,
                        static_cast<DWORD>(sizeof(*config)), &returned);
    if (error != ERROR_SUCCESS) {
        return error;
    }
    if (returned != sizeof(*config) || config->Size != sizeof(*config)) {
        return ERROR_INVALID_DATA;          // the driver wrote less than the contract
    }
    if (config->Version != KLOG_PROTOCOL_VERSION) {
        return ERROR_REVISION_MISMATCH;
    }
    return ERROR_SUCCESS;
}

DWORD KlogClient::SetConfig(const KLOG_CONFIG& config)
{
    KLOG_CONFIG copy = config;              // never mutate the caller's object
    copy.Version = KLOG_PROTOCOL_VERSION;
    copy.Size = static_cast<ULONG>(sizeof(KLOG_CONFIG));
    return Ioctl(KLOG_IOCTL_SET_CONFIG, &copy, static_cast<DWORD>(sizeof(copy)),
                 nullptr, 0, nullptr);
}

DWORD KlogClient::GetStats(KLOG_STATS* stats)
{
    if (stats == nullptr) {
        return ERROR_INVALID_PARAMETER;
    }
    DWORD returned = 0;
    DWORD error = Ioctl(KLOG_IOCTL_GET_STATS, nullptr, 0, stats,
                        static_cast<DWORD>(sizeof(*stats)), &returned);
    if (error != ERROR_SUCCESS) {
        return error;
    }
    if (returned != sizeof(*stats) || stats->Size != sizeof(*stats)) {
        return ERROR_INVALID_DATA;
    }
    return ERROR_SUCCESS;
}

DWORD KlogClient::Reset()
{
    return Ioctl(KLOG_IOCTL_RESET, nullptr, 0, nullptr, 0, nullptr);
}

DWORD KlogClient::RawIoctl(DWORD code, void* input, DWORD inputSize,
                           void* output, DWORD outputSize, DWORD* bytesReturned)
{
    return Ioctl(code, input, inputSize, output, outputSize, bytesReturned);
}

DWORD KlogClient::GetStatsOverlapped(KLOG_STATS* stats, DWORD timeoutMs)
{
    if (stats == nullptr) {
        return ERROR_INVALID_PARAMETER;
    }
    if (!IsOpen() || !m_overlapped) {
        return ERROR_INVALID_PARAMETER;     // needs a FILE_FLAG_OVERLAPPED handle
    }

    OVERLAPPED ov{};
    ov.hEvent = ::CreateEventW(nullptr, TRUE, FALSE, nullptr);   // manual reset
    if (ov.hEvent == nullptr) {
        return ::GetLastError();
    }

    DWORD returned = 0;
    DWORD error = ERROR_SUCCESS;
    if (!::DeviceIoControl(m_handle, KLOG_IOCTL_GET_STATS, nullptr, 0, stats,
                           static_cast<DWORD>(sizeof(*stats)), &returned, &ov)) {
        error = ::GetLastError();
        if (error == ERROR_IO_PENDING) {
            DWORD wait = ::WaitForSingleObject(ov.hEvent, timeoutMs);
            if (wait == WAIT_OBJECT_0) {
                error = ERROR_SUCCESS;
            } else if (wait == WAIT_TIMEOUT) {
                ::CancelIoEx(m_handle, &ov);
                ::WaitForSingleObject(ov.hEvent, INFINITE);   // the IRP must come back
                error = ERROR_TIMEOUT;
            } else {
                error = ::GetLastError();
            }
        }
    }

    if (error == ERROR_SUCCESS) {
        if (!::GetOverlappedResult(m_handle, &ov, &returned, FALSE)) {
            error = ::GetLastError();
        } else if (returned != sizeof(*stats)) {
            error = ERROR_INVALID_DATA;
        }
    }
    ::CloseHandle(ov.hEvent);
    return error;
}
```

### main.cpp

```cpp
#include <windows.h>
#include <cstdio>
#include "KlogClient.h"

namespace {

void Show(const char* step, DWORD error)
{
    if (error == ERROR_SUCCESS) {
        std::printf("[ ok ] %s\n", step);
    } else {
        std::printf("[fail] %s: %lu\n", step, error);
    }
}

}  // namespace

int main()
{
    KlogClient client;
    DWORD error = client.Open(KLOG_DOS_NAME, FILE_SHARE_READ | FILE_SHARE_WRITE);
    if (error != ERROR_SUCCESS) {
        std::printf("Open failed: %lu (is the KlogSink service started?)\n", error);
        return 1;
    }

    KLOG_CONFIG config{};
    error = client.GetConfig(&config);
    Show("GetConfig", error);
    std::printf("       capacity=%lu flags=0x%08lX version=%lu\n",
                config.Capacity, config.Flags, config.Version);

    config.Capacity = 4;
    config.Flags = KLOG_CFG_OVERWRITE_OLDEST;
    Show("SetConfig(capacity=4, overwrite oldest)", client.SetConfig(config));

    DWORD written = 0;
    Show("WriteRecord(hello)", client.WriteRecord(1, L"hello kernel", false, &written));
    Show("WriteRecord(urgent)", client.WriteRecord(2, L"second record", true, &written));

    KLOG_STATS stats{};
    Show("GetStats", client.GetStats(&stats));
    std::printf("       accepted=%lu dropped=%lu records=%lu handles=%lu\n",
                stats.Accepted, stats.Dropped, stats.Records, stats.OpenHandles);

    for (int i = 0; i < 6; ++i) {
        error = client.WriteRecord(0, L"filler", false, nullptr);
    }
    Show("WriteRecord x6 into a 4-slot ring", error);
    Show("GetStats after overflow", client.GetStats(&stats));
    std::printf("       accepted=%lu dropped=%lu records=%lu\n",
                stats.Accepted, stats.Dropped, stats.Records);

    // Deliberate failures: each one must come back as a specific Win32 error.
    KLOG_RECORD stale{};
    stale.Version = KLOG_PROTOCOL_VERSION + 1;
    stale.Size = static_cast<ULONG>(sizeof(KLOG_RECORD));
    error = client.WriteRaw(stale, nullptr);
    std::printf("[probe] stale version -> %lu (1306 expected)\n", error);

    error = client.RawIoctl(CTL_CODE(KLOG_DEVICE_TYPE, 0x8FF, METHOD_BUFFERED, FILE_ANY_ACCESS),
                            nullptr, 0, nullptr, 0, nullptr);
    std::printf("[probe] unknown function -> %lu (1 expected)\n", error);

    KLOG_CONFIG small{};
    DWORD returned = 0;
    error = client.RawIoctl(KLOG_IOCTL_GET_CONFIG, nullptr, 0, &small, 8, &returned);
    std::printf("[probe] 8-byte output -> %lu (122 expected)\n", error);

    Show("Reset", client.Reset());
    Show("GetStats after reset", client.GetStats(&stats));
    std::printf("       accepted=%lu dropped=%lu records=%lu\n",
                stats.Accepted, stats.Dropped, stats.Records);
    KlogClient async;
    error = async.Open(KLOG_DOS_NAME, FILE_SHARE_READ | FILE_SHARE_WRITE, true);
    if (error == ERROR_SUCCESS) {
        Show("overlapped GetStats", async.GetStatsOverlapped(&stats, 5000));
        std::printf("       handles=%lu (two are open)\n", stats.OpenHandles);
        async.Close();
    } else {
        std::printf("[fail] overlapped open: %lu\n", error);
    }
    client.Close();
    std::puts("done");
    return 0;
}
```

Expected output on a healthy system, abbreviated:

```text
[ ok ] GetConfig
       capacity=8 flags=0x00000001 version=1
[ ok ] SetConfig(capacity=4, overwrite oldest)
[ ok ] WriteRecord(hello)
[ ok ] WriteRecord(urgent)
[ ok ] GetStats
       accepted=2 dropped=0 records=2 handles=1
[ ok ] WriteRecord x6 into a 4-slot ring
       accepted=8 dropped=4 records=4
[probe] stale version -> 1306 (1306 expected)
[probe] unknown function -> 1 (1 expected)
[probe] 8-byte output -> 122 (122 expected)
```

## Variations and advanced use

- **Bulk payloads.** For a 1 MB record, buffered I/O copies twice (user to system buffer, then into
  your ring) and the copy is charged to non-paged pool. Switch the code to `METHOD_OUT_DIRECT` for
  driver-to-client and `METHOD_IN_DIRECT` for client-to-driver, and map the MDL on the kernel side.
- **A steady event stream.** Open with `FILE_FLAG_OVERLAPPED`, keep several IOCTLs pending, and use one
  event per request. Completion order tells you which reply arrived; `GetOverlappedResult` must be
  called per request.
- **Cancellation.** `CancelIoEx(handle, &overlapped)` cancels one IRP and returns 1168
  (`ERROR_NOT_FOUND`) if it already completed. On process exit, `CancelIoEx(handle, nullptr)` cancels
  everything on the handle, which is what lets the driver unload promptly.
- **Language bindings.** The same protocol from C# uses `SafeFileHandle`, `DeviceIoControl` P/Invoke
  with `ref` structs, and `[StructLayout(LayoutKind.Sequential, Pack = 8)]`; the pack value and field
  order must match `Public.h` exactly.

## IOCTL design checklist

- [ ] The device type is in the vendor range (`0x8000`-`0xFFFF`) and matches `IoCreateDevice`.
- [ ] Function codes start at `0x800` and are never reused with different semantics.
- [ ] Every code has a distinct access mask; nothing uses `FILE_ANY_ACCESS` unless it truly needs none.
- [ ] The method matches the data: buffered for structs, in/out-direct for bulk, never neither.
- [ ] Every struct carries `Version` and `Size`, uses fixed-width fields, and contains no pointers.
- [ ] The client sends `sizeof(struct)`; the driver checks the length before the first dereference.
- [ ] The driver sets `Information` to the bytes produced, and the client checks it.
- [ ] Output buffers are declared at full struct size; a short buffer returns 122, not a truncation.
- [ ] Unknown function codes return `STATUS_INVALID_DEVICE_REQUEST`, not success.
- [ ] The access mask matches the DACL: a read IOCTL on a write-only handle must fail with 5.
- [ ] Oversized input is rejected, or its trailing bytes are ignored — and the header says which.
- [ ] Every rejection path has a test in the client, with the expected `GetLastError` written down.

## Failure modes and diagnosis

| `GetLastError` | Meaning here | Likely cause | Fix |
|---|---|---|---|
| 2 | `ERROR_FILE_NOT_FOUND` | service not started, or no `\??\KlogSink` link | `sc query klgsink`, then check `\??\` in WinObj |
| 5 | `ERROR_ACCESS_DENIED` | DACL excludes the caller, or the access mask exceeds the handle rights | run elevated, or open with the rights the code requires |
| 6 | `ERROR_INVALID_HANDLE` | handle already closed, or `INVALID_HANDLE_VALUE` passed | track the handle in one object; close once |
| 1 | `ERROR_INVALID_FUNCTION` | unknown IOCTL, or an unhandled `IRP_MJ_*` (for example `ReadFile`) | match the code against the four macros |
| 87 | `ERROR_INVALID_PARAMETER` | wrong `Size`, `Capacity` outside 1-64, nonzero reserved field, unknown flag | dump the struct before sending it |
| 122 | `ERROR_INSUFFICIENT_BUFFER` | output buffer smaller than the struct the driver writes | pass `sizeof(KLOG_STATS)` |
| 1306 | `ERROR_REVISION_MISMATCH` | client and driver built from different `Public.h` | rebuild both, then reinstall the service |
| 32 | `ERROR_SHARING_VIOLATION` | second open with share mode 0 | pass `FILE_SHARE_READ \| FILE_SHARE_WRITE` |
| 997 | `ERROR_IO_PENDING` | overlapped request queued | wait on `ov.hEvent`, then `GetOverlappedResult` |
| 995 | `ERROR_OPERATION_ABORTED` | the IRP was cancelled | expected after `CancelIoEx`; clean up and retry |
| 1168 | `ERROR_NOT_FOUND` | `CancelIoEx` found nothing to cancel | the request already completed; collect its result |

| Symptom | Likely cause | First command |
|---|---|---|
| Client hangs in `DeviceIoControl` | synchronous handle with a driver that never completes the IRP | break in with the debugger, `!irpfind`, check the driver's queue |
| Works on x64, fails on x86 | a struct with a pointer, `size_t`, or `bool` | compare `sizeof` on both builds |
| Data looks shifted by 4 bytes | packing mismatch or a missing `Size` check | print `sizeof` and the first 16 bytes as hex |
| Driver will not stop | a handle is still open | `!devobj \Device\KlogSink` shows the reference count |

## Safety, IRQL, and cleanup checklist

- [ ] Every `CreateFileW` result is compared with `INVALID_HANDLE_VALUE`, and `GetLastError()` is
      captured immediately.
- [ ] The handle is owned by one object and closed exactly once, including on early returns.
- [ ] Handles are opened without inheritance, so a child process cannot keep the device open.
- [ ] Input structs are zero-initialized, then every field is set; reserved fields stay 0.
- [ ] `Version` and `Size` are set by the client, not copied from an uninitialized stack object.
- [ ] Output buffers are validated (`bytesReturned`, `Size`, `Version`) before any field is read, and
      every size passed to `DeviceIoControl` is `sizeof`, never a guess.
- [ ] `OVERLAPPED` and its event outlive the call; the event is closed only after completion.
- [ ] The client never writes a kernel address or a pointer into a shared struct.
- [ ] A privileged client is not used as a workaround for a DACL that should grant a service account.

## Verification

```text
cl /W4 /EHsc /std:c++17 main.cpp KlogClient.cpp /link kernel32.lib
main.exe                        -> the [ ok ] and [probe] lines shown above
sc stop klgsink
main.exe                        -> Open failed: 2
sc start klgsink
main.exe                        -> all [ ok ] lines again
kd> !devobj \Device\KlogSink    -> No matching devices found   (after sc stop)
```

A rising `accepted` count with `records` stuck at the ring capacity is the drop policy working.
