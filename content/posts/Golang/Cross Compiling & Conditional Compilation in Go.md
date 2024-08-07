---
title: "Cross Compiling & Conditional Compilation in Go"
date: 2023-09-13T18:01:32+08:00
draft: false
url: /posts/2023-09-13/Cross-Compiling&Conditional-Compilation-in-Go
tags: ["Golang","Cross Compiling","Conditional Compilation"]
---

## 0x00 前言

Cross Compiling / Conditional Compilation in Go

## 0x01 Cross Compiling

### 编译参数

#### GOOS

目标平台的操作系统（Darwin、Freebsd、Linux、Windows） 

#### GOARCH

目标平台的体系架构（386、amd64、arm） 交叉编译不支持 CGO 所以要禁用

### 编译平台

#### Linux

Linux 下编译 Mac 和 Windows 64位可执行程序

```bash
CGO_ENABLED=0 GOOS=darwin GOARCH=amd64 go build -ldflags="-s -w" -trimpath -o <File Name>
CGO_ENABLED=0 GOOS=windows GOARCH=amd64 go build -ldflags="-s -w" -trimpath -o <File Name>
```

#### Windows

Windows 下编译 Mac 和 Linux 64位可执行程序

```cmd
SET CGO_ENABLED=0
SET GOOS=darwin
SET GOARCH=amd64
go build -ldflags="-s -w" -trimpath -o <File Name>

SET CGO_ENABLED=0
SET GOOS=linux
SET GOARCH=amd64
go build -ldflags="-s -w" -trimpath -o <File Name>
```

#### Mac

Mac 下编译 Linux 和 Windows 64位可执行程序

```bash
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -ldflags="-s -w" -trimpath -o <File Name>
CGO_ENABLED=0 GOOS=windows GOARCH=amd64 go build -ldflags="-s -w" -trimpath -o <File Name>
```

## 0x02 Conditional Compilation

### Build tags 构建标签

#### 源码中插入注释

第一种实现条件编译的方法是在源码中插入注释，被称之为构建标签

构建标签的注释应该尽可能的接近源码文件的顶部位置

当Go编译一个包时，它会分析包内的每个源码文件并查找构建标签。标签决定了这个源码文件是否被编译

构建标签遵循以下三个原则：

1. 空格隔开的选项是或（OR）的关系
2. 逗号隔开的选项是与（AND）的关系
3. 每个选项由字母和数字组成。如果前面加上`!`，则表示反义

```go
// +build darwin freebsd netbsd openbsd
```

上面的例子，构建标签出现在某个源码文件的顶部，表示这个源码文件只会在支持kqueue的BSD系统中被编译

#### 多个构建标签

一个源码文件可以包含多个构建标签。构建规则是每个独立规则的逻辑与关系

如下例子表示该文件将在linux/386或darwin/386平台才会被编译；注意：构建标签和包声明间必须要有空行，否则会产生报错

```go
// +build linux darwin
// +build 386

package main
```

### File suffixes 文件名后缀

第二种条件编译的方法是通过源码文件的文件名实现的，这种方案比构造标签方案更简单

`go/build`包的文档有关于命名约定的描述。简单来说，如果文件名包含`_$GOOS.go`后缀，那么这个源码文件只会在对应的平台被编译。其他平台会忽略这个文件。另一种约定是`_$GOARCH.go`

这两种后缀可以组合起来，但要保证顺序正确：

```go
// 正确的格式
_$GOOS_$GOARCH.go

// 错误的格式
_$GOARCH_$GOOS.go
```

以下是文件名后缀的一些例子：

```go
mypkg_freebsd_arm.go // 只在 freebsd/arm 系统编译
mypkg_plan9.go       // 只在 plan9 编译
```

即使是在Linux或freebsd系统，这两个文件也会被忽略，原因是`go/build`包会忽略所有文件名以`.`和`_`开始的文件

## 0x03 使用构建标签还是文件名后缀

构建标签和文件名后缀在功能上是重叠的

比如，一个名为`mypkg_linux.go`的文件，再包含构建标签`// +build linux`会显得多余

通常来说，当只有一个特定平台或体系需要指定时，我们选择文件名后缀的方式：

```go
mypkg_linux.go         // 只在 linux 系统编译
mypkg_windows_amd64.go // 只在 windows amd 64位 平台编译
```

相反，如果你的文件需要指定给多个平台或体系使用，或者你需要排除某个特定平台时，我们选择构建标签的方式：

```go
// 在所有类unix平台编译
% grep '+build' $HOME/go/src/pkg/os/exec/lp_unix.go
// +build darwin dragonfly freebsd linux netbsd openbsd

// 在非Windows平台编译
% grep '+build' $HOME/go/src/pkg/os/types_notwin.go
// +build !windows
```
