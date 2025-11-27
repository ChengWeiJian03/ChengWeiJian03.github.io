---
title: 如何手动加载PE文件，论PeLoader的编写
date: 2025-11-27
toc: false
tags: [逆向]
---

## 摘要

PE文件相关知识铺天盖地，最开始只知道闷头看，轻重缓急重要与否一概不知，导致看完教程后似有所得但都不甚明了，故现挑出一主线，手动加载PE文件到内存中，以此分轻重缓急，梳理知识。

我们已经知道，一个程序正确运行需要从硬盘加载到内存当中，这篇文章主要就是用于描述**一个程序是怎么从硬盘加载到内存中的**。

项目地址：https://github.com/ChengWeiJian03/PeLoader



## 第一部分 PE文件概述

PE格式就是可执行文件格式，包括EXE、DLL、SYS都符合PE格式

### 1.1 这个格式大概长什么样

```c++
+-------------------+  0x00000000
| DOS Header (MZ)   |  ← DOS Stub + DOS头 (e_lfanew指向PE头)
+-------------------+  
| PE Header (NT)    |  ← NT签名 + File Header + Optional Header
+-------------------+
| Section Table     |  ← 描述所有节区的信息
+-------------------+
| .text (代码段)     |  ← 实际节区数据，按VirtualAddress对齐
| .data (数据段)     |
| .rdata (只读数据)  |
| .idata (导入表)    |
| .reloc (重定位表)  |
| .rsrc (资源)       |
| ...               |

```

或许这个图比较抽象，我们用一个二进制编辑器来打开微信看看，如下图所示，与上面对应，绿色的地方就是DOS头，中间两个颜色是DosStub，因为这部分并不重要所以在上面的图中没有对应部分，最下面露出一点尖尖角的黄色，就是NT头，节区在这张图中并没有表示。

![image-20251127164740040](C:\Users\ROOT\AppData\Roaming\Typora\typora-user-images\image-20251127164740040.png)

大部分PE文件，都符合上面两张图的样子，现在只需要知道一下我们会挨个来讲。

### 1.2 PE头概述

现在回想一下你写一个C++程序，一般都会有哪些东西，局部变量、全局变量、函数、一些资源文件比如程序的图标。

而他们编译完后都将储存在这个二进制文件中，但他们绝大部分都储存在Section中，那既然都储存在section中了，为什么我们需要有“头”这种东西呢？就我们就来看“头”里储存了哪些东西。

**DOS Header（DOS头，0x40字节）** 它的作用就是兼容老DOS系统，但在现代它的作用就是当你在DOS系统运行的时候弹出一句”这个软件并不支持DOS系统“，但dos头中的有一个字段，它可以告诉我们NT头的开始位置在哪，就是e_lfanew字段，好，我们来说一下e_lfanew字段是哪里来的。这个时候我们要百度搜索DOS头结构

```C++
typedef struct _IMAGE_DOS_HEADER {      // DOS .EXE header
    WORD   e_magic;                     // Magic number
    WORD   e_cblp;                      // Bytes on last page of file
    WORD   e_cp;                        // Pages in file
    WORD   e_crlc;                      // Relocations
    WORD   e_cparhdr;                   // Size of header in paragraphs
    WORD   e_minalloc;                  // Minimum extra paragraphs needed
    WORD   e_maxalloc;                  // Maximum extra paragraphs needed
    WORD   e_ss;                        // Initial (relative) SS value
    WORD   e_sp;                        // Initial SP value
    WORD   e_csum;                      // Checksum
    WORD   e_ip;                        // Initial IP value
    WORD   e_cs;                        // Initial (relative) CS value
    WORD   e_lfarlc;                    // File address of relocation table
    WORD   e_ovno;                      // Overlay number
    WORD   e_res[4];                    // Reserved words
    WORD   e_oemid;                     // OEM identifier (for e_oeminfo)
    WORD   e_oeminfo;                   // OEM information; e_oemid specific
    WORD   e_res2[10];                  // Reserved words
    LONG   e_lfanew;                    // File address of new exe header
} IMAGE_DOS_HEADER, *PIMAGE_DOS_HEADER;

```

