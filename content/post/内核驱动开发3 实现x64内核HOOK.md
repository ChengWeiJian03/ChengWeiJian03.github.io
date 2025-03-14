---
title: 内核驱动开发3 实现过PG保护的x64内核HOOK
published: 2024-11-19T22:06:26+08:00
summary: "实现过PG保护的x64内核HOOK"
tags: [内核,pg保护]
categories: '内核驱动'
draft: false 
lang: ''
---

# 第三部分 过PG保护的内核HOOK

## 3.1 什么是PG保护

Windows写了一个程序叫Patch Guard 在Windows内核的关键位置加上了监控，一旦被修改就会触发蓝屏，蓝屏是作为一种惩罚机制存在的

## 3.2 过PG保护的必要性

由于我们需要HOOK内核函数，而内核函数是在 PG 保护的范畴内，当 HOOK 完成后，NTOpenProcess 的内存空间发生变化，会被PG 检测到，随后电脑会蓝屏，在第零部分说了这是一种惩罚机制。一般在 hook 后半小时到一小时内会被检测到

## 3.3 内核内存共享

window 的内存管理机制中，PML4T、PDPT、PD、PT中的都分为两部分，其中一部分是用户层，其中每个 table 是隔离的，指向不同的页和物理地址，而另一部分指向的是内核部分的内存，里面包含了内核函数等内容，这部分的内存是共享的，每个进程的表的内核部分都长一样，导致如果在进程 A 中修改内核部分的函数，进程 B 的内核函数也会发生改变，因为内核空间的内存是共享的

## 3.4 PG 的代码在内核哪里

使用 IDA 打开内核exe，依据字节大小排序，倒数第二大的就是 PG 的代码，是一个匿名（没有符号）的函数

为什么这是 PG 函数：不知道，教程说的

## 3.5 什么是挂载在某个进程

如果我们的程序是进程 A，目标进程是进程 B，为了访问进程 B 的页表需要让进程 A 挂载到进程 B 上，使用的函数是 K，在windbg中使用的命令是.process /i 进程的内存地址,使用的函数是KeStackAttachProcess(process, &apc);

## 3.6 过 PG 保护原理

假设我们要HOOK进程A的NTOpenProcess,原本只需要随便在一个进程里修改函数，进程A的NTOpenProcess也会同步修改，而这块内存PG是会检测的

现在，我们创建一片物理内存空间，假设叫FakeNtoskrl，将Ntoskrl里的内容全部复制进FakeNtoskrl，修改FakeNtoskrl里面的NTOpenProcess内容，并且修改进程A里原本的PT，使其PTE指向FakeNtoskrl，则完成HOOK且不会被检测

由此我们就可以得到一个方法,我们创建一连串pdpt,pdt,pt,然后通过cr3找到pml4t,修改pml4t,让其中的pfn指向我们伪造的pdpt,修改伪造的pdpte,使其中的pfn指向伪造的pdt,修改伪造的pdt中的pfn使其指向伪造的pt.

但同时要了解一个点,内核函数的地址往往是指向一个2M的大页,而不是一个4k的小页,
这意味着他的指向不是pml4t-->pdpt-->pdt-->pt
而是pml4t-->pdpt-->pdt,pdt中的PFN就直接指向2M大小的物理内存

但是申请2m大小的连续物理内存是申请不下来的,因此我们可以构建一个虚假的pt,让pdt指向虚假的pt,让每个pt的表项指向4k大小的物理地址,pt可以储存512个表项,因此可以指向一个2m的大页,注意要将大页位置置为0

**在这里强调分割前后的区别**

**分割前**
| PDE   | page_frame_number | large_page | flags |
| ----- | ----------------- | ---------- | ----- |
| InPde | X                 | 1          | flags |


**申请PT表**
| PT Entry | flags | large_page | global | page_frame_number |
| -------- | ----- | ---------- | ------ | ----------------- |
| Pt[0]    | flags | 0          | 0      | StartPfn          |
| Pt[1]    | flags | 0          | 0      | StartPfn + 1      |
| Pt[2]    | flags | 0          | 0      | StartPfn + 2      |
| ...      | ...   | ...        | ...    | ...               |
| Pt[511]  | flags | 0          | 0      | StartPfn + 511    |

