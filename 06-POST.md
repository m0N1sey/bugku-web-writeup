# BugKu Web6 - POST

## 题目信息
- **平台**: [BugKu CTF](https://ctf.bugku.com/)
- **类型**: Web
- **难度**: ⭐
- **金币**: 1
- **分数**: 10

## 题目描述
页面提示需要用 POST 方式提交 `what` 参数，值为 `flag`。

## 解题过程

### 1. 打开题目
页面提示需要用 POST 方式提交参数。

### 2. Burp Suite 抓包
1. 配置浏览器代理到 Burp Suite
2. 开启 Burp 的 Intercept（拦截）
3. 刷新题目页面，Burp 拦截到 GET 请求

### 3. 改为 POST 请求
1. 在 Burp 的 Proxy → Intercept 标签中，右键点击请求
2. 选择 **Change request method**
3. 请求自动从 GET 变为 POST

### 4. 添加 POST 参数
在请求体（body）中添加：what=flag

### 5. 发送请求
点击 **Forward** 发送修改后的请求，页面返回 flag。

