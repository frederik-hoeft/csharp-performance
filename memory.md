# Memory based performance tests

- [Memory based performance tests](#memory-based-performance-tests)
  - [Copying unmanged memory](#copying-unmanged-memory)
    - [Results](#results)
      - [Unsafe.CopyBlock()](#unsafecopyblock)
      - [Unsafe.CopyBlockUnaligned()](#unsafecopyblockunaligned)
  - [pmdbs2XNative.Security.MemoryProtection](#pmdbs2xnativesecuritymemoryprotection)
    - [Results](#results-1)
      - [EncryptedMemory (Cross platform)](#encryptedmemory-cross-platform)
      - [Win32EncryptedMemory (DPAPI)](#win32encryptedmemory-dpapi)
      - [Win32ProtectedMemory (VirtualProtect)](#win32protectedmemory-virtualprotect)

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

---

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

---

### Results

#### EncryptedMemory (Cross platform)

```text
Completed 10000 iterations in 00:00:09.8042608.
That's 0.98042608 ms/it on average.
```

#### Win32EncryptedMemory (DPAPI)

```text
Completed 10000 iterations in 00:00:00.0598178.
That's 0.00598178 ms/it on average.
```

#### Win32ProtectedMemory (VirtualProtect)

```text
Completed 10000 iterations in 00:00:00.0067782.
That's 0.00067782 ms/it on average.
```
