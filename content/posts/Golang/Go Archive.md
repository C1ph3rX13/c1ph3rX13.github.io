---
title: "Go Archive"
date: 2023-10-25T09:36:19+08:00
draft: false
url: /posts/2023-09-21/Go-Archive
tags: ["Golang","Archive","解压缩"]
---

## 0x00 前言

Golang  `tar, gzip, zip `

## 0x01 tar

### 创建tar文件

使用 `os.Create()` 函数创建一个新的tar文件

```go
tarFile, err := os.Create("archive.tar")
if err != nil {
    log.Fatal(err)
}
defer tarFile.Close()
```

### 创建tar.Writer

使用 `tar.NewWriter()` 函数创建一个新的 `tar.Writer`，将tar文件和相应的写入流关联起来

```go
tarWriter := tar.NewWriter(tarFile)
defer tarWriter.Close()
```

### 获取被归档的文件信息

使用 `os.Open()` 和 `Stat()` 来获取，之后使用 `tarWriter.WriteHeader()` 写入文件信息

```go
	file, err := os.Open("target.json") // 打开被归档的文件
	if err != nil {
		log.Fatal(err)
	}
	defer file.Close()

	stat, err := file.Stat() // 读取被归档的文件的信息
	if err != nil {
		log.Fatal(err)
	}

	header := &tar.Header{  // 设置tar文件信息
		Name:    filepath.Base(filePath),
		Mode:    int64(stat.Mode()),
		Size:    stat.Size(),
		ModTime: stat.ModTime(),
	}

	err = tarWriter.WriteHeader(header)  // 写入文件信息
	if err != nil {
		log.Fatal(err)
	}
```

`tar.Header`结构体中的常用参数如下：

- `Name`：文件或目录的名称（包括路径）。在打包文件时，需要设置为要打包的文件的完整路径，以便在解包时正确还原文件的路径和名称；
- `Mode`：文件的权限和模式。使用`int64`表示，可以通过`os.FileMode`类型进行转换。它定义了文件的读、写和执行权限，以及文件类型（例如普通文件、目录、符号链接等）；
- `Size`：文件的大小（以字节为单位）。用于指定打包文件时的文件大小信息，以便在解包时正确还原文件的大小；
- `ModTime`：文件的修改时间。用于指定打包文件时的文件修改时间信息，以便在解包时正确还原文件的修改时间。

```go
type Header struct {
    Typeflag   byte          // 文件类型标志，指示文件的类型（普通文件、目录、符号链接等）
    Name       string        // 文件或目录的名称（包括路径）
    Linkname   string        // 如果是符号链接文件，则指向链接目标的路径
    Size       int64         // 文件的大小（以字节为单位）
    Mode       int64         // 文件的权限和模式
    Uid        int           // 文件所有者的用户ID
    Gid        int           // 文件所有者的组ID
    Uname      string        // 文件所有者的用户名
    Gname      string        // 文件所有者的组名
    ModTime    time.Time     // 文件的修改时间
    AccessTime time.Time     // 文件的访问时间
    ChangeTime time.Time     // 文件的变更时间
    Devmajor   int64         // 设备文件的主要设备号
    Devminor   int64         // 设备文件的次要设备号
    Xattrs     map[string]string  // 扩展属性（例如文件系统相关的属性）
    PAXRecords map[string]string  // PAX格式的记录（用于存储更多的元数据信息）
    Format     Format        // 存档格式（默认为FormatUSTAR）
}
```

这些字段用于描述和标识打包文件的各种属性，以便在解包时正确地还原文件的属性信息。以下是对每个字段的详细说明：

- `Typeflag`：文件类型标志，指示文件的类型（普通文件、目录、符号链接等）。常用的类型标志包括：
  - `TypeReg`：普通文件
  - `TypeDir`：目录
  - `TypeSymlink`：符号链接
  - `TypeChar`：字符设备文件
  - `TypeBlock`：块设备文件
  - `TypeFifo`：命名管道（FIFO）
  - `TypeCont`：连续文件
  - `TypeXHeader`：扩展头部
  - `TypeXGlobalHeader`：全局扩展头部
  - `TypeGNULongName`：GNU长文件名
  - `TypeGNULongLink`：GNU长链接名
  - `TypeGNUSparse`：GNU稀疏文件
