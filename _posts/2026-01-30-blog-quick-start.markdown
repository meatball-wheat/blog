---
layout: post
# remote_theme: mistakes/minimal-mistakes
title:  "用github pages和jekyll搭建一个建议博客"
date:   2026-01-30 17:00:00 +0800
categories: memo
---

## 目的
快速搭建一个建议的blog系统，没有太多美化的诉求，仅为记录各种随笔

## 系统
macOs，本机已有git环境，而且安装了homebrew

## 步骤和踩坑
- step1 创建git仓库，
  - 如果不是主站，可以自定义名称
- step2 安装ruby
  - 操作过程：https://jekyllrb.com/docs/installation/macos/
- step3 搭建脚手架
  - 拉取刚刚创建的git库
  - 创建一个全新自定义分支（看起来官方教程是这么建议的）
  - 删除所有文件（不用担心，脚手架会自动创建新的）
  - 初始化jekyll项目：`jekyll new --skip-bundle .`
  - 修改GemFile，将安装jekyll那行注释掉，同时修改`gem "github-pages", "~> GITHUB-PAGES-VERSION", group: :jekyll_plugins
`，其中通过[这里][getVersion]获取`GITHUB-PAGES-VERSION`
  - 将`Gemfile.lock
`加入到.gitignore
- step4 本地调试
  ``` shell
  # 先安装依赖
  bundle install
  # 开启本地服务
  bundle exec jekyll serve
  ```
- step5 本地调试通过后推送远端
- step6 在远程仓库的settings页面，选择page选择部署方式
  - Build and deployment选择GitHub Actions，然后在GitHub Pages Jekyll的workflows中将分支替换为step3创建的分支
  - 在Actions里面可以触发编译部署
- step7 创建成功！通过settings里面的visit site按钮可以直接跳转到地址

至此，大功告成！

<!-- ![Alt text]({{ "/assets/images/oh-yeah.png" | relative_url }}) -->

<img src="{{ '/assets/images/oh-yeah.png' | relative_url }}" 
     alt="Oh Yeah" 
     style="max-width: 50%; height: auto; display: block; margin: 0 auto;" />


[getVersion]: https://pages.github.com/versions.json