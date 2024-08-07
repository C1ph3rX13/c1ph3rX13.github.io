---
title: "Path Traversal Vulnerabilities With Python"
date: 2023-10-20T10:59:14+08:00
draft: false
url: /posts/2023-10-20/Path-Traversal-Vulnerabilities-With-Python
tags: ["Python","httpx","requests"]
---

## 前言

使用 httpx 发现包含路径穿越的 `/../` 的路径在请求的时候会被自动去除

```python
url = target + f'/media/xpack/../replay/'
    # 第一种请求方式
    resp = client.request(url=eurl, timeout=5, method='Get')
    # 第二种请求方式
    resp = client.get(url=url, timeout=5)
        
# Url Output
'/media/xpack/replay/'
```

## 原因

参考文章：https://mazinahmed.net/blog/testing-for-path-traversal-with-python/

我在 httpx 社区的提问：https://github.com/encode/httpx/discussions/2897

## 解决

### urllib.request

```python
url = "https://example.com/../../../etc/passwd"
urllib.request.urlopen(url)
```

### requests.Request

```python
url = "https://example.com/../../../etc/passwd"
with requests.Session() as s:
    r = requests.Request(method='GET', url=url, headers=headers)
    prep = r.prepare()
    prep.url = url
    req = s.send(prep, verify=False)
```
AND

```python
url = "https://example.com/../../../etc/passwd"
s = requests.Session()
r = requests.Request(method='GET', url=url)
prep = r.prepare()
prep.url = url
```

### Using urllib3.HTTPConnectionPool

```python
import urllib3

pool = urllib3.HTTPConnectionPool("localhost", 8000)
r = pool.urlopen("GET", "/../../../../doing/certain/check")
print(r.status)
```

### Downgrading urllib3

+ 降级 requests 或 urllib3

+ pip install urllib3==1.24.3

