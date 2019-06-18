---
title: Nginx使用HTTPS反向代理laravel的HTTP应用
date: 2019-06-18 14:38:57
categories:
- study
tags:
- Nginx
---

这几天配置Nginx服务器的反向代理，使用HTTPS代理Laravel的HTTP应用出现了一点小问题，经过一番研究，特地记录下来。

<!--more-->

## 使用场景
反向代理的概念不多做解释，此处应用反向代理主要原因如下：

1. 应用服务器不必暴露在公网，躲在防火墙后，通过内网连接代理服务器（proxy server）；
2. DNS设置简单，因为有两个不同应用服务器，对应不同的子域名（a.example.com, b.example.com），只需设置DNS `*.example.com` 到代理服务器IP即可；
3. SSL证书管理方便，只需申请一个通配符的证书 `*.example.com` 配置在代理服务器。


### 环境
系统环境都是*Ubuntu 18.04.2*，服务器都是使用*Nginx*，代理服务器配置SSL，使用HTTPS对外访问（HTTP强制跳转到HTTPS），应用服务器使用*Laravel*，HTTP对内网访问。


## 配置

### 应用服务器配置

应用服务器的nginx照常配置即可，可以设置`server_name`为对应的域名。

### 代理服务器配置

代理服务器从网上抄来一份，主要是设置`proxy_pass`，下面只是location block：

```b使
location / {
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_pass    http://ip;
}
```

注意要加 `proxy_set_header X-Forwarded-Proto $scheme` 这一条，让后面被代理的应用服务器知道对应的协议是什么，这样应用服务器生成url（如css，js文件）使用相应的协议。如果代理服务器确定使用https，那`$scheme`替换成`https`也可以。
>http://nginx.org/en/docs/http/ngx_http_core_module.html#var_scheme

## 问题

本以为这样就搞定了，但是在用域名测试的时候发现网站不能正常加载，浏览器有如下错误：

```
Mixed Content: The page at '<URL>' was loaded over HTTPS, but requested an insecure stylesheet '<URL>'. This request has been blocked; the content must be served over HTTPS.
```

在页面中混用了HTTPS和HTTP，浏览器会block掉不安全的HTTP链接。而HTTP链接主要是用laravel的`asset`生成的css, js文件，css文件没有加载上，所以就不能正常加载了。
>https://laravel.com/docs/5.5/helpers#method-asset

多方查询，有推荐使用`proxy_redirect`来替换HTTP为HTTPS，但是试用不成功。

也有说这些资源文件用uri路径来表示，`\css\example.css` 这样，不带协议。

或者页面增加一个meta，让HTTP升级为安全的HTTPS：
```
<meta http-equiv="Content-Security-Policy" content="upgrade-insecure-requests" />
```

这样在生产环境没问题，但是需要改动很多代码，而且开发、测试环境也需要ssl证书，很不方便。

在查询到几篇文章中，有提到后端是tomcat，需要修改tomcat的配置来接受`X-Forwarded-Proto` 这个Header来处理。我思考是不是laravel也需要类似设置，google一下发现了这个：
>https://laravel.com/docs/5.5/requests#configuring-trusted-proxies
>
>When running your applications behind a load balancer that terminates TLS / SSL certificates, you may notice your application sometimes does not generate HTTPS links. Typically this is because your application is being forwarded traffic from your load balancer on port 80 and does not know it should generate secure links.

Laravel从5.5开始，内置了一个 `TrustProxies`模块来解决这个问题，专门应对使用了Load Balancer等情况。

所以按照文档，修改 `protected $proxies = '**'`，因为应用服务器在内网，所以改成`**`也无所谓。

Laravel 5.5之前的版本，可以自行用`composer`来安装这个插件。
>https://github.com/fideloper/TrustedProxy

测试之后，问题解决。


## 参考链接
[NGINX下升级HTTPS踩到的坑](https://www.centos.bz/2018/01/nginx%E4%B8%8B%E5%8D%87%E7%BA%A7https%E8%B8%A9%E5%88%B0%E7%9A%84%E5%9D%91/)

[Load Blade assets with https in Laravel](https://stackoverflow.com/a/52049183)