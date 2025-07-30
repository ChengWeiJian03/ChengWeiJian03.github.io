---
title: CSGO逆向3 写出一个透视
published: 2022-09-27
description: "我们将结合上面两篇文章来写出一个简单的透视"
weight: 1
toc: false
tags: [逆向, CSGO]
---
本篇内容将包括
1. 透视实现的方法介绍
2. 通过进程名获取进程id和进程句柄
3. 通过进程id获取进程中的模块信息（模块大小，模块地址，模块句柄）
4. 读取游戏内存（人物ViewMatrix，敌人坐标，敌人生命值，敌人阵营）
5. 三维坐标转二维坐标（游戏内人物坐标转换成屏幕上的坐标）
6. glfw+imgui 在屏幕上的绘制直线

2025年看到文章底下的评论还是蛮感慨的,这段时间把cs2的写成博客发出来...

<!--more-->

 **实现效果：**

![1](../images/1996357-20221012220247734-1550113641.png)



## 透视实现的方法介绍

一般有两种方式，一种是外挂，一种是内挂，外挂是在创建一个透明窗口，在透明窗口上画线，让鼠标事件透过窗口，透明窗口覆盖在游戏窗口上。内挂是通过DLL注入，HOOK游戏中的绘制函数，在游戏绘制人物的时候绘制自己的线。还剩一种比较少用，但也可以实现，找到人物模型ID，在渲染到人物模型的时候关掉渲染缓冲（应该是叫这个？），使人物模型在墙模型前面渲染，导致可以直接看到人物。本篇文章采用的是外挂的形式，根据上篇文章已经可以创建出一个覆盖在屏幕上的透明窗口。



## 工具函数以及必备变量

变量名起的挺明白的，就不写注释了

```C++
DWORD g_process_id = NULL;
HANDLE g_process_handle = NULL;
UINT_PTR g_local_player = NULL;
UINT_PTR g_player_list_address = NULL;
UINT_PTR g_matrix_address = NULL;
UINT_PTR g_angle_address = NULL;
HWND g_game_hwnd = NULL;
module_information engine_module;
module_information client_module;
module_information server_module;
float g_client_width;
float g_client_height;
```

**把需要用到的偏移也声明一下**

```C++
#define dwViewMatrix 0x4DCF254
#define dwLocalPlayer 0xDC14CC
#define dwClientState 0x58CFDC
#define dwEntityList 0x4DDD93C
#define dwClientState_ViewAngles 0x4D90

#define m_vecOrigin 0x138
#define m_bDormant 0xED
#define m_lifeState 0x25F
#define m_iHealth 0x100
#define m_iTeamNum 0xF4
```

**函数:获取屏幕大小，保存到全局变量**

```C++
void GetWindowSize()
{
    HDC hdc = GetDC(nullptr);
    g_client_width = GetDeviceCaps(hdc, DESKTOPHORZRES);
    g_client_height = GetDeviceCaps(hdc, DESKTOPVERTRES);
    ReleaseDC(nullptr, hdc);
}
```

**函数:先写一个错误获取函数，以方便获取出错的信息**

```C++
void error(const char*text)
{
    MessageBoxA(nullptr, text, nullptr, MB_OK);
    exit(-1);
}
bool is_error()
{
    return GetLastError() != 0;
}
```

**函数:通过进程名获取进程id和进程句柄**

```C++
# 使用CreateToolhelp32Snapshot函数，创建进程快照，遍历系统快照中的进程名，遍历到process_name,则返回该进程的进程ID
DWORD get_process_id(const char*process_name)
{
    HANDLE snap = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);
    if (is_error()) error("CreateToolhelp32Snapshot失败");
    PROCESSENTRY32 process_info;
    ZeroMemory(&process_info, sizeof(process_info));
    process_info.dwSize = sizeof(process_info);
    char target[1024];
    ZeroMemory(target, 1024);
    strncpy_s(target, process_name, strlen(process_name));
    _strupr(target);
    bool state = Process32First(snap, &process_info);
    while (state)
    {
        if (strncmp(_strupr(process_info.szExeFile), target, strlen(target)) == 0)
        {
            return process_info.th32ProcessID;
        }
        state = Process32Next(snap, &process_info);
    }
    CloseHandle(snap);
    return 0;
}
```

