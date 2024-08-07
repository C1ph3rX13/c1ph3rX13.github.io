---
title: "Go Winres"
date: 2023-11-20T10:43:16+08:00
draft: false
url: /posts/2023-11-20/Go-Winres
tags: ["Golang","Go-Winres"]
---

## 0x00 前言

Go编译添加资源和程序信息

## 0x01 go-winres

### 项目地址

https://github.com/tc-hib/go-winres

### 安装

```powershell
go install github.com/tc-hib/go-winres@latest
```

### 初始化

在编译目录下初始化，生成一个配置模板。初始化会生成`winres`目录，其中`winres.json`为配置文件，两个`png`文件是自带的图标文件

```powershell
go-winres init
```

### 修改配置

#### 图标指定

1. 把图标放在和`winres.json`同级目录下
2. 修改`0000`这个属性中的值
3. 图片文件尺寸不能大于`256 x 256`

```json
"RT_GROUP_ICON": {
	"APP": {
		"0000": [
			"icon.png",  // 指定的图标文件，可修改为自定义图标
			"icon16.png"
		]
	}
}
```

#### 软件清单

软件清单部分声明在`RT_MANIFEST`属性中

```json
  "RT_MANIFEST": {
    "#1": {
      "0409": {
        "identity": {
          "name": "",
          "version": ""
        },
        "description": "",
        "minimum-os": "win7",
        "execution-level": "as invoker",
        "ui-access": false,
        "auto-elevate": false,
        "dpi-awareness": "system",
        "disable-theming": false,
        "disable-window-filtering": false,
        "high-resolution-scrolling-aware": false,
        "ultra-high-resolution-scrolling-aware": false,
        "long-path-aware": false,
        "printer-driver-isolation": false,
        "gdi-scaling": false,
        "segment-heap": false,
        "use-common-controls-v6": false
      }
    }
  }
```

上述并非所有属性都需要填写和修改，这里将重要的部分讲解一下：

- `description`：文件的描述信息
- `minimum-os`：最低要求操作系统
  - `"vista"`
  - `"win7"`
  - `"win8"`
  - `"win8.1"`
  - `"win10"`
- `execution-level`：应用程序所需的权限
  - `"as invoker"` 不需要任何权限
  - `"highest"` 使用当前用户的可用最高权限
  - `"administrator"` 强制要求管理员权限才能运行

对于`identify`属性，普通应用程序建议留空即可，也可以将其删除。

#### 元数据信息

```
  "RT_VERSION": {
    "#1": {
      "0000": {
        "fixed": {
          "file_version": "0.0.0.0",
          "product_version": "0.0.0.0"
        },
        "info": {
          "0409": {
            "Comments": "",
            "CompanyName": "",
            "FileDescription": "",
            "FileVersion": "",
            "InternalName": "",
            "LegalCopyright": "",
            "LegalTrademarks": "",
            "OriginalFilename": "",
            "PrivateBuild": "",
            "ProductName": "",
            "ProductVersion": "",
            "SpecialBuild": ""
          }
        }
      }
    }
  }
```

这里也并非所有属性都需要填写和修改

- `fixed` ：属性中的两个属性，主要是声明文件版本和程序版本，按照`x.y.z.w`的格式自己填写即可
- `info`：主要是属性信息，其中`0409`是英语的语言代码，表示在英文环境下显示其中的属性，在`info`中可以定义多个语言环境下的属性信息，具体大家可以查看官方项目示例，在其中：
  - `CompanyName` 公司名称
  - `FileDescription` 文件描述
  - `FileVersion` 文件版本
  - `ProductVersion` 程序版本
  - `LegalCopyright` 版权信息
  - `OriginalFilename` 原始文件名
  - `ProductName` 程序名称

## 0x02 编译

### 编译后无图标

不能指定文件编译

```powershell
go build -ldflags="-s -w" -trimpath
```

### Patch方法添加图标

```powershell
go-winres patch --in .\winres\winres.json target.exe
```

