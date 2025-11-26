---
title: "Homebrew"
date: 2025-01-27
lastmod: 2025-01-27
tags:
  - 开发工具/mac开发工具
  - 
publish: true  
---

mac里的一款软件、包管理工具，提供快捷的依赖关系整理
1. 安装命令「使用官网脚本安装」：`/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"`
2. 检查：`brew --version`
3. 使用：`brew install mysql`

首次配置参考教程：https://mirrors.tuna.tsinghua.edu.cn/help/homebrew/
注意
1. 配置brew本身的镜像源「否则其自身的update将会很慢」
2. 配置仓库的镜像源「否则其下载会很慢」