**函数:通过进程ID获取进程句柄**

```C++
HANDLE get_process_handle(DWORD process_id)
{
    HANDLE process_handle = OpenProcess(PROCESS_ALL_ACCESS, FALSE, process_id);
    if (is_error())
        error("get_process_handle失败");
return process_handle;
}
```

**函数:通过进程id获取进程中的模块信息（模块大小，模块地址，模块句柄）**

```C++
#可以发现偏移都是由 client.dll+xxxxx 此种形式构成，所以需要获取模块的地址

#先创建一个模块结构体，需要获取模块的模块大小，模块地址，模块句柄

class module_information
{
public:
    HANDLE module_handle;
    char module_name[1024];
    char *module_data;
    UINT_PTR module_address;
    int module_size;
    void alloc(int size)
    {
        module_size = size;
        module_data = (char *)VirtualAlloc(nullptr, size, MEM_COMMIT | MEM_RESERVE, PAGE_EXECUTE_READWRITE);
        if (is_error())error("申请内存失败");
    }
    void release()
    {
        if (module_data)VirtualFree(module_data, 0, MEM_RELEASE);
        module_data = nullptr;
    }
};
```

**函数:传入进程ID和需要获取的模块名，CreateToolhelp32Snapshot创建模块快照，遍历快照，比对模块名，获取模块信息**

```C++
void get_moduel_info(DWORD process_id, const char *name, OUT module_information&info)
{
    HANDLE snap = CreateToolhelp32Snapshot(TH32CS_SNAPMODULE, process_id);
    if (is_error())error("创建快照错误");
    MODULEENTRY32 module_info;
    ZeroMemory(&module_info, sizeof(module_info));
    module_info.dwSize = sizeof(module_info);
    char target[1024];
    ZeroMemory(target, 1024);
    strncpy(target, name, strlen(name));
    _strupr(target);
    bool status = Module32First(snap, &module_info);
    while (status)
    {
        if (strncmp(_strupr(module_info.szModule), target, sizeof(target)) == 0)
        {
            info.module_address = (UINT_PTR)module_info.modBaseAddr;
            info.module_handle = module_info.hModule;
            info.alloc(module_info.modBaseSize);
            DWORD size = read_memory(g_process_handle, info.module_address, info.module_data, info.module_size);//TODO
            CloseHandle(snap);
            return;
        }
        status = Module32Next(snap, &module_info);
    }
    error("未找到模块");
    return;
}
```

**函数:读取游戏内存函数**

例如之前得到 上下角度 = [[engine.dll+58CFDC]+00004D90] ，则可以 

ReadProcessMemory(g_process_handle, (LPVOID)(engine.dll+58CFDC), recv, size, &readsize);

ReadProcessMemory(g_process_handle, (LPVOID)recv, recv, size, &readsize);

函数的使用方法：ReadProcessMemory(句柄,地址,读到哪里,读多少,具体读了多少);

则可以读到上下角度

通过ReadProcessMemory函数读取内存，对这个函数进行打包，方便使用（好吧，我承认这个打包的很烂，几乎没有方便使用）

```C++
DWORD read_memory(HANDLE process, DWORD address, void *recv, int size)
{
    DWORD readsize;
    ReadProcessMemory(process, (LPVOID)address, recv, size, &readsize);
    return readsize;
    if (is_error())error("读取内存失败");
}
```

重写了一个我觉得比较好用的，各位可以酌情对其进行改写

```C++
template<class T>
T ReadMem(HANDLE ProcessHandle, UINT_PTR Address, int size)
{
    T Reader;
    ReadProcessMemory(ProcessHandle, (LPVOID)Address, &Reader, size, NULL);
    return Reader;
}
```

**函数:三维坐标转二维坐标**

创建两个结构体来储存二维坐标，一个用来储存三维坐标

```C++
struct Vec2
{public:
    float x, y;
};
struct Vec3
{
public:
    float x, y, z;
};
```

传入一个三维坐标和视角矩阵，算出人物在屏幕上的坐标 VecScreen