**创建新的PDT**
| PDE    | page_frame_number | large_page | flags |
| ------ | ----------------- | ---------- | ----- |
| OutPde | Pt Address        | 0          | flags |

```
[ InPde: Large Page ]             ==> [ OutPde: Points to PT ]
+----------------------------+        +--------------------------+
| page_frame_number: X        |        | page_frame_number: Pt Addr|
| large_page: 1               |        | large_page: 0            |
| flags: flags                |        | flags: flags             |
+----------------------------+        +--------------------------+

          |
          v

[ PT (Page Table) ]
+-----------------------------------------+
| Pt[0]: page_frame_number = StartPfn     |
| Pt[1]: page_frame_number = StartPfn + 1 |
| Pt[2]: page_frame_number = StartPfn + 2 |
| ...                                     |
| Pt[511]: page_frame_number = StartPfn+511|
+-----------------------------------------+

```

## 3.7 整理思路

(需要参看上一篇)
由上可以得到结论:

1. 需要一个函数把大页分割成小页
2. 由于PFN是物理地址,无法直接使用和运算,因此要封装一个物理地址转虚拟地址和一个虚拟地址转物理地址的函数
3. 需要一个函数可以算出需要hook的函数的一串原始表项基址(前面已经实现)
4. 需要MDL进行读写内存(前面已经实现)
5. 需要实现一个填充512个4k页面的函数

## 3.8 部分函数的实现(具体的代码在code文件夹)

**函数结构一览**
```C++
//函数结构一览

#include <ntifs.h>
#include <ntddk.h>
#include "stucter.h"
#include "ia32/ia32.hpp"


class hook
{
public:
	void OffPGE();
	bool ReplacePageTable(cr3 CR3, void* replaceAlignAddr, pde_64* pde);
	bool InstallInlineHook(void** originAddr, void* hookAddr);
	bool RemoveInlineHook(void* hookAddr);
	static hook* GetInstance();
	bool IsoationPageTable(PEPROCESS process, void* isolateAddr);
	bool SplitLargePage(pde_64 InPde, pde_64& OutPde);
	ULONG64 VaToPa(void* va);
	void* PaToVa(ULONG64 pa);

	UINT32 mHoookCount = 0;
	HOOK_INFO m_hook_info[MAX_HOOKS]={0};

	char* mTrampLinePool = 0; //跳板内存地址

	//跳板使用的字节
	UINT32 mPoolUsed = 0;

	//实例
	static hook* mInstance;

};

```

**分割页面函数实现**
```C++ 
// 分割页面函数实现
bool hook::SplitLargePage(pde_64 InPde, pde_64& OutPde)
{
	PHYSICAL_ADDRESS MaxAddrPa{ 0 }, LowAddrPa{ 0 };
	MaxAddrPa.QuadPart = MAXULONG64;
	LowAddrPa.QuadPart = 0;

	pt_entry_64* Pt;

	uint64_t StartPfn = InPde.page_frame_number;

	Pt = (pt_entry_64*)MmAllocateContiguousMemorySpecifyCache(PAGE_SIZE, LowAddrPa, MaxAddrPa, LowAddrPa, MmCached);
	if (!Pt) {
		DbgPrint("failed to allocate pt mem! \n ");
		return false;
	}

	for (int i = 0; i < 512; i++) {
		Pt[i].flags = InPde.flags;
		Pt[i].large_page = 0;
		Pt[i].global = 0;
		Pt[i].page_frame_number = StartPfn + i;
	}

	OutPde.flags = InPde.flags;
	OutPde.large_page = 0;
	OutPde.page_frame_number = VaToPa(Pt) / PAGE_SIZE;

	return true;
}
```

