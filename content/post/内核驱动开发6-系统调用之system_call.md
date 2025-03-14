---
title: 内核驱动开发6 系统调用之system_call
published: 2024-11-22T22:06:26+08:00
summary: "过LAT保护"
tags: [内核,LAT]
categories: '内核驱动'
draft: false 
lang: ''
---
# 第六部分 系统调用

## 系统调用流程

概览：有些函数，在三环调用，本质上是从三环进行system call，进入零环实现，例如CreateFile，openprocess，
还有些函数并不会有system call这个步骤，如memcopy等

```Markdown 系统调用流程图
+-------------------------------+
|           应用程序             |  
+---------------+---------------+
                |
                v
+---------------+---------------+
|           3Ring API           |  
+---------------+---------------+
                |                  
                v                 <----- system call
+---------------+---------------+
|      0Ring中对应功能的函数      |  
+---------------+---------------+
                |
                v                 <----- system retl
+---------------+---------------+
|          返回到3Ring           | 
+---------------+---------------+
                |
                v
+---------------+---------------+
|         返回到应用程序         |
+---------------+---------------+

```

实践理解：

1. 使用xdbg 打开微信
2. 在调试器中选择[符号]栏
3. 搜索函数openprocess，会得到两个版本，一个是ZW版本，一个是NT版本，通过汇编发现都是调用同样的内容，说明在三环两个函数是同样的内容
![图 0](../images/da5992df0218c42f251ffd8654f539d485de3253355873f6dfdbddb25b289af7.png)  
![图 1](../images/fe0764c8bf774fc9b81fca4ab444dbbcbcc059afa55c4c1c4a4dca7befb0ecb2.png)  
4. 读汇编，test byte ptr ds:[7FFE0308],1 是在比较7FFE0308这个结构中的某个位是否为1
5. 使用IDA静态调试，发现静态状态下也是test byte ptr ds:[7FFE0308]，说明他没有被重定位，导致没有被重定位的原因：有一个结构叫_KUSER_SHARED_DATA，保存着零环合三环公用的一些结构，零环和三环都能去访问这个结构体,第308位偏移是**SystemCall**
系统时间信息:如 InterruptTime、SystemTime、TimeZoneBias 等字段存储系统时间相关信息。
系统配置信息:如 NtSystemRoot、NtBuildNumber、NtProductType 等字段存储系统版本和配置信息。
处理器特性:ProcessorFeatures 数组存储 CPU 支持的功能特性
安全和调试信息:如 KdDebuggerEnabled、SafeBootMode、DbgErrorPortPresent 等字段用于系统调试和安全相关功能。
性能计数器:如 QpcFrequency 字段存储高精度计数器频率。
系统调用号:SystemCall 字段存储当前系统调用号。
其他系统级共享数据:如 RNGSeedVersion(随机数生成器种子版本)、SuiteMask(系统版本特性)等。


好的,我来详细解释一下x64和x86的函数调用约定,并比较它们的区别:

x86 (32位) 函数调用约定:

1. cdecl (C声明):
   - 参数从右向左压入栈
   - 调用者负责清理栈
   - 返回值在EAX寄存器中

2. stdcall (标准调用):
   - 参数从右向左压入栈
   - 被调用函数负责清理栈
   - 返回值在EAX寄存器中

3. fastcall (快速调用):
   - 前两个参数使用ECX和EDX寄存器
   - 其余参数从右向左压入栈
   - 被调用函数负责清理栈
   - 返回值在EAX寄存器中

x64 (64位) 函数调用约定:

1. Microsoft x64 调用约定:
   - 前四个整数/指针参数使用RCX, RDX, R8, R9寄存器
   - 前四个浮点参数使用XMM0-XMM3寄存器
   - 其余参数从右向左压入栈
   - 调用者负责为寄存器参数在栈上分配32字节的影子空间
   - 返回值在RAX寄存器中(浮点值在XMM0中)

2. System V AMD64 ABI (Linux, macOS等):
   - 整数/指针参数使用RDI, RSI, RDX, RCX, R8, R9寄存器
   - 浮点参数使用XMM0-XMM7寄存器
   - 其余参数从右向左压入栈
   - 返回值在RAX寄存器中(浮点值在XMM0中)

主要区别:

1. 参数传递:
   - x86主要依赖栈传递参数
   - x64更多地使用寄存器传递参数,提高了效率

2. 寄存器使用:
   - x64有更多的通用寄存器可用(R8-R15)

3. 返回值处理:
   - x64可以更有效地处理较大的返回值(如通过寄存器对)

4. 栈对齐:
   - x64要求16字节对齐,x86通常是4字节对齐

5. 影子空间:
   - x64 Windows调用约定特有的32字节影子空间

6. 调用者清理栈:
   - x64统一由调用者清理栈,简化了异常处理

7. 指针大小:
   - x64使用64位指针,而x86使用32位指针

8. 优化机会:
   - x64由于有更多寄存器和更宽的数据路径,提供了更多优化机会