好复杂一个结构体，我们要怎么用呢，我们都知道WORD是二字节，从上面的图中截出来这一段拿下来

![image-20251127170046671](C:\Users\ROOT\AppData\Roaming\Typora\typora-user-images\image-20251127170046671.png)

那么第一个二字节对应的就是e_magic，第二个二次节对应的就是e_cblp，严格按照上面代码块中的排布一一对应下来，我们就能得到e_lfanew字段中储存的值是多少。

当然我们有更简单的方法，现在的二进制编辑器中基本上都支持模板，模板会自动帮你一一对应，我们在模板中查看一下，对应关系我已经帮你标出来了。

![image-20251127170414423](C:\Users\ROOT\AppData\Roaming\Typora\typora-user-images\image-20251127170414423.png)

那么我们就可以通过这个偏移来得到NT头，DOS头的任务已经完成了，现在主角给到NT头

```c++
NT头结构如下
+-------------------+
| PE 签名 (Signature) |  // "PE\0\0" (0x00004550)
+-------------------+
| 文件头 (File Header) |  // IMAGE_FILE_HEADER
+-------------------+
| 可选头 (Optional Header) |  // IMAGE_OPTIONAL_HEADER{32|64}
+-------------------+
| 数据目录表 (Data Directories) |  // 嵌入在 Optional Header 末尾
+-------------------+

```

**NT头**，乍一看好像比DOS头少了很多东西，真正重要的东西都藏在“可选头”中，这个可选头一点都不“可选”，它可以展开很长一大段，并且里面每个字段都很重要。整个nt头中最重要的东西只有两个，一个是**可选头**，一个是**数据目录表**

```c++
可选头结构如下: 

typedef struct _IMAGE_OPTIONAL_HEADER32 {
    WORD    Magic;                      // 0x010B (PE32)
    BYTE    MajorLinkerVersion;         // 链接器主版本
    BYTE    MinorLinkerVersion;         // 链接器次版本
    DWORD   SizeOfCode;                 // 代码段大小
    DWORD   SizeOfInitializedData;      // 已初始化数据段大小
    DWORD   SizeOfUninitializedData;    // 未初始化数据段大小 (BSS)
    DWORD   AddressOfEntryPoint;        // 入口点 RVA
    DWORD   BaseOfCode;                 // 代码段基址 RVA
    DWORD   BaseOfData;                 // 数据段基址 RVA (32-bit 特有)
    DWORD   ImageBase;                  // 首选加载基址 (e.g., 0x400000)
    DWORD   SectionAlignment;           // 内存中节对齐粒度 (通常 0x1000)
    DWORD   FileAlignment;              // 文件中节对齐粒度 (通常 0x200)
    WORD    MajorOperatingSystemVersion;// OS 主版本
    WORD    MinorOperatingSystemVersion;// OS 次版本
    WORD    MajorImageVersion;          // 映像主版本
    WORD    MinorImageVersion;          // 映像次版本
    WORD    MajorSubsystemVersion;      // 子系统主版本
    WORD    MinorSubsystemVersion;      // 子系统次版本
    DWORD   Win32VersionValue;          // 保留 (通常 0)
    DWORD   SizeOfImage;                // 映像总大小 (内存中)
    DWORD   SizeOfHeaders;              // 所有头大小 (到第一个节前)
    DWORD   CheckSum;                   // 校验和 (驱动程序需要)
    WORD    Subsystem;                  // 子系统类型 (e.g., 2=GUI, 3=Console)
    WORD    DllCharacteristics;         // DLL 特性 (e.g., ASLR 等)
    DWORD   SizeOfStackReserve;         // 栈保留大小
    DWORD   SizeOfStackCommit;          // 栈提交大小
    DWORD   SizeOfHeapReserve;          // 堆保留大小
    DWORD   SizeOfHeapCommit;           // 堆提交大小
    DWORD   LoaderFlags;                // 加载器标志 (已弃用)
    DWORD   NumberOfRvaAndSizes;        // 数据目录数量 (通常 16)
    IMAGE_DATA_DIRECTORY DataDirectory[IMAGE_NUMBEROF_DIRECTORY_ENTRIES];
                                        // 数据目录数组 (16 个 IMAGE_DATA_DIRECTORY)
} IMAGE_OPTIONAL_HEADER32, *PIMAGE_OPTIONAL_HEADER32;

```