**替换表项函数实现**
```C++
// 替换表项函数实现
bool hook::ReplacePageTable(cr3 CR3, void* replaceAlignAddr, pde_64* pde)
{
	uint64_t* Va4kb, * VaPt, * VaPdt, * VaPdpt, * VaPml4t;


	PHYSICAL_ADDRESS MaxAddrPa{ 0 }, LowAddrPa{ 0 };
	MaxAddrPa.QuadPart = MAXULONG64;
	LowAddrPa.QuadPart = 0;

	PAGE_TABLE pageTable = { 0 };

	Va4kb = (uint64_t*)MmAllocateContiguousMemorySpecifyCache(PAGE_SIZE, LowAddrPa, MaxAddrPa, LowAddrPa, MmCached);
	VaPt = (uint64_t*)MmAllocateContiguousMemorySpecifyCache(PAGE_SIZE, LowAddrPa, MaxAddrPa, LowAddrPa, MmCached);
	VaPdt = (uint64_t*)MmAllocateContiguousMemorySpecifyCache(PAGE_SIZE, LowAddrPa, MaxAddrPa, LowAddrPa, MmCached);
	VaPdpt = (uint64_t*)MmAllocateContiguousMemorySpecifyCache(PAGE_SIZE, LowAddrPa, MaxAddrPa, LowAddrPa, MmCached);

	VaPml4t = (uint64_t*)PaToVa(CR3.address_of_page_directory * PAGE_SIZE);

	if (!Va4kb || !VaPt || !VaPdt || !VaPdpt) {
		DbgPrint("failed to allocate page table ! \n ");
		return false;
	}

	pageTable.VirtualAddress = replaceAlignAddr;
	GetPageTable(pageTable);



	UINT64 pml4eindex = ((UINT64)replaceAlignAddr & 0xFF8000000000) >> 39;
	UINT64 pdpteindex = ((UINT64)replaceAlignAddr & 0x7FC0000000) >> 30;
	UINT64 pdeindex = ((UINT64)replaceAlignAddr & 0x3FE00000) >> 21;
	UINT64 pteindex = ((UINT64)replaceAlignAddr & 0x1FF000) >> 12;



	if (pageTable.Entry.pde->large_page) {
		MmFreeContiguousMemorySpecifyCache(VaPt, PAGE_SIZE, MmCached);
		VaPt = (uint64_t*)PaToVa(pde->page_frame_number * PAGE_SIZE);
	}
	else {
		memcpy(VaPt, pageTable.Entry.pte - pteindex, PAGE_SIZE);
	}

	memcpy(Va4kb, replaceAlignAddr, PAGE_SIZE);

	memcpy(VaPdt, pageTable.Entry.pde - pdeindex, PAGE_SIZE);
	memcpy(VaPdpt, pageTable.Entry.pdpte - pdpteindex, PAGE_SIZE);


	auto pReplacPte = (pte_64*)&VaPt[pteindex];
	pReplacPte->page_frame_number = VaToPa(Va4kb) / PAGE_SIZE;

	auto pReplacPde = (pde_64*)&VaPdt[pdeindex];
	pReplacPde->page_frame_number = VaToPa(VaPt) / PAGE_SIZE;
	pReplacPde->large_page = 0;

	auto pReplacPdpt = (pdpte_64*)&VaPdpt[pdpteindex];
	pReplacPdpt->page_frame_number = VaToPa(VaPdt) / PAGE_SIZE;


	auto pReplacPml4e = (pml4e_64*)&VaPml4t[pml4eindex];
	pReplacPml4e->page_frame_number = VaToPa(VaPdpt) / PAGE_SIZE;

	//WRK 
	KeFlushEntireTb(true, false);

	OffPGE();

	return true;
}
```

**虚拟地址和物理地址互转**
```C++
ULONG64 hook::VaToPa(void* va)
{
	PHYSICAL_ADDRESS pa{ 0 };
	pa = MmGetPhysicalAddress(va);
	return pa.QuadPart;
}

void* hook::PaToVa(ULONG64 pa)
{
	PHYSICAL_ADDRESS Pa{0};
	Pa.QuadPart = pa;
	return MmGetVirtualForPhysical(Pa);
}
```