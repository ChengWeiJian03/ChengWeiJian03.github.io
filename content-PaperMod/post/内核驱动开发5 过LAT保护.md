---
title: 内核驱动开发5 过LAT保护
published: 2024-11-21T22:06:26+08:00
summary: "过LAT保护"
tags: [内核,LAT]
categories: '内核驱动'
draft: false 
lang: ''
---

关键字：LAT表(导入表)

## 什么是LAT保护

IAT表是编译文件的时候写在PE结构里的,记录了导出的函数名称,地址

VKG做了什么呢,他修改了LAT表

假设原本cheat的函数要从0xFFFF000ff000调用

VKG.sys的地址是0xFFFF000000000

当我们打印cheat函数的调用地址,就会发现地址是0xFFFF000000010,是从VKG内部调用,而不是原本的进程

导致所有的函数都会过一遍VKG的保护,如果有异常,就会被拦截

## 解决方案

把原本的函数写成函数指针,用函数指针调用原本的Windows函数 
```C++ 

typedef NTSTATUS(NTAPI* pfnNtOpenProcess)(PHANDLE, ACCESS_MASK, POBJECT_ATTRIBUTES, PCLIENT_ID);
typedef NTSTATUS(NTAPI* pfnNtCreateFile)(PHANDLE, ACCESS_MASK, POBJECT_ATTRIBUTES, PIO_STATUS_BLOCK, PLARGE_INTEGER, ULONG, ULONG, ULONG, ULONG, PVOID, ULONG);
typedef NTSTATUS(NTAPI* pfnNtWriteFile)(HANDLE, HANDLE, PIO_APC_ROUTINE, PVOID, PIO_STATUS_BLOCK, PVOID, ULONG, PLARGE_INTEGER, PULONG);

pfnNtOpenProcess g_OriNtOpenProcess;
pfnNtCreateFile g_OriNtCreateFile;
pfnNtWriteFile g_OriNtWriteFile;
```