都说了是概述，可选头等真正用到的时候我们再细讲，我们按照顺序往下来，下面是**节表**

**Section Table（节表，NumberOfSections \* 40字节）**：每个节表项描述一个节区，作用是找到正确的**节区**

```C++
以下为Section Table结构

typedef struct _IMAGE_SECTION_HEADER {
    BYTE    Name[8];                    // 节名称（8 字节 ASCII，右填充 NULL 或空格）
    union {
        DWORD   PhysicalAddress;        // 物理地址（旧版字段，实际常用于 SizeOfRawData 的备用）
        DWORD   VirtualSize;            // 虚拟大小（节在内存中的大小，页对齐）
    } Misc;
    DWORD   VirtualAddress;             // 虚拟地址（节在 RVA 中的起始偏移，页对齐）
    DWORD   SizeOfRawData;              // 原始数据大小（文件中的实际大小）
    DWORD   PointerToRawData;           // 指向原始数据的文件偏移（File Offset）
    DWORD   PointerToRelocations;       // 重定位表偏移（旧版 PE，已弃用）
    DWORD   PointerToLinenumbers;       // 行号表偏移（调试用，已弃用）
    WORD    NumberOfRelocations;        // 重定位数（旧版，已弃用）
    WORD    NumberOfLinenumbers;        // 行号数（调试用，已弃用）
    DWORD   Characteristics;            // 节属性标志（例如可读、可写、可执行等）
} IMAGE_SECTION_HEADER, *PIMAGE_SECTION_HEADER;

```

**Sections（节区）**：储存实际的代码、数据，按文件偏移（PointerToRawData）读取。

## 一些关键概念补全

1. **关于内存**

	在写程序的时候，或者在动态调试程序的时候，所看到的内存地址，全部都是虚拟内存地址，虚拟内存地址和物理地址相互映射，虚拟内存可以通过页表映射来查找到实际的物理内存（PTPDE、PTE这些），为了快速访问，虚拟内存一般以**4KB (0x1000)**页的方式进行对齐（SectionAlignment），为了在硬盘上储存更紧密，在文件中一般以**512字节 (0x200)**的方式对齐（FileAlignment）。

2. **RVA 与 VA**

   - **RVA (Relative Virtual Address, 相对虚拟地址)**：PE文件中所有地址引用（如节区的VirtualAddress、入口点AddressOfEntryPoint）都是RVA，它是相对于**ImageBase**的偏移量，不是绝对地址。
   - **VA (Virtual Address, 虚拟地址)**：实际内存中的绝对地址，计算公式：**VA = ImageBase + RVA**。
   - 示例：如果ImageBase=0x400000，入口点RVA=0x1000，则OEP（Original Entry Point，原入口点）VA=0x401000。

3. **ImageBase（首选加载基址）**

   可选头中的**ImageBase**字段（通常如0x400000），表示PE文件**首选**加载到的内存基址。如果系统能分配到这个地址，就无需重定位；否则加载到其他地址，并通过重定位表修正所有内部地址引用。

4. **节区对齐与映射规则**

   - **文件对齐 (FileAlignment, 通常0x200)**：节区在文件中的数据（从PointerToRawData开始，长度SizeOfRawData）按此对齐，便于磁盘读写。
   - **内存对齐 (SectionAlignment, 通常0x1000)**：节区在内存中（从ImageBase + VirtualAddress开始）按页对齐。
   - **映射过程**：读取文件数据到内存对应位置，如果VirtualSize > SizeOfRawData，则用零填充剩余部分至对齐大小；节区属性（Characteristics）决定权限（如CODE可执行、DATA可读写）。