```C++
bool WorldToScreen(const Vec3& VecOrgin, Vec2& VecScreen, float* Matrix)
{
    VecScreen.x = VecOrgin.x *Matrix[0] + VecOrgin.y*Matrix[1] + VecOrgin.z*Matrix[2] + Matrix[3];
    VecScreen.y = VecOrgin.x *Matrix[4] + VecOrgin.y*Matrix[5] + VecOrgin.z*Matrix[6] + Matrix[7];
    float w = VecOrgin.x*Matrix[12] + VecOrgin.y*Matrix[13] + VecOrgin.z*Matrix[14] + Matrix[15];
    if (w < 0.01f)
    {
        return false;
    }
    Vec2 NDC;
    NDC.x = VecScreen.x / w;
    NDC.y = VecScreen.y / w;
    VecScreen.x = (g_client_width / 2 * NDC.x) + (NDC.x + g_client_width / 2);
    VecScreen.y = (g_client_height / 2 * NDC.y) + (NDC.y + g_client_height / 2);
    ConvertToRange(VecScreen);
    return true;
}
void ConvertToRange(Vec2 &Point)
{
    Point.x /= g_client_width;
    Point.x *= 2.0f;
    Point.x -= 1.0f;
    Point.y /= g_client_height;
    Point.y *= 2.0f;
    Point.y -= 1.0f;
}
```

**函数:GLFW画线**

```C++
void DrawLine(Vec2& start, Vec2& end)
{
    glLineWidth(1.2);

    glBegin(GL_LINES);

    glColor4f(255, 255, 255, 100);
    glVertex2f(start.x, start.y);
    glVertex2f(end.x, end.y);
    glEnd();
}
```

**函数:写一个init函数，实现初始化**

```C++
void init_address(const char*process_name)
{
    std::cout << "请先启动游戏"<< std::endl;

    DWORD process_id = get_process_id(process_name);
    HANDLE process_handle = get_process_handle(process_id);
    g_process_id = process_id; //将pid保存到全局变量
    g_process_handle = process_handle;//将process_handle保存到全局变量
    //获取模块信息
    get_moduel_info(process_id, "engine.dll", engine_module);
    get_moduel_info(process_id, "client.dll", client_module);
    get_moduel_info(process_id, "server.dll", server_module);

    UINT_PTR temp_address;
    float Matrix[16];
    UINT_PTR matrix_address = client_module.module_address + dwViewMatrix; //获取视角矩阵地址
    g_matrix_address = matrix_address; //将视角矩阵地址保存到全局变量

    //获取人物视角地址
    ReadProcessMemory(g_process_handle, (LPVOID)(engine_module.module_address + 0x58CFDC), &temp_address, 4, NULL);//[engine.dll + 58CFDC]+00004D90
    g_angle_address = temp_address + dwClientState_ViewAngles;

    //获取本地人物地址 [client.dll+0xDC04CC]+100 = 生命值
    ReadProcessMemory(g_process_handle, (LPVOID)(client_module.module_address + dwLocalPlayer), &temp_address, 4, NULL);
    g_local_player = temp_address; //[g_local_player+100] = 生命值

    //获得ENtitylist地址  [client.dll+0x4DDC90C + i *0x10]+100 = 敌人生命值
     g_player_list_address = client_module.module_address + dwEntityList;
}
```



## 透视实现思路

 通过进程名（csgo.exe）获取进程ID

​			↓

通过进程ID获取进程句柄、client.dll模块的信息

​			↓

 通过进程句柄读取人物视角矩阵地址、本地人物对象地址、敌人对象地址 并保存到全局变量（初始化完成）

​			↓

 获得屏幕大小储存在全局变量、创建透明窗口

​			↓

循环遍历敌人对象，通过地址读取到人物的视角矩阵、敌人的位置

​			↓

 在循环中将敌人的位置结合矩阵，转换成2D坐标

​			↓

 再循环中在透明窗口上把算出来的坐标画出来



**再写一段伪代码出来帮助理解，代码贴在后面**

```C++
int main
{
    获取视角矩阵地址、获取本地人物地址、获取敌人对象地址
    获取屏幕分辨率
    根据屏幕分辨率创建窗口
    while（1）消息循环
    {　　　 清除画的线
      获得视角矩阵，因为会变，所以需要不停的获取
      for(int i=0;i<64;i++)因为游戏人数最大为64
        {
            获得自己的阵营
            获取当前敌人对象
            根据对象获取人物血量、阵营、生存状态、敌人是否有效
            如果敌人血量<=0 或者 敌人阵营=自己阵营 或者 无效 或者 敌人对象为空 或者 敌人生存状态则遍历下一个对象
            获得敌人的位置
            将敌人的坐标转换为2D坐标
            画线
          }
    }
}
```