这些差异反映了64位架构的优势,如更大的地址空间、更多的寄存器和更高的性能潜力。理解这些调用约定对于底层编程、逆向工程和性能优化非常重要。
    ```
    3: kd> dt _KUSER_SHARED_DATA
    ntdll!_KUSER_SHARED_DATA
    +0x000 TickCountLowDeprecated : Uint4B
    +0x004 TickCountMultiplier : Uint4B
    +0x008 InterruptTime    : _KSYSTEM_TIME
    +0x014 SystemTime       : _KSYSTEM_TIME
    +0x020 TimeZoneBias     : _KSYSTEM_TIME
    +0x02c ImageNumberLow   : Uint2B
    +0x02e ImageNumberHigh  : Uint2B
    +0x030 NtSystemRoot     : [260] Wchar
    +0x238 MaxStackTraceDepth : Uint4B
    +0x23c CryptoExponent   : Uint4B
    +0x240 TimeZoneId       : Uint4B
    +0x244 LargePageMinimum : Uint4B
    +0x248 AitSamplingValue : Uint4B
    +0x24c AppCompatFlag    : Uint4B
    +0x250 RNGSeedVersion   : Uint8B
    +0x258 GlobalValidationRunlevel : Uint4B
    +0x25c TimeZoneBiasStamp : Int4B
    +0x260 NtBuildNumber    : Uint4B
    +0x264 NtProductType    : _NT_PRODUCT_TYPE
    +0x268 ProductTypeIsValid : UChar
    +0x269 Reserved0        : [1] UChar
    +0x26a NativeProcessorArchitecture : Uint2B
    +0x26c NtMajorVersion   : Uint4B
    +0x270 NtMinorVersion   : Uint4B
    +0x274 ProcessorFeatures : [64] UChar
    +0x2b4 Reserved1        : Uint4B
    +0x2b8 Reserved3        : Uint4B
    +0x2bc TimeSlip         : Uint4B
    +0x2c0 AlternativeArchitecture : _ALTERNATIVE_ARCHITECTURE_TYPE
    +0x2c4 BootId           : Uint4B
    +0x2c8 SystemExpirationDate : _LARGE_INTEGER
    +0x2d0 SuiteMask        : Uint4B
    +0x2d4 KdDebuggerEnabled : UChar
    +0x2d5 MitigationPolicies : UChar
    +0x2d5 NXSupportPolicy  : Pos 0, 2 Bits
    +0x2d5 SEHValidationPolicy : Pos 2, 2 Bits
    +0x2d5 CurDirDevicesSkippedForDlls : Pos 4, 2 Bits
    +0x2d5 Reserved         : Pos 6, 2 Bits
    +0x2d6 CyclesPerYield   : Uint2B
    +0x2d8 ActiveConsoleId  : Uint4B
    +0x2dc DismountCount    : Uint4B
    +0x2e0 ComPlusPackage   : Uint4B
    +0x2e4 LastSystemRITEventTickCount : Uint4B
    +0x2e8 NumberOfPhysicalPages : Uint4B
    +0x2ec SafeBootMode     : UChar
    +0x2ed VirtualizationFlags : UChar
    +0x2ee Reserved12       : [2] UChar
    +0x2f0 SharedDataFlags  : Uint4B
    +0x2f0 DbgErrorPortPresent : Pos 0, 1 Bit
    +0x2f0 DbgElevationEnabled : Pos 1, 1 Bit
    +0x2f0 DbgVirtEnabled   : Pos 2, 1 Bit
    +0x2f0 DbgInstallerDetectEnabled : Pos 3, 1 Bit
    +0x2f0 DbgLkgEnabled    : Pos 4, 1 Bit
    +0x2f0 DbgDynProcessorEnabled : Pos 5, 1 Bit
    +0x2f0 DbgConsoleBrokerEnabled : Pos 6, 1 Bit
    +0x2f0 DbgSecureBootEnabled : Pos 7, 1 Bit
    +0x2f0 DbgMultiSessionSku : Pos 8, 1 Bit
    +0x2f0 DbgMultiUsersInSessionSku : Pos 9, 1 Bit
    +0x2f0 DbgStateSeparationEnabled : Pos 10, 1 Bit
    +0x2f0 SpareBits        : Pos 11, 21 Bits
    +0x2f4 DataFlagsPad     : [1] Uint4B
    +0x2f8 TestRetInstruction : Uint8B
    +0x300 QpcFrequency     : Int8B
    +0x308 SystemCall       : Uint4B
    ```

6. 此时按照我们当前的发现，应当可以用`dt _KUSER_SHARED_DATA 7FFE0000`，来访问到值，同样可以用，`dt _nt!KUSER_SHARED_DATA 0xFFFFF78000000000`,得到一样的结果，在用户层的应用必须要用用户层的虚拟地址，内核层的应用用内核层的虚拟地址。这个结构同一块物理内存，映射到用户层是0x7FFE000,映射到内核层就是0xFFFF F780 0000 0000
7. 分析调用，发现是在某个值=1的时候执行int 2E，不等于1的时候进行system call
8. 使用IDA静态分析，发现ds的内容是一致的，说明他没有被重定位，在Windows中，这种写法的，基本都是_KUSER_SHARED_DATA 结构体里的某个结构 ，_KUSER_SHARED_DATA结构体里保存了零环合三环公用的一些结构
9.  紧接着，是INT 2E,INT是调用一些中断处理程序，以2E为索引，在中断描述符表（IDT）中找