5. **数据目录 (Data Directory)**

   可选头末尾的数组（NumberOfRvaAndSizes通常16个IMAGE_DATA_DIRECTORY），每个条目包含**RVA**和**Size**，指向关键表的位置：
   | 索引 | 名称                  | 作用                     |
   |------|-----------------------|--------------------------|
   | 0    | Export Table         | DLL导出函数表            |
   | 1    | Import Table         | 导入DLL及函数表 (IAT)   |
   | 5    | Relocation Table     | 重定位表                 |
   | 6    | Resource Table       | 资源（如图标、对话框）  |
   | ...  | ...                  | ...                      |
   - 手动加载时，主要处理1（导入）和5（重定位）。

6. **重定位表 (Relocation Table)**

   如果实际加载基址（RealBase） != ImageBase，需要遍历重定位表（数据目录表数组[5]指向），修正所有“硬编码”VA：
   - 结构：块头（RVA + Size） + 类型+偏移数组（类型3=IMAGE_REL_BASED_HIGHLOW，偏移是需要修正的RVA）。
   - 修正公式：内存[RVA + 偏移] += (RealBase - ImageBase)。

7. **导入表 (Import Table / IAT)**

   - **INT (Import Name Table)**：列出依赖DLL名和函数名/序号。
   - **IAT (Import Address Table)**：实际地址表，加载时填充GetProcAddress获取的函数VA。
   - 解析步骤：遍历导入描述符 → LoadLibrary(DLL) → GetProcAddress(函数) → 填入IAT。

8. **入口点 (Entry Point)**

   可选头**AddressOfEntryPoint**是RVA，加载完成后，跳转到**ImageBase + RVA**执行程序逻辑（如mainCRTStartup）。对于DLL，还需处理DllMain。

这些概念是手动加载PE的核心，接下来文章将基于它们实现PeLoader：映射头+节区 → 处理导入 → 处理重定位 → 跳转入口点。



## 第二部分 手动加载PE：从文件到内存的全过程

**先贴出全流程的伪代码，我们先有个宏观认识**

1. 首先我们要把要其中的应用的全部内容读进来CreateFile()+ReadFile()

2. 为了能把读到的内容正确加人到内存中，我们需要申请一块内存空间，而申请的大小是NT头里的SizeOfImage字段
3. 紧接着我们把**PE头**全部复制到开辟的内存中
4. 把所有的**节**都复制到内存中，由于节表在PE头中，所以不用额外复制
5. 由于IAT是在内存加载的时候动态填充，因此需要根据导入表找到需要加载的DLL文件，只需要在自己的exe文件中加载即可，因为进程之间的空间是共享的，不需要在打开的文件空间额外拷贝一份，根据INT表，找到需要加载的函数，把加载后的函数地址填充到IAT表中。
6. 由于在硬盘中的对齐策略和在内存中的对齐策略是不同的，因此函数地址会发生变化，所以需要重定位，需要重定位的函数会写在重定位表中。
7. 修改程序入口点，执行PE文件的主函数

