---
title: The Low Level Operating System Primitives
date: 2024-01-30T14:55:04+08:00
draft: false
url: /posts/2024-01-30/The-low-level-Operating-System-Primitives
tags:
  - Golang
  - syscall
  - windows
slug: English Preview
---
> The Low Level Operating System Primitives
> <!--more-->

# [syscall package](https://pkg.go.dev/syscall?GOOS=windows)

## 加载动态链接库（DLL）

### syscall.LoadDLL

- LoadDLL函数用于显式地加载一个DLL，并返回一个`syscall.DLL`对象和一个`error`。`syscall.DLL`对象包含了DLL的句柄以及导出函数的地址。
- 当使用LoadDLL函数加载DLL时，如果加载失败，会返回一个非`nil`的`error`，并且需要检查`error`来处理加载失败的情况。

```go
// LoadDLL loads the named DLL file into memory.
//
// If name is not an absolute path and is not a known system DLL used by
// Go, Windows will search for the named DLL in many locations, causing
// potential DLL preloading attacks.
//
// Use LazyDLL in golang.org/x/sys/windows for a secure way to
// load system DLLs.
func LoadDLL(name string) (*DLL, error) {
	namep, err := UTF16PtrFromString(name)
	if err != nil {
		return nil, err
	}
	var h uintptr
	var e Errno
	if sysdll.IsSystemDLL[name] {
		h, e = loadsystemlibrary(namep)
	} else {
		h, e = loadlibrary(namep)
	}
	if e != 0 {
		return nil, &DLLError{
			Err:     e,
			ObjName: name,
			Msg:     "Failed to load " + name + ": " + e.Error(),
		}
	}
	d := &DLL{
		Name:   name,
		Handle: Handle(h),
	}
	return d, nil
}
```

  #### syscall.MustLoadDLL

- MustLoadDLL函数也用于加载一个DLL，但是它会在加载失败时直接`panic`。因此，MustLoadDLL适用于对加载DLL失败情况没有特别处理逻辑的场景。
- MustLoadDLL函数的使用方式与LoadDLL类似，但不需要检查error，因为它会在加载失败时直接panic。

```go
// MustLoadDLL is like LoadDLL but panics if load operation fails.
func MustLoadDLL(name string) *DLL {
	d, e := LoadDLL(name)
	if e != nil {
		panic(e)
	}
	return d
}
```

### syscall.NewLazyDLL

- NewLazyDLL函数用于创建一个`*syscall.LazyDLL`对象，该对象包含了DLL的名称和导出函数的延迟加载信息。
- LazyDLL对象并不立即加载DLL，而是在调用其Load方法时才会去加载DLL，并返回一个`*syscall.LazyDLL`对象和一个`error`。这样做的好处是可以将DLL的加载延迟到需要使用其中的函数时再进行。

```go
// A LazyDLL implements access to a single DLL.
// It will delay the load of the DLL until the first
// call to its Handle method or to one of its
// LazyProc's Addr method.
//
// LazyDLL is subject to the same DLL preloading attacks as documented
// on LoadDLL.
//
// Use LazyDLL in golang.org/x/sys/windows for a secure way to
// load system DLLs.
type LazyDLL struct {
	mu   sync.Mutex
	dll  *DLL // non nil once DLL is loaded
	Name string
}

// Load loads DLL file d.Name into memory. It returns an error if fails.
// Load will not try to load DLL, if it is already loaded into memory.
func (d *LazyDLL) Load() error {
	// Non-racy version of:
	// if d.dll == nil {
	if atomic.LoadPointer((*unsafe.Pointer)(unsafe.Pointer(&d.dll))) == nil {
		d.mu.Lock()
		defer d.mu.Unlock()
		if d.dll == nil {
			dll, e := LoadDLL(d.Name)
			if e != nil {
				return e
			}
			// Non-racy version of:
			// d.dll = dll
			atomic.StorePointer((*unsafe.Pointer)(unsafe.Pointer(&d.dll)), unsafe.Pointer(dll))
		}
	}
	return nil
}

// mustLoad is like Load but panics if search fails.
func (d *LazyDLL) mustLoad() {
	e := d.Load()
	if e != nil {
		panic(e)
	}
}

// Handle returns d's module handle.
func (d *LazyDLL) Handle() uintptr {
	d.mustLoad()
	return uintptr(d.dll.Handle)
}

// NewProc returns a LazyProc for accessing the named procedure in the DLL d.
func (d *LazyDLL) NewProc(name string) *LazyProc {
	return &LazyProc{l: d, Name: name}
}

// NewLazyDLL creates new LazyDLL associated with DLL file.
func NewLazyDLL(name string) *LazyDLL {
	return &LazyDLL{Name: name}
}
```
# [windows package](https://pkg.go.dev/golang.org/x/sys)

