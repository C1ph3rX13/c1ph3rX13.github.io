---
title: "Selenium Config Python"
date: 2023-10-12T17:49:41+08:00
draft: false
url: /posts/2023-10-12/Selenium-Config-Python
tags: ["Python","Selenium"]
---

## 0x00 前言

Selenium Python Usage Note - Base Config

## 0x01 Base Config

### Webdriver Path

#### error output

```python
WebDriver.__init__() got an unexpected keyword argument 'executable_path'
```

version < `selenium: 4.10.0`

```python
from selenium import webdriver
from selenium.webdriver.chrome.options import Options

	option = webdriver.ChromeOptions()
	driver = webdriver.Chrome(executable_path='./chromedriver.exe', options=option)

	driver.get('https://www.google.com/')
```

version >= `selenium: 4.10.0`

```python
from selenium import webdriver
from selenium.webdriver.chrome.options import Options

	# 设置 chromedriver 的路径，使用新版的 Service
	# selenium version > 4.9
	service = Service(executable_path='webdriver/chromedriver.exe')

	# 创建浏览器对象
	driver = webdriver.Chrome(service=service, options=options)
```

#### 参考

Selenium Github：https://github.com/SeleniumHQ/selenium/commit/9f5801c82fb3be3d5850707c46c3f8176e3ccd8e

### Set ChromeOptions

设置 Chrome 的启动参数

```python
	options = webdriver.ChromeOptions()
	options.add_argument('--disable-gpu')
	options.add_argument('--no-sandbox')
	options.add_argument('--window-size=1400,800')
```

#### 参考

Selenium Wiki：https://www.selenium.dev/zh-cn/documentation/webdriver/browsers/chrome/

Google：https://chromedriver.chromium.org/capabilities

### Wait

#### Explicit waits

显式等待：等待元素加载完成才进行定位

```python
from selenium.webdriver.support.wait import WebDriverWait

    # 定位用户名和密码输入框
    username_input = browser.find_element(by=By.NAME, value='username')
    password_input = browser.find_element(by=By.ID, value='password')
    btn = browser.find_element(by=By.XPATH, value='//*[@id="login-form"]/div[5]/button')

    # 显示等待
    wait = WebDriverWait(browser, timeout=3)
    wait.until(lambda username_wait: username_input.is_displayed())
    wait.until(lambda password_wait: password_input.is_displayed())
    wait.until(lambda btn_wait: btn.is_displayed())
```

## 0x02 Action

### find_element()

**by**：指定按照对应的方式来定位元素，可以使用以下几种方式

- **By.ID**：根据元素的 id 属性来定位元素
- **By.NAME**：根据元素的 name 属性来定位元素
- **By.CLASS_NAME**：根据元素的 class 属性指定的值来查找元素
- **By.CSS_SELECTOR**：根据 CSS 选择器的方式来查找元素
- **By.XPATH**：根据 XPath 表达式来查找元素
- **By.LINK_TEXT**：查找文本精确匹配的 < a > 标签元素
- **By.PARTIAL_LINK_TEXT**：查找文本模糊匹配的 < a > 标签元素
- **By.TAG_NAME**：根据标签名称来查找元素，不太常用

**value**：元素位置，字符串类型。具体取决于定位方式的选择

```python
from selenium.webdriver.common.by import By

	username_input = browser.find_element(by=By.NAME, value='username')
	password_input = browser.find_element(by=By.ID, value='password')
```

### 执行 JavaScript

```python
# 执行 JavaScript 代码，滚动到页面底部
driver.execute_script("window.scrollTo(0, document.body.scrollHeight);")
```

## 0x03 Cookies Config

### 获取 Cookies 信息

```python
# 获取当前页面的所有 cookie 信息 (返回字典)
    cookies = browser.get_cookies()
    # 将字典形式的 cookies 转换成 JSON 格式的字符串
    json_cookies = json.dumps(cookies)

    # 生成 cookie 文件名
    cookie_name = f'./cookie.json'

    # 将 JSON 格式的 cookies 写入到本地文件
    with open(cookie_name, "w") as f:
        f.write(json_cookies)
```

### 加载本地 Cookies

```python
def add_cookies():
    try:
        # 读取 cookie.json
        with open('./cookie.json', "r") as f:
            json_cookies = f.read()
        # 将 JSON 格式的 cookies 转换成字典形式
        cookies = json.loads(json_cookies)
        # 将 Cookie 列表转换为字典
        cookies_dict = {cookie['name']: cookie['value'] for cookie in cookies}
        # 返回 cookies
        return cookies_dict
    except FileNotFoundError as error:
        logger.error('Read Error: %s', error)
```

### 利用 Cookies 进行自动登录

```python
# 加载之前保存的 cookies
for cookie in cookies:
    driver.add_cookie(cookie)

# 刷新页面，已加载的 cookies 将自动登录
driver.refresh()
```

## 0x04 ChromeOptions

```python
    options.add_argument('--no-sandbox') # 解决DevToolsActivePort文件不存在的报错
    options.add_argument('window-size=1400,800') # 指定浏览器分辨率
    options.add_argument('--disable-gpu') # 谷歌文档提到需要加上这个属性来规避bug
    options.add_argument('--incognito') # 隐身模式（无痕模式）
    options.add_argument('--disable-javascript') # 禁用javascript
    options.add_argument('--start-maximized') # 最大化运行（全屏窗口）,不设置，取元素会报错
    options.add_argument('--disable-infobars') # 禁用浏览器正在被自动化程序控制的提示
    options.add_argument('--hide-scrollbars') # 隐藏滚动条, 应对一些特殊页面
    options.add_argument('blink-settings=imagesEnabled=false') # 不加载图片, 提升速度
    options.add_argument('--headless') # 浏览器不提供可视化页面. linux下如果系统不支持可视化不加这条会启动失败
    options.add_argument("--disable-dev-shm-usage")
    options.add_argument("--disable-plugins-discovery")
    options.add_argument('--no-first-run')
    options.add_argument('--no-service-autorun')
    options.add_argument('--no-default-browser-check')
```

## 0x05 Proxy

设置代理

```python
    if PROXIES:
        options.add_argument(f"--proxy-server={PROXIES['all://']}")
```