```c++
main():
    filename = "test.exe"
    hFile = CreateFile(filename)  // 打开文件
    fileSize = GetFileSize(hFile)
    fileData = new char[fileSize]  // 读取文件到缓冲区
    ReadFile(hFile, fileData, fileSize)
    
    imageBase = AllocMemory(fileData)  // 根据PE头.SizeOfImage分配可执行内存 (VirtualAlloc, PAGE_EXECUTE_READWRITE)
    
    CopyHeaders(fileData, imageBase)   // 复制DOS头 + NT头 + 节表 (大小: SizeOfHeaders)
    
    CopySections(fileData, imageBase)  // 遍历所有节区，复制原始数据到VirtualAddress位置 (SizeOfRawData)
    
    FixIAT(fileData, imageBase)        // 解析导入表：
                                       //   遍历IMAGE_IMPORT_DESCRIPTOR (每个DLL)
                                       //   LoadLibrary(DLL名)
                                       //   遍历INT (OriginalFirstThunk) -> 解析函数名/Ordinal -> GetProcAddress -> 填充IAT (FirstThunk)
    
    PerformRelocations(imageBase)      // 如果实际基址 != 首选ImageBase，进行重定位：
                                       //   delta = imageBase - PE.ImageBase
                                       //   遍历重定位块 (IMAGE_BASE_RELOCATION链)
                                       //   每个条目：高4位类型(IMAGE_REL_BASED_HIGHLOW/DIR64)，低12位offset
                                       //   fixupAddr = imageBase + block.VirtualAddress + offset
                                       //   *fixupAddr += delta  // 修正地址引用
    
    // 获取并调用入口点
    dosHeader = (DOS_HEADER*)imageBase
    ntHeaders = (NT_HEADERS*)(imageBase + dosHeader.e_lfanew)
    entryPoint = (imageBase + ntHeaders.AddressOfEntryPoint)  // OEP (Original Entry Point)
    call entryPoint()  // 执行PE文件的主函数（如WinMain/DLLMain）

// 辅助函数简述（已内联关键逻辑）：
AllocMemory(fileData):
    ntHeaders = 从fileData解析NT头
    return VirtualAlloc(NULL, ntHeaders.SizeOfImage, COMMIT|RESERVE, EXECUTE_READWRITE)
```

### 现在我们来动手实现

1. **读取PE文件内容**

    ```C++
    LPCSTR filename = "test.exe";
    HANDLE hFile = CreateFileA(filename, GENERIC_READ, FILE_SHARE_READ, NULL, OPEN_EXISTING, FILE_ATTRIBUTE_NORMAL, 0);
    
    if (hFile)
    {
        DWORD dwFileSize = GetFileSize(hFile, NULL);
        LPSTR fileData = new CHAR[dwFileSize];
    
        if (fileData)
        {
            RtlZeroMemory(fileData, dwFileSize);
            DWORD dwReadSize = 0;
            if (ReadFile(hFile, fileData, dwFileSize, &dwReadSize, NULL))
                std::cout << "文件加载成功";
            else
                std::cout << "文件加载失败";
            LPVOID imageBase = AllocMemery(fileData);
    
            MEMORY_BASIC_INFORMATION mbi;
            SIZE_T querySize = VirtualQuery(imageBase, &mbi, sizeof(mbi));
            if (querySize == sizeof(mbi)) {
                printf("查询区域大小: 0x%zX (%zu 字节)\n", mbi.RegionSize, mbi.RegionSize);
                // mbi.RegionSize == allocSize（正常情况下）
                printf("状态: %s, 保护: 0x%X\n",
                    (mbi.State == MEM_COMMIT ? "已提交" : "预留"), mbi.Protect);
            }
            else {
                printf("查询失败\n");
            }
        }
    ```

2. **开辟内存空间**
   ```c++
   LPVOID AllocMemery(LPSTR fileData)
   {
   	//根据NT Headers->Optional Header->Size of image 来决定申请多大的内存
   	PIMAGE_DOS_HEADER DosHeader = (PIMAGE_DOS_HEADER)fileData;
   	//计算NT头
   	PIMAGE_NT_HEADERS NtHeaders = (PIMAGE_NT_HEADERS)((ULONG_PTR)fileData + DosHeader->e_lfanew);
   	//通过NT头计算大小
   	LPVOID imageBase = VirtualAlloc(NULL, NtHeaders->OptionalHeader.SizeOfImage, MEM_COMMIT | MEM_RESERVE, PAGE_EXECUTE_READWRITE);
   	if (!imageBase)
   		return nullptr;
   	return imageBase;
   }
   ```

   