## 加载动态链接库（DLL）

### windows.LoadDLL

- `LoadDLL`函数用于显式地加载一个非系统的DLL，并返回一个`*windows.DLL`对象和一个error。`*windows.DLL`对象包含了DLL的句柄以及导出函数的地址。
- 与`syscall.LoadDLL`类似，`windows.LoadDLL`用于显式加载非系统DLL的方法。

```go
// LoadDLL loads DLL file into memory.
//
// Warning: using LoadDLL without an absolute path name is subject to
// DLL preloading attacks. To safely load a system DLL, use LazyDLL
// with System set to true, or use LoadLibraryEx directly.
func LoadDLL(name string) (dll *DLL, err error) {
	namep, err := UTF16PtrFromString(name)
	if err != nil {
		return nil, err
	}
	h, e := syscall_loadlibrary(namep)
	if e != 0 {
		return nil, &DLLError{
			Err:     e,
			ObjName: name,
			Msg:     "Failed to load " + name + ": " + e.Error(),
		}
	}
	d := &DLL{
		Name:   name,
		Handle: h,
	}
	return d, nil
}
```

### windows.MustLoadDLL

- `MustLoadDLL`函数用于加载一个非系统的DLL。它会在加载失败时直接panic，因此适用于对加载DLL失败情况没有特别处理逻辑的场景。
- `MustLoadDLL`函数与`syscall.MustLoadDLL`类似，但是在`windows`包中专门用于加载非系统DLL。

```go
// MustLoadDLL is like LoadDLL but panics if load operation failes.
func MustLoadDLL(name string) *DLL {
	d, e := LoadDLL(name)
	if e != nil {
		panic(e)
	}
	return d
}
```

### windows.NewLazyDLL & windows.NewLazySystemDLL

- `LazySystemDLL`是一个类型，表示系统的动态链接库。它实际上是`*windows.LazyDLL`类型的实例，用于延迟加载系统DLL。

- `NewLazySystemDLL`函数用于创建一个`*windows.LazyDLL`对象，该对象代表系统的动态链接库。这个对象可以延迟加载系统DLL，并提供访问其中导出函数的能力。
- 与`syscall.NewLazyDLL`类似，`windows.NewLazySystemDLL`是用于延迟加载系统DLL的方法。

```go
// A LazyDLL implements access to a single DLL.
// It will delay the load of the DLL until the first
// call to its Handle method or to one of its
// LazyProc's Addr method.
type LazyDLL struct {
	Name string

	// System determines whether the DLL must be loaded from the
	// Windows System directory, bypassing the normal DLL search
	// path.
	System bool

	mu  sync.Mutex
	dll *DLL // non nil once DLL is loaded
}

// Load loads DLL file d.Name into memory. It returns an error if fails.
// Load will not try to load DLL, if it is already loaded into memory.
func (d *LazyDLL) Load() error {
	// Non-racy version of:
	// if d.dll != nil {
	if atomic.LoadPointer((*unsafe.Pointer)(unsafe.Pointer(&d.dll))) != nil {
		return nil
	}
	d.mu.Lock()
	defer d.mu.Unlock()
	if d.dll != nil {
		return nil
	}

	// kernel32.dll is special, since it's where LoadLibraryEx comes from.
	// The kernel already special-cases its name, so it's always
	// loaded from system32.
	var dll *DLL
	var err error
	if d.Name == "kernel32.dll" {
		dll, err = LoadDLL(d.Name)
	} else {
		dll, err = loadLibraryEx(d.Name, d.System)
	}
	if err != nil {
		return err
	}

	// Non-racy version of:
	// d.dll = dll
	atomic.StorePointer((*unsafe.Pointer)(unsafe.Pointer(&d.dll)), unsafe.Pointer(dll))
	return nil
}

// mustLoad is like Load but panics if search fails.
func (d *LazyDLL) mustLoad() {
	e := d.Load()
	if e != nil {
		panic(e)
	}
}

// Handle returns d's module handle.
func (d *LazyDLL) Handle() uintptr {
	d.mustLoad()
	return uintptr(d.dll.Handle)
}

// NewProc returns a LazyProc for accessing the named procedure in the DLL d.
func (d *LazyDLL) NewProc(name string) *LazyProc {
	return &LazyProc{l: d, Name: name}
}

// NewLazyDLL creates new LazyDLL associated with DLL file.
func NewLazyDLL(name string) *LazyDLL {
	return &LazyDLL{Name: name}
}

// NewLazySystemDLL is like NewLazyDLL, but will only
// search Windows System directory for the DLL if name is
// a base name (like "advapi32.dll").
func NewLazySystemDLL(name string) *LazyDLL {
	return &LazyDLL{Name: name, System: true}
}
```

