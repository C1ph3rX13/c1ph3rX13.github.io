---
title: "Go Resty Transport"
date: 2023-09-21T14:30:13+08:00
draft: false
url: /posts/2023-09-21/Go-Resty-Transport
tags: ["Golang","Resty","Transport","高并发复用连接优化"]
---

## 0x00 前言

Go Resty 高并发复用连接优化

## 0x01 优化思路

### resty\transport.go

resty.New() 在创建客户端是 http.Transport 的默认配置相关

```go
func createTransport(localAddr net.Addr) *http.Transport {
	dialer := &net.Dialer{
		Timeout:   30 * time.Second,
		KeepAlive: 30 * time.Second,
		DualStack: true,
	}
	if localAddr != nil {
		dialer.LocalAddr = localAddr
	}
    // 返回 &http.Transport 结构体
	return &http.Transport{
		Proxy:                 http.ProxyFromEnvironment,
		DialContext:           dialer.DialContext,
		ForceAttemptHTTP2:     true,
		MaxIdleConns:          100,
		IdleConnTimeout:       90 * time.Second,
		TLSHandshakeTimeout:   10 * time.Second,
		ExpectContinueTimeout: 1 * time.Second,
		MaxIdleConnsPerHost:   runtime.GOMAXPROCS(0) + 1,
	}
}
```

### http\transport.go

1. 创建了一个名为`clientConfig`的HTTP客户端配置

2. `clientConfig := &http.Client{}`：创建一个空的HTTP客户端配置对象，其中`&`表示取该对象的地址

3. `Transport: &http.Transport{}`：在客户端配置中创建一个传输层对象，该对象用于定义HTTP请求的传输选项

4. `MaxConnsPerHost: nil`：设置在传输层对象中每个主机允许的最大连接数

```go
clientConfig := &http.Client{
	Transport: &http.Transport{
		MaxConnsPerHost: 2000,
	},
}
```

## 0x02 优化代码

### NewWithClient()

使用`resty.NewWithClient()`传入自定义的`&http.Client`

```go
restyClient := resty.NewWithClient(clientConfig)
```

### SetTransport()

使用`SetTransport()`设置`MaxConnsPerHost`

```go
client := resty.New().
	SetTransport(&http.Transport{
		MaxIdleConnsPerHost: 10,
		MaxConnsPerHost: 200,
	})
```
