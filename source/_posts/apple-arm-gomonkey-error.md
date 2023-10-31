---
title: Apple arm架构下gomonkey报错问题解决
date: 2023-10-11 17:11:01
tags:
  - arm
  - gomonkey
categories:
  - 技术
  - 编程语言
  - Go
---
# 问题一： undefined: buildJmpDirective
需要将gomonkey升级到github.com/agiledragon/gomonkey/v2

# 问题二：permission denied
修改modify_binary_darwin.go文件，去掉syscall.PROT_EXEC
![](/article/apple-arm-gomonkey-error/1280X1280.png)
