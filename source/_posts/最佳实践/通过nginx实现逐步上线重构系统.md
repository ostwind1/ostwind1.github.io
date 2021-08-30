---
title: 通过nginx实现逐步上线重构系统
date: 2021-03-21 21:18:42
tags: 工程实践
category: 最佳实践
---
<!-- more -->
## 00 背景
公司现有一个历史遗留项目，项目为 `PHP5.6`+`Smart` 构建的单站点应用。现使用 `Laravel` + `Vue` 重构为前后端分离项目。
为减小一次性上线风险，使用nginx反向代理能力，分模块逐步上线重构后的项目。
<!-- more -->

## 01 操作方法
> nginx 规则：https://moonbingbing.gitbooks.io/openresty-best-practices/content/ngx/nginx_local_pcre.html
通过nginx代理规则，将新项目的路由通过老项目域名代理
```
location ~ /(center|vendor|favicon|cost) {
    proxy_pass https://new.domain.com;
    proxy_set_header X-Real-IP $remote_addr;
}
```
https://new.domain.com 是 重构后前端项目的域名。 vendor 和 favicon 是前端项目的资源路径。
此时，当浏览器请求指定路由时，就会将请求代理至前端项目。
## 02 其他方法
如果项目使用了 k8s 并且前后端项目在同一个命名空间下可以通过配置 `ingress` 来代替 配置项目的`nginx`。本次重构因前后端项目不在同一命名空间故未使用此方法。