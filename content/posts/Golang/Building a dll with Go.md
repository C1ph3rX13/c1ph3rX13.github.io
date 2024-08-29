---
title: Building a Dll With Go
date: 2024-05-13T10:39:48+08:00
draft: false
url: /posts/2024-05-13/Building-a-dll-with-Go
tags:
  - Golang
  - Dll
slug: English Preview
---
> Building a dll with Go
> <!--more-->

# Installation

### MingGW

https://www.mingw-w64.org/

https://sourceforge.net/projects/mingw-w64/files/mingw-w64/mingw-w64-release/
# Making DLLs

+ 导入 `C` 模块
+ 导出函数需要`//export`标记，注意`//`与`export`之间没有空格：`//export FuncName`

+ 需要`main`函数，可为空

```go
package main

import "C"
import (
 "fmt"
)

//export HelloWorld
func HelloWorld() {
 fmt.Println("Hello World!")
}

func main() {}
```

# Compiling the DLL

```cmd
go -o dllmain.dll -buildmode=c-shared dllmain.go

go build -ldflags="-s -w" -buildmode=c-shared -o dllmain.dll -trimpath dllmain.go
```

# To load and Execute a DLL

导入`.h`调用

```go
package main

// #cgo LDFLAGS: -L. -ldllmain
// #include "dllmain.h"
import "C"

func main() {
 C.HelloWorld()
}
```

# Calling DLL

想要触发`init()`，需要导出一个函数作为类似`main()`的初始化触发点

```go
package main

import (
	"C"
	"fmt"
)

func init() {
	// evil code
}

//export EXPORTED_FUNCTION_NAME
func <DLL EXPORT FUNCTION>() {
    // 必须要有触发点才能执行 init()，内容可以为空
    // 当然也可以在导出函数中执行
}

func main() {}
```

`syscall.SyscallN() ` 触发

```go
package main

import (
	"syscall"

	"golang.org/x/sys/windows"
)

func main() {
	handle, err := windows.LoadLibrary("<DLL NAME>")
	if err != nil {
		panic(err)
	}

	proc, err := windows.GetProcAddress(handle, "<DLL EXPORT FUNCTION>")
	if err != nil {
		panic(err)
	}

	syscall.SyscallN(uintptr(proc))
}
```

`LoadDLL()` 触发

```go
package main

import (
	"golang.org/x/sys/windows"
)

func main() {
	handle, err := windows.LoadDLL("<DLL NAME>")
	if err != nil {
		panic(err)
	}

	proc, err := handle.FindProc("<DLL EXPORT FUNCTION>")
	if err != nil {
		panic(err)
	}

	proc.Call()
}
```

# Bypass

```go
	cmd := exec.Command("powershell.exe")

	cmd := exec.Command(strings.ToLower("%s%s%s","Po","wEr","sHEL","l.exE"))
```

