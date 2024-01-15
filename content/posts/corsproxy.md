---
title: node反向代理解决跨域问题
date: 2018-01-10 12:06:12
categories:
- http
tags:
- cors
---

### 创建proxy server

``` bash
$ npm install -g corsproxy
$ corsproxy
## with custom port: CORSPROXY_PORT=1234 corsproxy
## with custom host: CORSPROXY_HOST=localhost corsproxy
## with debug server: DEBUG=1 corsproxy
## with custom payload max bytes set to 10MB (1MB by default): CORSPROXY_MAX_PAYLOAD=10485760 corsproxy
```
使用方法：
- http://localhost:1337/localhost:3000/sign_in
- http://localhost:1337/my.domain.com/path/to/resource

### 创建测试Server
[https://github.com/troygoode/node-cors-server](https://github.com/troygoode/node-cors-server)

``` bash
$ npm install
$ node app.js
```

### 创建测试Client
[https://github.com/troygoode/node-cors-client](https://github.com/troygoode/node-cors-client)

``` bash
$ npm install
$ node app.js
```
修改client项目中index.js的访问url
```
http://localhost:1337/localhost:3000
```
访问http://localhost:3001 查看测试页面