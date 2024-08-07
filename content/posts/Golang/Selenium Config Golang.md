---
title: "Selenium Config Golang"
date: 2023-10-18T17:47:26+08:00
draft: false
url: /posts/2023-10-12/Selenium-Config-Golang
tags: ["Golang","Selenium"]
---

## 0x00 前言

Selenium Golang Usage Note - Base Config

## 0x01 Base Config

### Webdriver Path

```go
import (
	"github.com/tebeka/selenium"
	"github.com/tebeka/selenium/chrome"
)

// 设置 chromeDriverPath & port
const (
	chromeDriverPath = "./chromedriver.exe"
	port             = 4443
)
```

### Server & Client Config

#### Server Config

Selenium Golang 不能像 Python 一样自启动 Driver，需要先设置

```go
// 需要先启动 ChromeDriverService
	_, err := selenium.NewChromeDriverService(chromeDriverPath, port)
	if err != nil {
		log.Panicf("Error starting the ChromeDriver Server: %v", err)
	}
```

#### Client Config

1. 设置 Google Chrome 的启动参数
2. 将启动参数添加到配置
3. 设置 Debug 模式
4. 使用配置后的 driver 打开页面

```go
	// 设置 Chrome 的启动参数
	caps := selenium.Capabilities{
		"browserName": "chrome",
	}

	// 使用字符串插值来替换 --proxy-server 参数中的 PROXIES，变量的值将被动态地插入到启动参数
	opts := chrome.Capabilities{
		Args: []string{
			"--headless",
			"--no-sandbox",
			"--disable-gpu",
			"--user-agent=Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/118.0.0.0 Safari/537.36 Edg/118.0.2088.46",
		}}

	// 动态设置代理
	if PROXIES != "" {
		opts.Args = append(opts.Args, fmt.Sprintf("--proxy-server=%s", PROXIES))
	}
	caps.AddChrome(opts)

	// 设置 Debug 模式为 false
	selenium.SetDebug(false)

	// 启动 Chrome 访问网页
	driver, err := selenium.NewRemote(caps, fmt.Sprintf("http://localhost:%d/wd/hub", port))
	if err != nil {
		log.Panicf("Error starting the ChromeDriver Remote: %v", err)
	}
	
	// 打开登录页面
	if err := driver.Get(Url); err != nil {
		log.Panicf("Request Error: %v", err)
	} else {
		log.Info("Request Successfully")
	}
	time.Sleep(time.Second * 3)
```

## 0x02 Element

Selenium Golang 提供的元素定位方法与 Python 类似

```go
	// 定位
	usernameInput, _ := browser.FindElement(selenium.ByName, "username")
	passwordInput, _ := browser.FindElement(selenium.ByID, "password")
	btn, _ := browser.FindElement(selenium.ByXPATH, "//*[@id=\"login-form\"]/div[5]/button")

	// 清空输入框
	_ = usernameInput.Clear()
	_ = passwordInput.Clear()

	// 输入用户名和密码
	_ = usernameInput.SendKeys(username)
	_ = passwordInput.SendKeys(password)

	// 点击提交
	_ = btn.Click()
```

## 0x03 Cookies

### 获取 Cookies 

```go
	// 获取 cookies
	cookies, _ := driver.GetCookies()
```

### 转换 Cookies

`go-resty` 使用，转换的类型为：`[]*http.Cookie`

```go
	// 将 []Cookie 转换为 JSON 格式字符串
	data, _ := json.MarshalIndent(cookies, "", "  ")

	// 将 JSON 字符串写入文件
	err := os.WriteFile("cookies.json", data, 0777)
	if err != nil {
		log.Panicf("Write Failed : %v", cookies)
	} else {
		log.Info("Cookies Successful Write")
	}
```

### Cookies 编码

`go-resty` 接收的 Cookies 类型为：`[]*http.Cookie`，当包含特殊字符时报错：

```cmd
# 在设置 Cookie 时存在无效的字节，这可能是因为 Cookie 的值包含特殊字符或非法字符导致的
# headers 也可能存在此问题
net/http: invalid byte '"' in Cookie.Value; dropping invalid bytes
```

解决方法：使用 `url.QueryEscape()` 函数对值进行编码

```go
    for _, cookie := range cookies {
        cookie.Value = url.QueryEscape(cookie.Value)
    }
```

在编码 Cookies 值和 headers 值之后，需要更新请求中的 `SetCookies()` 和 `SetHeaders()` 方法

```go
	// 读取 cookie.json
	file, err := os.ReadFile("cookies.json")
	if err != nil {
		log.Warnf("Read Error: %v", err)
	}

	// 解析 JSON 格式的 cookies
	var cookies []*http.Cookie
	err = json.Unmarshal(file, &cookies)
	if err != nil {
		log.Fatal("JSON Error: ", err)
	}

	// 编码 Cookie 值
	for _, cookie := range cookies {
		cookie.Value = url.QueryEscape(cookie.Value)
	}
	
	// 设置 headers
    headers := map[string]string{
            "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/117.0.0.0 Safari/537.36",
        }

	// 遍历 cookies 切片，将每个 cookie 添加到 headers 中
	for _, cookie := range cookies {
		if cookie.Name == "csrftoken" {
			headers["X-CSRFToken"] = cookie.Value
			log.Warnf("X-CSRFToken Found: %v", cookie.Value)
			break
		}
	}

	// 编码 headers 值
	for key, value := range headers {
		headers[key] = url.QueryEscape(value)
	}

	// 配置 client
	client := resty.New().
		SetCookies(cookies).
		SetHeaders(headers)
	// 配置代理
	if PROXIES != "" {
		client.SetProxy(PROXIES)
	}
```

