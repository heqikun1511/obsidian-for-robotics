---
tags:
  - 网络
  - 计算机基础
---


## 一、先理解 SPA 是怎么工作的

### 传统多页应用（MPA）

传统网站里，每个 URL 对应服务器上一个真实文件：

text

/                → index.html
/about           → about.html
/contact         → contact.html
/oauth/callback  → oauth/callback.html

浏览器每点一个链接，就向服务器请求一个新 HTML 文件，服务器返回对应文件，页面整体刷新。

**关键点：URL 和服务器上的文件是一一对应的。**

### 单页应用（SPA）

SPA（用 Vue、React、Angular 等构建）只有**一个** HTML 文件，通常是 `index.html`。整个应用的所有页面、路由、逻辑，全部打包进 JS 文件里。

text

/                → index.html（唯一入口）
/about           → 还是 index.html
/contact         → 还是 index.html
/oauth/callback  → 还是 index.html

用户第一次访问时，浏览器下载 `index.html` 和一堆 JS/CSS。之后**所有页面切换都在浏览器内部完成，不再向服务器请求新的 HTML**。

这就是「单页」的含义：**整站只有一个 HTML 页面**。

## 二、问题来了：用户直接访问 /oauth/callback 会怎样

上面说的是「用户在应用内点击导航」。但如果用户**直接从浏览器地址栏输入 `/oauth/callback`**，或者**刷新页面**，或者**从外部链接跳进来**，情况就完全不同了。

这时浏览器会做一件很自然的事：**向服务器发起一个真实的 HTTP 请求，请求 `/oauth/callback` 这个路径。**

而静态服务器（Nginx、Apache、对象存储、GitHub Pages 等）的逻辑是：

> 你请求什么路径，我就去磁盘上找对应的文件，找到就返回，找不到就返回 404。

服务器上一看：

text

/var/www/
├── index.html
├── assets/
│   ├── app.js
│   └── style.css

**根本没有 `/oauth/callback` 这个文件，也没有这个目录。** 于是服务器返回 **404 Not Found**。

用户看到的不是 OAuth 回调页面，而是一个错误页。**SPA 挂掉了。**

这就是 SPA 部署到静态服务器的**根本矛盾**：

**SPA 的前端路由是「虚拟」的，URL 并不对应真实文件；但静态服务器默认按「真实文件」来找，找不到就 404。**


## 三、try_files 是怎么解决的

Nginx 的 `try_files` 指令，作用就是**按顺序尝试多个候选路径，返回第一个存在的；如果全都不存在，就回退到最后一个指定的文件**。

配置：

nginx

location / {
    try_files $uri $uri/ /index.html;
}

逐个解释这三个参数：

|参数|含义|举例（请求 `/oauth/callback`）|
|---|---|---|
|`$uri`|尝试请求的路径本身作为文件|找 `/var/www/oauth/callback` → 不存在|
|`$uri/`|尝试作为目录，找目录下的 index|找 `/var/www/oauth/callback/` → 不存在|
|`/index.html`|前面都失败，回退到这个文件|返回 `/var/www/index.html`|

所以请求 `/oauth/callback` 时：

1. 服务器先找有没有 `oauth/callback` 这个文件 → 没有
    
2. 再找有没有 `oauth/callback/` 这个目录 → 没有
    
3. 回退，返回 `index.html`
    

浏览器拿到的是 `index.html`，也就是 SPA 的入口。接着：

4. 浏览器加载 `index.html` 里的 JS
    
5. JS 启动，读取当前 URL：`/oauth/callback`
    
6. 前端路由匹配到 `/oauth/callback` 对应的组件
    
7. 组件从 URL 里读取 OAuth 参数（如 `?code=xxx` 或 hash 里的 token）
    
8. 完成登录逻辑，渲染回调页面
    

**用户看到的是正常的 OAuth 回调页，而不是 404。** 这就是这段话的核心。

---

## 四、为什么 OAuth 回调特别依赖这个机制