## 实现代码

**GetImformation.h**

```C++
#pragma once
struct Vec2
{public:
    float x, y;

};
struct Vec3
{
public:
    float x, y, z;

};
bool is_error();
void error(const char*text);
class module_information
{
public:
    HANDLE module_handle;
    char module_name[1024];
    char *module_data;
    UINT_PTR module_address;
    int module_size;
    void alloc(int size)
    {
        module_size = size;
        module_data = (char *)VirtualAlloc(nullptr, size, MEM_COMMIT | MEM_RESERVE, PAGE_EXECUTE_READWRITE);
        if (is_error())error("申请内存失败");
    }
    void release()
    {
        if (module_data)VirtualFree(module_data, 0, MEM_RELEASE);
        module_data = nullptr;
    }
};
void init_address(const char*process_name);
DWORD get_process_id(const char*process_name);

HANDLE get_process_handle(DWORD process_id);

void ConvertToRange(Vec2 &Point);
bool WorldToScreen(const Vec3& VecOrgin, Vec2& VecScreen, float* Matrix);
void get_moduel_info(DWORD process_id, const char *name, OUT module_information&info);
DWORD read_memory(HANDLE process, DWORD address, void *recv, int size);

template<class T>
T ReadMem(HANDLE ProcessHandle, UINT_PTR Address, int size)
{
    T Reader;
    ReadProcessMemory(ProcessHandle, (LPVOID)Address, &Reader, size, NULL);
    return Reader;
}
```

**GetImformation.cpp**

