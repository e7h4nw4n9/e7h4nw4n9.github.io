---
URL: https://tryhackme.com/room/foolsm8v2
Difficulty: Medium
tags:
  - web
---

> [!abstract] 简述
> 该靶机属于中等难度的 web 挑战。任务是在网页棋盘上完成国际象棋的“一步将杀”，从而获得 Flag。该靶机的难点在于识别原型污染(Prototype pollution)漏洞位置，以及如何实现漏洞利用。

## 分析

### 端口信息收集

> [!warning] nmap 说明
> 扫描靶机过程中为了加快扫描进度，扫描速度使用的都是 `T5`。实际渗透、红队行动中需要根据实际情况来设置扫描速度，避免使用 `T5` 。

使用命令 `sudo nmap -p- -T5 -oN port.txt {HOST}` 扫描开放的端口，扫描结果如下：

![[IMG-20260714144841105.jpg]]

可以看到服务器开放了 22、3000 端口，接下来进行端口服务信息扫描：

![[IMG-20260714145613472.jpg]]

**从扫描结果中，可以看到 3000 端口上运行的是 Node.js 服务。**

### 网页操作分析

接下来访问网页，查看网页操作、API 等信息：

![[IMG-20260714145948924.jpg]]

网页打开后，将 车 走到截图中的位置。从控制台可以看到请求了 `/api/move` 接口，接口返回成功，但是网页出现了如图所示的提示信息。 `/api/move` 接口的请求、返回信息如下（使用 curl 重新请求，并用 jq 命令格式化了输出）：

![[IMG-20260714152635447.jpg]]

从返回信息中，可以看到关键提示信息 `"reason": "reward gate closed: session.config.unlocked is not set"`。**因此可知，该靶机的关键是要将 `session.config.unlocked` 设置为 true**。

网页中还有 `Save preferences` 和 `reset` 这两个操作，这两个操作对应的 API 请求分别如下：

![[IMG-20260714153127830.jpg]]

![[IMG-20260714153201686.jpg]]

> [!info] /api/move 接口操作说明
> 在使用 curl 多次请求 `/api/move` 接口时，会发现第一次请求成功后接着在请求该接口时会返回返回，错误提示信息为 `illegal move`。因此每次成功请求 `/api/move` 接口后，都需要请求一次 `/api/reset` 接口。

接下来分析网页源码。在网页 HTML、app.js 源码中搜索 config、unlocked，均未获取到搜索结果。

![[IMG-20260714153707370.jpg]]

![[IMG-20260714153844656.jpg]]

**综上，可以猜测需要通过 API 请求来通过 Prototype pollution 漏洞修改 `session.config.unlocked`的值。**

## 漏洞利用

首先通过 `/api/settings` 接口来修改 `session.config.unlocked` 的值，使用如下的恶意参数：

```json
{"theme":"forest","pieceSet":"outline","animationMs":180, "constructor": {"prototype": {"unlocked": true}}}
```

使用 curl 命令来请求接口：

![[IMG-20260714154902696.jpg]]

接下来在请求 `/api/move` 接口：

![[IMG-20260714155033222.jpg]]

可以看到接口返回了 flag。

## 总结

在完成该靶机的练习时，我首先考虑的是从 client 端完成该任务。通过浏览器 debugger 修改执行过程中的必要参数等操作，均未获取到 flag。最终意识到是需要在 server 端完成修改 `config`的配置，接着开始尝试直接在 `/api/move` 接口中添加 `prototype pollution` 相关的恶意参数（ `__proto__`, `constructor` 这两种参数）来获取 flag，同样也是失败的。最后尝试通过 `/api/settings` 接口来实现 `config` 的修改，才终于完成了该靶机。

复盘整个过程，可以得到如下经验：

1. 需要先定位漏洞是在 client 端还是 server 端；
2. 梳理攻击路径，不要一开始就从目标接口开始测试。

Prototype pollution 的相关说明，可以参考[portswigger 的 prototype pollution 课程](https://portswigger.net/web-security/prototype-pollution)。