OAuth 登录流程大致是：

1. 用户在 SPA 点「用 Google 登录」
    
2. 跳转到 Google 的授权页
    
3. 用户授权后，Google **重定向回你的应用**，地址类似：
    

text

https://yourapp.com/oauth/callback?code=abc123

4. 浏览器访问这个 URL —— 注意，**这是一次真实的、从外部发起的请求**，不是应用内导航
    
5. 如果服务器没配 `try_files`，直接 404，登录流程断裂
    
6. 配了 `try_files`，服务器返回 `index.html`，JS 启动后读取 `?code=abc123`，拿去换 token
    

**OAuth 回调是典型的「外部直接访问 SPA 深层路径」场景**，所以它最容易暴露 SPA 路由回退没配好的问题。

类似的场景还有：

- 用户刷新任意非根页面（如 `/dashboard`、`/user/123`）
    
- 用户收藏了某个深层页面，下次直接打开
    
- 从邮件、微信、其他网站点链接进来
    

---

## 五、更深入：几种回退写法与坑

### 写法 1：最基础

nginx

`try_files $uri $uri/ /index.html;`

适合绝大多数 SPA。但有个小问题：**如果请求的是真实存在的静态资源，但路径写错了**，也会回退到 `index.html`，返回 200 而不是 404。比如请求一个不存在的 `app.js`，会拿到 HTML，浏览器解析 JS 时报错，比较难排查。

### 写法 2：只对「非静态资源」回退

nginx

location / {
    try_files $uri $uri/ /index.html;
}
location /assets/ {
    try_files $uri =404;
}

静态资源目录单独处理，找不到就老老实实 404，避免误回退。

### 写法 3：用命名 location 更清晰

nginx

location / {
    try_files $uri $uri/ @spa;
}
location @spa {
    rewrite ^ /index.html break;
}

### 常见坑

**1. 放在错误的位置**

`try_files` 必须放在 `location /` 里，如果放在 `location /api/` 之类的规则后面，可能被更精确的 location 匹配抢先，导致不生效。

**2. 和反向代理混用**

如果 SPA 同时有后端 API：

nginx

location /api/ {
    proxy_pass http://backend;
}
location / {
    try_files $uri $uri/ /index.html;
}

注意 `/api/` 的 location 要写在前面且更精确，否则 API 请求可能被 `try_files` 回退成 HTML。

**3. Hash 路由不需要 try_files**

如果 SPA 用的是 **hash 路由**（URL 形如 `/#/about`），路径部分永远是 `/`，服务器只需返回 `index.html` 即可，不需要回退。因为 `#` 后面的内容浏览器不会发给服务器。

只有 **history 路由**（URL 形如 `/about`）才需要 `try_files`。

**4. 其他静态服务器的等价配置**

`try_files` 是 Nginx 的写法，其他平台各有对应：

|平台|等价做法|
|---|---|
|Apache|`.htaccess` 里 `RewriteRule . /index.html [L]`|
|Netlify|`_redirects` 文件：`/* /index.html 200`|
|Vercel|`vercel.json` 里配置 rewrites|
|GitHub Pages|用 `404.html` 技巧，或 Jekyll 配置|
|对象存储 + CDN|配置「404 回源到 index.html」或「默认首页」|

---

## 六、一句话总结

> **SPA 只有一个 `index.html`，但它的前端路由会产生很多「服务器上并不存在」的 URL。当用户直接访问或刷新这些 URL 时，静态服务器默认会 404。`try_files $uri $uri/ /index.html` 的作用就是：找不到真实文件时，统一回退到 `index.html`，把路由控制权交还给前端 JS。OAuth 回调 `/oauth/callback` 正是这种「外部直接访问深层路径」的典型场景，所以必须配好这条规则，否则登录流程会断在 404 上。**

本质上，这是**把「服务器端路由」的职责，移交给了「客户端路由」**——服务器只负责把入口文件发出去，具体渲染哪个页面，由浏览器里的 JS 根据 URL 决定。