```C++
#include<Windows.h>
#include<TlHelp32.h>
#include"GetIMformation.h"


DWORD g_process_id = NULL;
HANDLE g_process_handle = NULL;
UINT_PTR g_local_player = NULL;
UINT_PTR g_player_list_address = NULL;
UINT_PTR g_matrix_address = NULL;
UINT_PTR g_angle_address = NULL;
HWND g_game_hwnd = NULL;
module_information engine_module;
module_information client_module;
module_information server_module;
float g_client_width;
float g_client_height;

#define dwViewMatrix 0x4DCF254
#define dwLocalPlayer 0xDC14CC
#define dwClientState 0x58CFDC
#define dwEntityList 0x4DDD93C
#define dwClientState_ViewAngles 0x4D90

#define m_vecOrigin 0x138
#define m_bDormant 0xED
#define m_lifeState 0x25F
#define m_iHealth 0x100
#define m_iTeamNum 0xF4
//获取模块信息
void get_moduel_info(DWORD process_id, const char *name, OUT module_information&info)
{
    HANDLE snap = CreateToolhelp32Snapshot(TH32CS_SNAPMODULE, process_id);
    if (is_error())error("创建快照错误");
    MODULEENTRY32 module_info;
    ZeroMemory(&module_info, sizeof(module_info));
    module_info.dwSize = sizeof(module_info);
    char target[1024];
    ZeroMemory(target, 1024);
    strncpy(target, name, strlen(name));
    _strupr(target);
    bool status = Module32First(snap, &module_info);
    while (status)
    {
        if (strncmp(_strupr(module_info.szModule), target, sizeof(target)) == 0)
        {
            info.module_address = (UINT_PTR)module_info.modBaseAddr;
            info.module_handle = module_info.hModule;
            info.alloc(module_info.modBaseSize);
            //DWORD size = read_memory(g_process_handle, info.module_address);//TODO
            DWORD size = read_memory(g_process_handle, info.module_address, info.module_data, info.module_size);//TODO
            CloseHandle(snap);
            return;
        }
        status = Module32Next(snap, &module_info);
    }
    error("未找到模块");
    return;
}
void error(const char*text)
{
    MessageBoxA(nullptr, text, nullptr, MB_OK);
    exit(-1);
}
bool is_error()
{
    return GetLastError() != 0;
}
DWORD read_memory(HANDLE process, DWORD address, void *recv, int size)
{
    DWORD readsize;
    ReadProcessMemory(process, (LPVOID)address, recv, size, &readsize);
    return readsize;
    if (is_error())error("读取内存失败");
}

HANDLE get_process_handle(DWORD process_id)
{
    HANDLE process_handle = OpenProcess(PROCESS_ALL_ACCESS, FALSE, process_id);
    if (is_error())
        error("get_process_handle失败");
    std::cout << "进程句柄为：" << std::hex << process_handle << std::endl;
    return process_handle;
}
DWORD get_process_id(const char*process_name)
{
    HANDLE snap = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);
    if (is_error()) error("CreateToolhelp32Snapshot失败");
    PROCESSENTRY32 process_info;
    ZeroMemory(&process_info, sizeof(process_info));
    process_info.dwSize = sizeof(process_info);
    char target[1024];
    ZeroMemory(target, 1024);
    strncpy_s(target, process_name, strlen(process_name));
    _strupr(target);
    bool state = Process32First(snap, &process_info);
    while (state)
    {
        if (strncmp(_strupr(process_info.szExeFile), target, strlen(target)) == 0)
        {

            CloseHandle(snap);
            return process_info.th32ProcessID;
        }
        state = Process32Next(snap, &process_info);
    }
    CloseHandle(snap);
    MessageBoxA(NULL, "查找进程id失败", "提示", MB_OK);
    return 0;
}
void GetWindowSize()
{
    HDC hdc = GetDC(nullptr);
    g_client_width = GetDeviceCaps(hdc, DESKTOPHORZRES);
    g_client_height = GetDeviceCaps(hdc, DESKTOPVERTRES);
    ReleaseDC(nullptr, hdc);
}

void init_address(const char*process_name)
{
    std::cout << "请先启动游戏"<< std::endl;

    DWORD process_id = get_process_id(process_name);
    HANDLE process_handle = get_process_handle(process_id);
    g_process_id = process_id;

    //获取模块信息
    g_process_handle = process_handle;

    get_moduel_info(process_id, "engine.dll", engine_module);
    get_moduel_info(process_id, "client.dll", client_module);
    get_moduel_info(process_id, "server.dll", server_module);


    //TODO 要写一个特征码寻址，获取视角矩阵信息
    UINT_PTR temp_address;
    float Matrix[16];
    UINT_PTR matrix_address = client_module.module_address + dwViewMatrix;
    g_matrix_address = matrix_address;

    //获取 人物视角地址
    ReadProcessMemory(g_process_handle, (LPVOID)(engine_module.module_address + 0x58CFDC), &temp_address, 4, NULL);//[engine.dll + 58CFDC]+00004D90
    g_angle_address = temp_address + 0x00004D90;

    //获取本地人物地址 [client.dll+0xDC04CC]+100 = 生命值
    ReadProcessMemory(g_process_handle, (LPVOID)(client_module.module_address + dwLocalPlayer), &temp_address, 4, NULL);
    g_local_player = temp_address; //[g_local_player+100] = 生命值
    temp_address = 0;

    //获得ENtitylist地址  [client.dll+0x4DDC90C + i *0x10]+100 = 敌人生命值
     g_player_list_address = client_module.module_address + dwEntityList;
}

bool WorldToScreen(const Vec3& VecOrgin, Vec2& VecScreen, float* Matrix)
{
    VecScreen.x = VecOrgin.x *Matrix[0] + VecOrgin.y*Matrix[1] + VecOrgin.z*Matrix[2] + Matrix[3];
    VecScreen.y = VecOrgin.x *Matrix[4] + VecOrgin.y*Matrix[5] + VecOrgin.z*Matrix[6] + Matrix[7];
    float w = VecOrgin.x*Matrix[12] + VecOrgin.y*Matrix[13] + VecOrgin.z*Matrix[14] + Matrix[15];
    if (w < 0.01f)
    {
        return false;
    }
    Vec2 NDC;
    NDC.x = VecScreen.x / w;
    NDC.y = VecScreen.y / w;
    VecScreen.x = (g_client_width / 2 * NDC.x) + (NDC.x + g_client_width / 2);
    VecScreen.y = (g_client_height / 2 * NDC.y) + (NDC.y + g_client_height / 2);
    ConvertToRange(VecScreen);
    return true;
}
void ConvertToRange(Vec2 &Point)
{
    Point.x /= g_client_width;
    Point.x *= 2.0f;
    Point.x -= 1.0f;
    Point.y /= g_client_height;
    Point.y *= 2.0f;
    Point.y -= 1.0f;
}
```
**main.cpp**

