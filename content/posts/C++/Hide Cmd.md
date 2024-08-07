---
title: "Hide Cmd"
date: 2023-11-30T16:24:30+08:00
draft: false
url: /posts/2023-11-30/Hide-Cmd
tags: ["C++","Anti"]
---

## 0x00 前言

控制台程序的实现方法

## 0x01 控制台窗口程序

### 控制台窗口

使用 `GetConsoleWindow` 获取控制台窗口的句柄，然后使用 `ShowWindow` 函数将其隐藏

```C++
#include <windows.h>

int main() {
    HWND hwnd = GetConsoleWindow();
    ShowWindow(hwnd, SW_HIDE);

    // 在这里执行你的程序逻辑

    return 0;
}
```

### 子进程窗口隐藏

如果你的程序启动了一个子进程（例如，通过 `CreateProcess` 函数），你可以通过设置 `STARTUPINFO` 结构体的 `dwFlags` 成员来隐藏子进程的控制台窗口。

```C++
#include <windows.h>

int main() {
    STARTUPINFO si = { sizeof(STARTUPINFO) };
    PROCESS_INFORMATION pi;

    si.dwFlags = STARTF_USESHOWWINDOW;
    si.wShowWindow = SW_HIDE;

    // 在这里设置命令行参数等，然后启动子进程
    CreateProcess(NULL, "your_child_process.exe", NULL, NULL, FALSE, 0, NULL, NULL, &si, &pi);

    // 在这里执行你的程序逻辑

    return 0;
}
```

## 0x02 GUI程序的实现方法

在Windows上，`WinMain` 和 `wWinMain` 都是 Windows 应用程序的入口点。它们的主要区别在于字符集和参数类型。

1. **字符集：**

   - `WinMain` 通常用于 ANSI 字符集，即窄字符（`char`）。

   - `wWinMain` 用于 Unicode 字符集，即宽字符（`wchar_t`）。

1. **参数类型：**

   - `WinMain` 的参数使用 `LPSTR` 表示字符串，即 ANSI 字符串。

   - `wWinMain` 的参数使用 `LPWSTR` 表示字符串，即 Unicode 字符串。

### WinMain

在Windows API中，`CALLBACK` 是一个宏，通常用于标识回调函数的类型。这个宏的定义为：

```C++
#define CALLBACK __stdcall
```

`CALLBACK` 定义了回调函数的调用约定，即函数调用的规则。在Windows平台上，`__stdcall` 是一种常见的调用约定，也叫做标准调用约定。它规定了函数参数的传递方式和调用栈的清理方式。

在特定的情境下，使用正确的调用约定很重要。在Windows API中，很多回调函数都使用 `CALLBACK` 宏进行声明，以确保它们使用正确的调用约定，能够正确地被系统调用。

在下述代码中，`CALLBACK` 主要是为了符合 `WinMain` 函数的声明要求。`WinMain` 是Windows GUI应用程序的入口点，其声明如下：

```C++
int CALLBACK WinMain(
    HINSTANCE hInstance,
    HINSTANCE hPrevInstance,
    LPSTR lpCmdLine,
    int nCmdShow
);
```

使用 `CALLBACK` 宏帮助确保 `WinMain` 函数按照标准调用约定进行声明，以便与Windows API的期望相符。在这个上下文中，`CALLBACK` 实际上是 `__stdcall` 的别名。

第一种实现方式：

```C++
#include <Windows.h>

int CALLBACK WinMain(HINSTANCE hInstance, HINSTANCE hPrevInstance, LPSTR lpCmdLine, int nCmdShow) {
   // 函数实现

    return 0;
}
```

第二种实现方式：

```C++
#define WIN32_LEAN_AND_MEAN
#include <windows.h>

int WinMain(HINSTANCE hInstance, HINSTANCE hPrevInstance, LPSTR lpCmdLine, int nCmdShow) {
    HWND hwnd = GetConsoleWindow();
    ShowWindow(hwnd, SW_HIDE);
    
    // 函数实现
    
    return 0;
}
```

### wWinMain

1. `APIENTRY` 是一个宏，用于标识函数的调用约定。在 Windows API 中，`APIENTRY` 通常被定义为 `__stdcall`，这是一种常见的调用约定。

2. 调用约定规定了函数在被调用时，参数的传递方式、栈的清理方式等规则。在 Windows API 中，`__stdcall` 是默认的调用约定，也是大多数 Win32 API 函数的标准调用约定。在函数声明中使用 `APIENTRY` 或 `__stdcall` 用于确保函数按照标准调用约定进行编译。

```C++
int APIENTRY wWinMain(
    _In_ HINSTANCE hInstance,
    _In_opt_ HINSTANCE hPrevInstance,
    _In_ LPWSTR lpCmdLine,
    _In_ int nCmdShow
) {
    // 函数实现
    return 0;
}
```