- `Name`：文件或目录的名称（包括路径）。在打包文件时，需要设置为要打包的文件的完整路径，以便在解包时正确还原文件的路径和名称。
- `Linkname`：如果是符号链接文件，则指向链接目标的路径。
- `Size`：文件的大小（以字节为单位）。用于指定打包文件时的文件大小信息，以便在解包时正确还原文件的大小。
- `Mode`：文件的权限和模式。使用`int64`表示，可以通过`os.FileMode`类型进行转换。它定义了文件的读、写和执行权限，以及文件类型（例如普通文件、目录、符号链接等）。
- `Uid`和`Gid`：文件所有者的用户ID和组ID。用于指定打包文件时的文件所有者信息，以便在解包时正确还原文件的所有者。
- `Uname`和`Gname`：文件所有者的用户名和组名。用于指定打包文件时的文件所有者信息，以便在解包时正确还原文件的所有者。
- `ModTime`：文件的修改时间。用于指定打包文件时的文件修改时间信息，以便在解包时正确还原文件的修改时间。
- `AccessTime`：文件的访问时间。用于指定打包文件时的文件访问时间信息，以便在解包时正确还原文件的访问时间。
- `ChangeTime`：文件的变更时间。用于指定打包文件时的文件变更时间信息，以便在解包时正确还原文件的变更时间。
- `Devmajor`和`Devminor`：设备文件的主要和次要设备号。用于指定打包设备文件时的设备号信息，以便在解包时正确还原设备文件的设备号。
- `Xattrs`：扩展属性（例如文件系统相关的属性）。用于存储额外的文件属性信息。
- `PAXRecords`：PAX格式的记录，用于存储更多的元数据信息。
- `Format`：存档格式。默认为`FormatUSTAR`，表示使用USTAR格式进行打包。其他可选的格式有`FormatPAX`和`FormatGNU`。

这些参数用于描述和标识打包文件的属性，以便在解包时正确地还原文件的属性信息。根据需要，可以设置这些字段来自定义打包文件的属性。

### 完成tar打包

使用`tarWriter.Close()`方法完成tar打包过程，并关闭tar文件。

```go
	_, err = io.Copy(tarWriter, file) // 打开被归档的文件：file
	if err != nil {
		log.Fatal(err)
	}
```

## 0x02 gz

### 创建输出文件

```go
	// 创建输出文件
    gzFile, err := os.Create("output.gz")
    if err != nil {
        log.Panic(err)
    }
    defer gzFile.Close()
```

### 创建gzip.Writer

```go
	gzipWriter := gzip.NewWriter(gzFile)
    defer gzipWriter.Close()
```

### 设置元数据

```go
	// 设置元数据
	gzipWriter.Header = gzip.Header{
		Comment: "This is a compressed file.",
		Extra:   []byte("Some extra data."),
		ModTime: time.Now(),
		Name:    "data.txt",
		OS:      1,
	}
```

`gzip.Header`结构体定义了用于设置gzip文件元数据的字段

```go
type Header struct {
	Comment string    // comment
	Extra   []byte    // "extra data"
	ModTime time.Time // modification time
	Name    string    // file name
	OS      byte      // operating system type
}
```

- `Name string`：要打包的文件的名称。

- `Comment string`：注释信息。

- `Extra []byte`：额外的数据，可以用于存储自定义的元数据。

- `ModTime time.Time`：文件的修改时间。

- `OS byte`：操作系统类型，该字段是一个无符号8位整数，表示文件所在的操作系统。

  以下是一些常见的操作系统类型及其对应的数字值：

  - 0：未知操作系统
  - 1：MS-DOS和Windows
  - 2：Amiga
  - 3：OpenVMS
  - 4：UNIX
  - 5：VM/CMS
  - 6：Atari TOS
  - 7：HPFS文件系统（OS/2、NT）
  - 8：Macintosh
  - 9：Z-System
  - 10：CP/M
  - 11：TOPS-20
  - 12：NTFS文件系统（NT）
  - 13：QDOS
  - 14：Acorn RISCOS
  - 15：Unknown

### 设置压缩级别

**设置压缩级别要在设置元数据之前，否则会造成元数据缺失**

默认为`gzip.DefaultCompression`

```go
	// 创建gzip.Writer并设置压缩级别
    gzipWriter, err = gzip.NewWriterLevel(gzFile, gzip.BestCompression)
    if err != nil {
       log.Panic(err)
    }
    defer gzipWriter.Close()
```

在Go语言的`compress/gzip`包中，可以设置不同的压缩级别来控制gzip压缩的速度和压缩比。`gzip.Writer`结构体提供了一个`CompressionLevel`字段，用于设置压缩级别。

```go
const (
	NoCompression      = flate.NoCompression
	BestSpeed          = flate.BestSpeed
	BestCompression    = flate.BestCompression
	DefaultCompression = flate.DefaultCompression
	HuffmanOnly        = flate.HuffmanOnly
)

const (
	NoCompression      = 0
	BestSpeed          = 1
	BestCompression    = 9
	DefaultCompression = -1

	// HuffmanOnly disables Lempel-Ziv match searching and only performs Huffman
	// entropy encoding. This mode is useful in compressing data that has
	// already been compressed with an LZ style algorithm (e.g. Snappy or LZ4)
	// that lacks an entropy encoder. Compression gains are achieved when
	// certain bytes in the input stream occur more frequently than others.
	//
	// Note that HuffmanOnly produces a compressed output that is
	// RFC 1951 compliant. That is, any valid DEFLATE decompressor will
	// continue to be able to decompress this output.
	HuffmanOnly = -2
)
```