```C++
#include <stdio.h>
#include<cstdlib>
#include<Windows.h>
#include<iostream>
#include<cmath>

#include <GLFW/glfw3.h>
#include "imgui/imgui.h"
#include "imgui/imgui_impl_glfw.h"
#include "imgui/imgui_impl_opengl3.h"

#include "GetIMformation/GetIMformation.h"


//声明外部变量
extern DWORD g_process_id;
extern HANDLE g_process_handle;
extern UINT_PTR g_local_player;
extern UINT_PTR g_player_list_address;
extern UINT_PTR g_matrix_address;
extern UINT_PTR g_angle_address;
extern HWND g_game_hwnd;
extern module_information engine_module;
extern module_information client_module;
extern module_information server_module;
extern float g_client_width;
extern float g_client_height;

void DrawLine(Vec2& start, Vec2& end)
{
    glLineWidth(1.2);

    glBegin(GL_LINES);

    glColor4f(255, 255, 255, 100);
    glVertex2f(start.x, start.y);
    glVertex2f(end.x, end.y);
    glEnd();
}
void ShowMenu(GLFWwindow* Window)
{
    glfwSetWindowAttrib(Window, GLFW_MOUSE_PASSTHROUGH, GLFW_FALSE);
}

void HideMenu(GLFWwindow* Window)
{
    glfwSetWindowAttrib(Window, GLFW_MOUSE_PASSTHROUGH, GLFW_TRUE);
}

static void glfw_error_callback(int error, const char* description)
{
    fprintf(stderr, "Glfw Error %d: %s\n", error, description);
}

void GetWindowSize()
{
    HDC hdc = GetDC(nullptr);
    g_client_width = GetDeviceCaps(hdc, DESKTOPHORZRES);
    g_client_height = GetDeviceCaps(hdc, DESKTOPVERTRES);
    ReleaseDC(nullptr, hdc);
}


int main(int, char**)
{
    /////////////////////////功能性代码////////////////////////////////////////////////////////////////////////////////////////////
    GetWindowSize();
    init_address("csgo.exe");
    UINT_PTR temp_address;

    /////////////////////////功能性代码////////////////////////////////////////////////////////////////////////////////////////////


    // Setup window
    glfwSetErrorCallback(glfw_error_callback);
    if (!glfwInit())
        return 1;
    GLFWmonitor* monitor = glfwGetPrimaryMonitor();


    //###########################设置窗口###########################
    auto glsl_version = "#version 130";
    int Height = glfwGetVideoMode(monitor)->height;
    int Width = glfwGetVideoMode(monitor)->width;
    glfwWindowHint(GLFW_FLOATING, true);
    glfwWindowHint(GLFW_RESIZABLE, false);
    glfwWindowHint(GLFW_MAXIMIZED, true);
    glfwWindowHint(GLFW_TRANSPARENT_FRAMEBUFFER, true);
    //###########################设置窗口###########################


    GLFWwindow* window = glfwCreateWindow(Width, Height, "titile", nullptr, nullptr);
    if (window == nullptr)
        return 1;
    glfwSetWindowAttrib(window, GLFW_DECORATED, false); //设置没有标题栏
    ShowWindow(GetConsoleWindow(), SW_HIDE);
    glfwMakeContextCurrent(window);
    glfwSwapInterval(1);
    IMGUI_CHECKVERSION();
    ImGui::CreateContext();
    ImGuiIO& io = ImGui::GetIO();
    (void)io;
    ImGui::StyleColorsDark();
    ImGui_ImplGlfw_InitForOpenGL(window, true);
    ImGui_ImplOpenGL3_Init(glsl_version);


    bool bMenuVisible = true;
    bool Dormant;

    int EntityTeamNum;
    int lifestate;
    int blood;
    int iTeamNum;

    float temp_pos[3];
    float Matrix[16];

    Vec2 LineOrigin;
    Vec2 ScreenCoord;
    Vec3 EntityLocation;

    LineOrigin.x = 0.0f;
    LineOrigin.y = -1.0f;

    UINT_PTR Entity;

    while (!glfwWindowShouldClose(window))
    {
        glfwPollEvents();
        glClear(GL_COLOR_BUFFER_BIT);
        ImGui_ImplOpenGL3_NewFrame();
        ImGui_ImplGlfw_NewFrame();
        ImGui::NewFrame();
    
        if (GetAsyncKeyState(VK_F11) & 1)
        {
            bMenuVisible = !bMenuVisible;
            if (bMenuVisible)
                ShowMenu(window);
            else
                HideMenu(window);
        }
        
        //界面设计
        if (bMenuVisible)
        {
            ImGui::Text("USE F11 TO Hiden/Show");
            ImGui::Text("");
            if (ImGui::Button("exit")) return 0;
        }
        ReadProcessMemory(g_process_handle, (LPVOID)(client_module.module_address + dwLocalPlayer),
            &g_local_player, 4, nullptr);
        if(g_local_player!=0)
        {
            
            ScreenCoord.x = 0.0f;
            ScreenCoord.y = -1.0f;
            g_angle_address = ReadMem<UINT_PTR>(g_process_handle, (engine_module.module_address + dwClientState), 4)+ dwClientState_ViewAngles;
            ReadProcessMemory(g_process_handle, (LPCVOID)(client_module.module_address + dwViewMatrix), Matrix,
                              sizeof(float) * 16, nullptr);
            for (short int i = 0; i < 64; ++i)
            {
                ReadProcessMemory(g_process_handle, (LPVOID)(client_module.module_address + dwLocalPlayer),
                    &g_local_player, 4, nullptr);

                ReadProcessMemory(g_process_handle, (LPCVOID)(g_local_player + m_iTeamNum), &iTeamNum, 4, nullptr);

                //获取敌人实体
                ReadProcessMemory(g_process_handle, (LPCVOID)(client_module.module_address + dwEntityList + i * 0x10),
                    &Entity, sizeof(float), nullptr);

                ReadProcessMemory(g_process_handle, (LPVOID)(Entity + m_bDormant), &Dormant, sizeof(bool), nullptr);
                ReadProcessMemory(g_process_handle, (LPVOID)(Entity + m_lifeState), &lifestate, 4, nullptr);
                ReadProcessMemory(g_process_handle, (LPCVOID)(Entity + m_iTeamNum), &EntityTeamNum, 4, nullptr);
                ReadProcessMemory(g_process_handle, (LPCVOID)(Entity + m_iHealth), &blood, 4, nullptr);

                if ((Entity == NULL) || (Entity == g_local_player) || (EntityTeamNum == iTeamNum) || (blood <= 0) ||
                    lifestate || Dormant)
                    continue;

                ReadProcessMemory(g_process_handle, (LPVOID)(Entity + m_vecOrigin), &temp_pos, 12, nullptr);
                EntityLocation.x = temp_pos[0], EntityLocation.y = temp_pos[1], EntityLocation.z = temp_pos[2];

                if (!WorldToScreen(EntityLocation, ScreenCoord, Matrix))
                    continue;

                if (true)
                {
                    DrawLine(LineOrigin, ScreenCoord);
                }
            }
        }
        // Rendering
        ImGui::Render();
        int display_w, display_h;
        glfwGetFramebufferSize(window, &display_w, &display_h);
        glViewport(0, 0, display_w, display_h);
        ImGui_ImplOpenGL3_RenderDrawData(ImGui::GetDrawData());
        glfwSwapBuffers(window);
    }

    // Cleanup
    ImGui_ImplOpenGL3_Shutdown();
    ImGui_ImplGlfw_Shutdown();
    ImGui::DestroyContext();
    glfwDestroyWindow(window);
    glfwTerminate();

    return 0;
}
```





## 至此一个简单的透视就写完了，本系列也完结力....



后记：

- 感觉有些地方逻辑讲的不是很清晰，有问题可以在评论区里提
- 里面写的偏移全都是写死的，等CSGO一更新就不能用了，解决这个问题的方法，就是写一个特征码查找，但不能保证所有人都是来学思路和方法的，对于直接copy代码的，起码偏移要自己找一下。
- 方框、自瞄、防闪其实也写了，但感觉不是很适合在这里发，如果后面想发了，再写一个补充篇吧
- 因为是从一堆功能中抽出一个小透，所以可能报错，少点什么东西，能自己补的自己补一下，不能的评论区说一下，我来补

