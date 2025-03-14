---
title: 内核驱动开发4 TLB机制与PCIDE和KPTI验证
published: 2024-11-20T22:06:26+08:00
summary: "实现过PG保护的x64内核HOOK"
tags: [内核,TLB,KPTI]
categories: '内核驱动'
draft: false 
lang: ''
---

# 第四部分 TLB机制和PCID和KPTI验证

## 1. 什么是TLB

详见:**x86 x64体系探索及编程[邓志] 高清扫描 完整版.pdf-318页**

TLB的出现是为了增加性能,我们已经知道了,访问内存的过程需要把虚拟内存转换成物理内存,而转换的过程需要pml4t-->pdpt-->pdt-->pt,如果每次访问内存都遍历这四个表就会很耗时,因此处理器引入了TLB

TLB是catche的一类,通过TLB处理器可以绕过内存里的table和table entry,直接在catche里查找页的转换后的结果(就是虚拟内存和物理内存一一对应的关系),这个结果包括了最终的物理页基址和页的属性

Intel64 实现了两个catche,一个TLB,一个是Paging-structure Catche,TLB保存了一一对应的关系,而另一个保存了他的寻址过程(页表项属性),即pml4t-->pdpt-->pdt-->pt四个表的地址

当处理器首次成功访问page frame 会在当前PCID的TLB里建立相应的 TLB entry来保存page frame,或者建立global TLB entry 保存global page frame**(当page的G=1时,所有的内核函数都有G=1,TLB条目会被标记为global , 在TLB缓存驻留,不会被刷新 )**

当成功建立TLB entry后,在访问虚拟内存就不会进行查表操作,而是查TLB entry,以此在增加效率

## 2. 那么在HOOK函数时会导致什么问题呢?

注:一般在涉及内存操作才会去查TLB表,因此在前面的文章中,我们使用MOV RAX,0x12345678 jmp mov 就不会查TLB,因此不会蓝屏,如果使用jmp 0x12345678 就会去查TLB表,

为什么会蓝屏:原本JMP 0x12345678 ,CPU执行的指令是查0x12345678上面的值,原本0x12345678上面的值应该是一个地址,但是此时0x12345678上面的值被我们修改掉了,导致蓝屏

我们过PG的HOOK是通过修改PML4T来进行的,但是由于TLB entry的存在,导致无法保证系统运行的函数是被我们HOOK过的,因此我们需要主动刷新TLB

但同时出现一个新的问题,每一个CPU核心都有一个TLB entry,就导致我们在刷新TLB的时候,需要对每个核心都进行刷新

## 3. 开了TLB机制的标志是什么

KPTI: 将内核空间和用户空间的页表分离，每次在用户模式和内核模式之间切换时都需要切换页表。

KPTI开了,CR3.PCID才会有值
 
CR3.PCID=1 的时候开起来PCIE

KPTI开了之后 ,刷新TLB的时候 catch也会同步消失

## 4. 如何主动刷新TLB

如果刷新了TLB ,Paging-structure Catche也会同时刷新

1. Intel提供了INVLPG 函数,一个参数(线性地址),会无效所有与该线性地址相关的TLB,如果开了KPTI,INVLPG无效内核层内存的时候只能无效内核层,无法无效用户层
2. 如果mov cr3,会刷新TLB,但不包括global
3. 修改Cr4的PGE位,会刷新所有TLB条目和global条目,Cr4的PGE是选择启动global page配置
4. 由于是每个核心一个CR4,所以要修改每个核心的CR4,有一个函数可以让每个核心都运行同一个函数(KeIpiGenericCall)


## 5. 如何实现

**写一个函数修改CR4**

```C++

ULONG_PTR KipiBroadcastWorker(
	ULONG_PTR Argument
)
{
	UNREFERENCED_PARAMETER(Argument);
	KIRQL irql = KeRaiseIrqlToDpcLevel();//提升当前进程特权级
	_disable(); //屏蔽中断
	ULONG64 cr4 = __readcr4(); 
	cr4 &= 0xffffffffffffff7f;
	__writecr4(cr4);
	_enable();
	KeLowerIrql(irql);

	return 0;

}
```

**用KeIpiGenericCall调用那个函数**

```C++
KeIpiGenericCall(KipiBroadcastWorker, NULL);
```

**Windows 提供了一个函数刷新TLB,但是没有给符号和导出需要自己声明**

```C++
EXTERN_C VOID
KeFlushEntireTb(
	__in BOOLEAN Invalid,
	__in BOOLEAN AllProcessors
);

KeFlushEntireTb(true, false); // 使用
```