下面是压缩级别的选项及其区别：

- `gzip.DefaultCompression`：默认压缩级别，值为`-1`。此级别会根据输入数据的特性自动选择一个合适的压缩级别。通常情况下，它会提供一个平衡的压缩速度和压缩比；
- `gzip.NoCompression`：无压缩，值为`0`。使用该级别时，数据将以未经压缩的形式存储在gzip文件中，不会进行任何压缩操作。这样可以快速地将数据存储到gzip文件中，但文件大小不会减小；
- `gzip.BestSpeed`：最快速的压缩级别，值为`1`。该级别会以更快的速度进行压缩，但压缩比会相对较低。适用于对压缩速度要求较高的场景；
- `gzip.BestCompression`：最高压缩级别，值为`9`。该级别会以更高的压缩比进行压缩，但压缩速度会相对较慢。适用于对压缩比要求较高的场景。

### 写入数据

```go
    data := []byte("This is the content to be compressed.")
    _, err = gzipWriter.Write(data)
    if err != nil {
        log.Panic(err)
    }
	
	// 使用 io.Copy() 方法
	data := []byte("This is the content to be compressed.")
	dataReader := bytes.NewReader(data)

	_, err = io.Copy(gzipWriter, dataReader)
	if err != nil {
		log.Panic(err)
	}
```

### 刷新缓冲区

使用`gzipWriter.Flush()`方法刷新`gzip.Writer`对象的缓冲区，确保所有数据都被写入输出文件

```go
    err = gzipWriter.Flush()
    if err != nil {
        log.Panic(err)
    }
```

### 结束打包

使用`gzipWriter.Close()`方法结束打包过程，确保所有数据都被写入输出文件并关闭相关资源

```go
    err = gzipWriter.Close()
    if err != nil {
        log.Panic(err)
    }
```

### 0x03 zip

### 创建ZIP文件

使用`os.Create`函数创建一个新的ZIP文件，并获得一个`io.Writer`接口用于写入ZIP文件的内容

```go
    zipFile, err := os.Create("example.zip")
    if err != nil {
        // 处理错误
    }
    defer zipFile.Close()
```

### 创建ZIP写入器

使用`zip.NewWriter`函数创建一个新的ZIP写入器，将ZIP文件的内容写入到之前创建的ZIP文件中

```go
    zipWriter := zip.NewWriter(zipFile)
    defer zipWriter.Close()
```

### 添加文件到ZIP文件中

通过调用`zipWriter.Create`方法创建一个新文件的写入器，并将文件的内容写入到ZIP文件中。可以使用循环来添加多个文件

```go
    // 压缩文件
	// 创建一个文件写入器
	fileToZip, err := os.Open("file.txt")
    if err != nil {
        // 处理错误
    }
    defer fileToZip.Close()

    fileWriter, err := zipWriter.Create("file.txt")
    if err != nil {
        // 处理错误
    }

    _, err = io.Copy(fileWriter, fileToZip)
    if err != nil {
        // 处理错误
    }

	// 压缩字节切片
	// 创建一个文件写入器
	fileWriter, err := zipWriter.Create("data.txt")
	if err != nil {
		// 处理错误
	}

	// 将数据写入文件
	data := []byte("This is the content to be compressed.")
	_, err = fileWriter.Write(data)
	if err != nil {
		// 处理错误
	}
```

### 设置文件的元数据

可以使用`fileWriter`的`FileInfo`方法来获取文件的元数据，并进一步设置ZIP文件中的文件属性，如权限、修改时间等

```go
    fileInfo, err := fileToZip.Stat()
    if err != nil {
        // 处理错误
    }

    fileHeader, err := zip.FileInfoHeader(fileInfo)
    if err != nil {
        // 处理错误
    }

    fileHeader.Name = "file.txt" // 设置文件名

    // 设置其他属性，如权限和修改时间
    fileHeader.SetMode(os.ModePerm)
    fileHeader.SetModTime(time.Now())

    fileWriter, err := zipWriter.CreateHeader(fileHeader)
    if err != nil {
        // 处理错误
    }

    _, err = io.Copy(fileWriter, fileToZip)
    if err != nil {
        // 处理错误
    }

	// 第二种方法：元数据结构体
	fileWriter, err := zipWriter.CreateHeader(&zip.FileHeader{
		Name:     "data.txt",
		Modified: time.Now(),
	})
	if err != nil {
		// 处理错误
	}

	// 将数据写入文件
	data := []byte("This is the content to be compressed.")
	_, err = fileWriter.Write(data)
	if err != nil {
		// 处理错误
	}
```

### 完成ZIP打包

在完成所有文件的添加后，需要调用`zipWriter.Close`方法来完成ZIP文件的打包，将ZIP文件的目录信息写入到ZIP文件中，并关闭ZIP写入器

```go
    err = zipWriter.Close()
    if err != nil {
        // 处理错误
    }
```
