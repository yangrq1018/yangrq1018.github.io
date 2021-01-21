---
title: pip的配置
toc:
  number: false
  enable: true
tags:
- Python
- pip
categories: 技术
description: Python的package manager - pip
---

我们在{% post_link gitlab-ci %}中提到Gitlab可以在任何一个repo设置一个repo私有的PyPI。
那么，我们把使用git tag比如`release/1.0.0`这样的标签，推送到服务器触发CI，自动发布到PyPI上之后，
如何在任何一台有访问这个PyPI权限的电脑上安装package呢。

首先，PyPI是包含在Package Registry里面的，而Package Registry是Gitlab API的一部分，想要访问
PyPI，就必须先通过API的验证。Gitlab API的验证方式主要有三种
- Private token，即你可以在Settings>Token里手动申请一个token，然后记住，来证明你的身份
- CI job token，这个token在CI的容器环境内自动定义为一个环境变量，`$CI_JOB_TOKEN`
- Deploy token，暂时不展开

一般创建一个单独的repo，用这个repo的Package Registry来存储全部项目的安装包

假设我们成功编译、测试、上传、发布了一个Python包到这个公共库的PyPI里面，打开PyPI的页面后，后看到

Installation
Pip Command
```
pip install hello-world --extra-index-url https://__token__:<your_personal_token>@gitlab.example.com/api/v4/projects/:id/packages/pypi/simple
```
这个`your_personal_token`意味你必须要有权限才能从PyPI `pip install hello-world`。

固然我们可以每次`pip install`都提供这串URL，但还是比较不方便，如何让`pip`在每次安装的时候不仅检索官方的index，也检索我们的私有PyPI呢？

`pip`提供了一[个配置文件](https://pip.pypa.io/en/stable/user_guide/#config-file)，这个文件在Windows里是用户目录文件下的`%APPDATA%\pip\pip.ini`，
如果没有这个文件就手动创建；在Unix是`~/.config/pip/pip.conf`，内容和格式是一样的。

```
[global]
proxy = http://127.0.0.1:8080
trusted-host =  pypi.python.org
                pypi.org
                gitlab.example.com

extra_index_url = https://__token__:<your_personal_token>@gitlab.example.com/api/v4/projects/:id/packages/pypi/simple
```
- `pip install`命令的很多参数，都可以在这个文件中配置，比如`extra_index_url`对应的就是`pip install --extra-index-url`。
- `proxy`让`pip`走代理
- 如果你的PyPI server用的是私有证书，而且客户机没有信任该证书，`trusted-host`可以让`pip`不验证服务器的身份
- 设置`extra_index_url`后所有的`pip install`都会额外检索这个PyPI

> 私有PyPI里的包建议和官方PyPI的包分开命名，比如package叫做`hello-world`，发布在PyPI的名称可以叫`my-site-hello-world`，
> 否则如果你的package和官方PyPI的package名字重合的话，你的`pip install`会优先安装官方的package