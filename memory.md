# Memory based performance tests

- [Memory based performance tests](#memory-based-performance-tests)
  - [Allocating heap memory](#allocating-heap-memory)
    - [Results](#results)
      - [Unmanaged heap allocation (Marshal.AllocHGlobal())](#unmanaged-heap-allocation-marshalallochglobal)
      - [Managed heap allocation (new())](#managed-heap-allocation-new)
    - [Comparison](#comparison)
    - [Conclusion](#conclusion)
  - [Copying unmanged memory](#copying-unmanged-memory)
    - [Results](#results-1)
      - [Unsafe.CopyBlock()](#unsafecopyblock)
      - [Unsafe.CopyBlockUnaligned()](#unsafecopyblockunaligned)
    - [Comparison](#comparison-1)
    - [Conclusion](#conclusion-1)
  - [pmdbs2XNative.Security.MemoryProtection](#pmdbs2xnativesecuritymemoryprotection)
    - [Results](#results-2)
      - [EncryptedMemory (Custom code / cross platform)](#encryptedmemory-custom-code--cross-platform)
      - [Win32EncryptedMemory (CryptProtectMemory() / DPAPI)](#win32encryptedmemory-cryptprotectmemory--dpapi)
      - [Win32ProtectedMemory (VirtualProtect() / no real encryption)](#win32protectedmemory-virtualprotect--no-real-encryption)
    - [Comparison](#comparison-2)
    - [Conclusion](#conclusion-2)

## Allocating heap memory

Using unoptimized Debug build (no debugger attached) on .NET 5.0.

Test program code:

```csharp
//#define MANAGED

#if !MANAGED
#define UNMANAGED
#endif

using System;
using System.Diagnostics;
using System.Runtime.InteropServices;

const int iterations = 100_000_000;

Stopwatch stopwatch = new();
stopwatch.Start();
for (int _i = 0; _i < iterations; _i++)
{
#if UNMANAGED
    unsafe
    {
        IntPtr myBytes = Marshal.AllocHGlobal(65536);
        *((byte*)myBytes) = 1;
        Marshal.FreeHGlobal(myBytes);
    }
#endif
#if MANAGED
    byte[] myBytes = new byte[65536];
    myBytes[0] = 1;
#endif
}

stopwatch.Stop();
Console.WriteLine($"Completed {iterations} iterations in {stopwatch.Elapsed}.");
Console.WriteLine($"That's {stopwatch.Elapsed.TotalMilliseconds / iterations} ms/it on average.");
```

### Results

#### Unmanaged heap allocation (Marshal.AllocHGlobal())

```text
Completed 100000000 iterations in 00:00:19.4149530.
That's 0.00019414953 ms/it on average.
```

That's roughly `0.19 μs` per allocate / free cycle!

#### Managed heap allocation (new())

```text
Completed 100000000 iterations in 00:05:03.6484364.
That's 0.003036484364 ms/it on average.
```

### Comparison

| Type | Total time | ms / it | relative to slowest |
|---|---|---|---|
| managed | `00:05:03.6484364` | `0.003036484364` | `1` |
| unmanaged | `00:00:19.4149530` | `0.00019414953` | `15.64` |

### Conclusion

Unmanaged heap memory management is about `15.64` times **faster** than managed heap memory.

## Copying unmanged memory

Using unoptimized Debug build (no debugger attached) on .NET 5.0.

Test program code:

```csharp
using System;
using System.Diagnostics;
using System.Runtime.CompilerServices;
using System.Runtime.InteropServices;

unsafe
{
    const int length = 65536;
    const int iterations = 10_000_000;

    IntPtr hSource = Marshal.AllocHGlobal(length);
    IntPtr hTarget = Marshal.AllocHGlobal(length);

    byte* pSource = (byte*)hSource;
    byte* pTarget = (byte*)hTarget;

    Stopwatch stopwatch = new Stopwatch();
    stopwatch.Start();
    for (int i = 0; i < iterations; i++)
    {
        // Copy methods to test ...
    }
    stopwatch.Stop();
    Marshal.FreeHGlobal(hSource);
    Marshal.FreeHGlobal(hTarget);

    Console.WriteLine($"Completed {iterations} iterations in {stopwatch.Elapsed}.");
    Console.WriteLine($"That's {stopwatch.Elapsed.TotalMilliseconds / iterations} ms/it on average.");
}
```

### Results

#### Unsafe.CopyBlock()

```text
Completed 10000000 iterations in 00:00:25.2574299.
That's 0.00252574299 ms/it on average.
```

#### Unsafe.CopyBlockUnaligned()

```text
Completed 10000000 iterations in 00:00:25.2497422.
That's 0.00252497422 ms/it on average.
```

### Comparison

| Type | Total time | ms / it | relative to slowest |
|---|---|---|---|
| `Unsafe.CopyBlock()` | `00:00:25.2574299` | `0.00252574299` | `1` |
| `Unsafe.CopyBlockUnaligned()` | `00:00:25.2497422` | `0.00252497422` | `1` |

### Conclusion

`Unsafe.CopyBlock()` and `Unsafe.CopyBlockUnaligned()` are pretty much the same.

## pmdbs2XNative.Security.MemoryProtection

Using unoptimized Debug build (no debugger attached) on .NET 5.0.

Test program code:

```csharp
using pmdbs2XNative.Security.MemoryProtection.Native.Win32;
using pmdbs2XNative.Security.MemoryProtection.Universal;
using System;
using System.Diagnostics;
using System.Text;

const int iterations = 10_000;

// random /etc/passwd data ...
byte[] bytes = Encoding.UTF8.GetBytes("root:!:0:0::/:/usr/bin/ksh\ndaemon:!:1:1::/etc:\nbin:!:2:2::/bin:\nsys:!:3:3::/usr/sys:\nadm:!:4:4::/var/adm:\nuucp:!:5:5::/usr/lib/uucp: \nguest:!:100:100::/home/guest:\nnobody:!:4294967294:4294967294::/:\nlpd:!:9:4294967294::/:");
EncryptedMemory memory = new EncryptedMemory(256);
memory.Write(bytes, 0);

Stopwatch stopwatch = new Stopwatch();
stopwatch.Start();
for (int i = 0; i < iterations; i++)
{
    memory.Unprotect();
    memory.Protect();
}
stopwatch.Stop();

Console.WriteLine($"Completed {iterations} iterations in {stopwatch.Elapsed}.");
Console.WriteLine($"That's {stopwatch.Elapsed.TotalMilliseconds / iterations} ms/it on average.");
```

### Results

#### EncryptedMemory (Custom code / cross platform)

```text
Completed 10000 iterations in 00:00:09.8042608.
That's 0.98042608 ms/it on average.
```

#### Win32EncryptedMemory (CryptProtectMemory() / DPAPI)

```text
Completed 10000 iterations in 00:00:00.0598178.
That's 0.00598178 ms/it on average.
```

#### Win32ProtectedMemory (VirtualProtect() / no real encryption)

```text
Completed 10000 iterations in 00:00:00.0067782.
That's 0.00067782 ms/it on average.
```

### Comparison

| Type | Total time | ms / it | relative to slowest |
|---|---|---|---|
| `EncryptedMemory` | `00:00:09.8042608` | `0.98042608` | `1` |
| `Win32EncryptedMemory` | `00:00:00.0598178` | `0.00598178` | `163.9` |
| `Win32ProtectedMemory` | `00:00:00.0067782` | `0.00067782` | `1446.4` |

### Conclusion

The custom, pure C# `EncryptedMemory` memory protection with AES256-CBC-HMAC-BLAKE2b encrytion is the slowest. The native Windows APIs `CryptProtectMemory()` from DP-API is about `163.9` faster than the custom `EncryptedMemory` implementation but is specific to Windows while `VirtualProtect()` is `1446.4` times faster but doesn't provide "real" cryptographic protection although it may however prevent tools like CheatEngine from reading process memory easily.