3. **把PE头全部复制到开辟的内存中**
   ```c++
   bool CopyHeaders(LPSTR fileData, LPVOID imageBase)
   {
   	PIMAGE_DOS_HEADER DosHeader = (PIMAGE_DOS_HEADER)fileData;
   	PIMAGE_NT_HEADERS NtHeaders = (PIMAGE_NT_HEADERS)((ULONG_PTR)fileData + DosHeader->e_lfanew);
   	RtlCopyMemory(imageBase, DosHeader, NtHeaders->OptionalHeader.SizeOfHeaders);
   	return true;
   }

4. **把所有的节都复制到内存中**
   ```C++
   void CopySections(LPSTR fileData, LPVOID imageBase)
   {
   	PIMAGE_DOS_HEADER DosHeader = (PIMAGE_DOS_HEADER)fileData;
   	PIMAGE_NT_HEADERS NtHeaders = (PIMAGE_NT_HEADERS)((ULONG_PTR)fileData + DosHeader->e_lfanew);
   
   	PIMAGE_SECTION_HEADER SectionHeader = IMAGE_FIRST_SECTION(NtHeaders);
   
   	DWORD dwSectionNumber = NtHeaders->FileHeader.NumberOfSections;
   	for (DWORD i = 0; i < dwSectionNumber; i++)
   	{
   		RtlCopyMemory((PVOID)((ULONG_PTR)imageBase + SectionHeader->VirtualAddress), (PVOID)((ULONG_PTR)DosHeader + SectionHeader->PointerToRawData), (SectionHeader->SizeOfRawData));
   
   		SectionHeader++;
   	}
   
   }
   ```

5. **修复IAT表**
   ```C++
   bool FixIAT(LPSTR fileData, LPVOID imageBase)
   {
   	PIMAGE_DOS_HEADER DosHeader = (PIMAGE_DOS_HEADER)fileData;
   	PIMAGE_NT_HEADERS NtHeaders = (PIMAGE_NT_HEADERS)((ULONG_PTR)fileData + DosHeader->e_lfanew);
   	PIMAGE_DATA_DIRECTORY pImportDir = &NtHeaders->OptionalHeader.DataDirectory[IMAGE_DIRECTORY_ENTRY_IMPORT];
   	if (pImportDir->VirtualAddress == 0) {
   		return true;  // 无导入，成功
   	}
   	// 2. 遍历 IMAGE_IMPORT_DESCRIPTOR 数组
   	PIMAGE_IMPORT_DESCRIPTOR pImportDesc = (PIMAGE_IMPORT_DESCRIPTOR)RVA_TO_VA(imageBase, pImportDir->VirtualAddress);  // 宏：((PVOID)((ULONG_PTR)base + rva))
   	while (pImportDesc->Name != 0)
   	{
   		LPCSTR szDllName = (LPCSTR)RVA_TO_VA(imageBase, pImportDesc->Name);
   		HMODULE hDll = LoadLibraryA(szDllName);
   		if (hDll == NULL)
   			return false;
   		PIMAGE_THUNK_DATA Thunk = (PIMAGE_THUNK_DATA)((ULONG_PTR)imageBase + pImportDesc->FirstThunk);     // IAT (要写的目标)
   		PIMAGE_THUNK_DATA OrigThunk = (PIMAGE_THUNK_DATA)((ULONG_PTR)imageBase + pImportDesc->OriginalFirstThunk);  // INT (只读源)
   
   		while (OrigThunk->u1.AddressOfData != 0)
   		{
   			if (OrigThunk->u1.Ordinal & IMAGE_ORDINAL_FLAG)  // Ordinal导入 (高位标记)
   			{
   				// // 提取Ordinal低16位，用MAKEINTRESOURCE传GetProcAddress（API内部区分序数），就是提取函数序号
   				LPCSTR funcName = (LPCSTR)(OrigThunk->u1.Ordinal & 0xFFFF);
   				Thunk->u1.Function = (ULONG_PTR)GetProcAddress(hDll, funcName);  
   			}
   			else  // 名称导入
   			{
   				// AddressOfData是RVA -> VA，指向IMAGE_IMPORT_BY_NAME
   				PIMAGE_IMPORT_BY_NAME ImpByName = (PIMAGE_IMPORT_BY_NAME)((ULONG_PTR)imageBase + OrigThunk->u1.AddressOfData);
   				// Name是ANSI字符串
   				Thunk->u1.Function = (ULONG_PTR)GetProcAddress(hDll, ImpByName->Name);  // 解析函数地址
   			}
   			if (Thunk->u1.Function == 0) return FALSE;  // 失败返回
   			Thunk++;      // IAT下一个
   			OrigThunk++;  // INT下一个
   		}
   		pImportDesc++;  // 下一个DLL
   	}
   	return true;
   }
   ```

6. **基址重定位**
   ```C++
   //重定位
   
   BOOL PerformRelocations(LPVOID imageBase)
   {
   	//重定位需要什么 1. 需要修正的地址 2. 建议的加载基址 3. 实际加载的基址
   	// 
   	// 需要修正的地址在重定位表中即IMAGE_BASE_RELOCATION结构体中的TypeOffset的低12位中，建议的加载基址在NT头的Optional Header中的ImageBase字段，实际加载的基址就是imageBase参数
   
   	//一个 PE文件只有一个 重定位表，但这个单一表内部确实是多个块的链式结构，每个块又包含多个重定位项
   	
   	PIMAGE_DOS_HEADER DosHeader = (PIMAGE_DOS_HEADER)imageBase;
   	PIMAGE_NT_HEADERS NtHeaders = (PIMAGE_NT_HEADERS)((ULONG_PTR)imageBase + DosHeader->e_lfanew);
   	ULONG_PTR delta = (ULONG_PTR)imageBase - NtHeaders->OptionalHeader.ImageBase;
   	if (delta == 0) return TRUE;  // 无需重定位
   
   	PIMAGE_DATA_DIRECTORY RelocDir = &NtHeaders->OptionalHeader.DataDirectory[IMAGE_DIRECTORY_ENTRY_BASERELOC]; //重定位表
   	if (RelocDir->VirtualAddress == 0) return TRUE;  // 无重定位表
   
   	//_IMAGE_DATA_DIRECTORY里面的VirtualAddress是当前重定位表的RVA，因此通过RVA加上imageBase就能得到重定位表的VA
   	PIMAGE_BASE_RELOCATION RelocBlock = (PIMAGE_BASE_RELOCATION)((ULONG_PTR)imageBase + RelocDir->VirtualAddress);//第一个重定位块
   
   	while (RelocBlock->VirtualAddress != 0)
   	{
   		DWORD entryCount = (RelocBlock->SizeOfBlock - sizeof(IMAGE_BASE_RELOCATION)) / sizeof(WORD); //计算当前重定位块中 重定位条目的个数，整个块的字节大小减去头部大小，再除以每个条目大小
   		
   		// 我当前的编译器中IMAGE_BASE_RELOCATION的TypeOffset被注释掉了，因此在计算relocData的时候不能直接使用RelocBlock->TypeOffset，而是需要通过计算得到
   		PWORD relocData = (PWORD)((PBYTE)RelocBlock + sizeof(IMAGE_BASE_RELOCATION)); //当前重定位表的地址加上头的大小就是 重定位条目数组的起始地址
   
   		for (DWORD i = 0; i < entryCount; i++)
   		{
   			WORD type = relocData[i] >> 12; //高4位是类型
   			WORD offset = relocData[i] & 0xFFF; //低12位是偏移
   
   			if (type == IMAGE_REL_BASED_ABSOLUTE) continue;  // 跳过
   
   			ULONG_PTR* fixup = (ULONG_PTR*)((ULONG_PTR)imageBase + RelocBlock->VirtualAddress + offset);
   			if (type == IMAGE_REL_BASED_HIGHLOW || type == IMAGE_REL_BASED_DIR64)  // x86 或 x64
   			{
   				*fixup += delta;
   			}
   			// 可扩展其他类型，如 ARM 等
   		}
   
   		RelocBlock = (PIMAGE_BASE_RELOCATION)((PBYTE)RelocBlock + RelocBlock->SizeOfBlock);
   	}
   
   	return TRUE;
   
   }
   ```

   

7. **修改程序入口**
   
   ```C++
   typedef VOID(*EntryPoint)();
   			EntryPoint entry = (EntryPoint)((ULONG_PTR)imageBase + NtHeaders->OptionalHeader.AddressOfEntryPoint);
   			entry();
   ```
   
   

**至此PeLoader的编写就完成了,可以在开头的GitHub下载到完成代码**
