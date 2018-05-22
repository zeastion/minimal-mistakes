---
  title: "Minimal Mistakes 本地环境搭建及功能配置"
  categories:
    - 超能力
  date: 2018-05-22
  toc: true
  toc_label: "Minimal Mistakes"
  toc_icon: "people-carry"
  header:
    teaser: /assets/images/githubpages.jpg
---

## Jekyll 测试环境

本地搭建 Jekyll 服务，用于博客新增功能测试

### rvm

```bash
# yum update -y

# gpg --keyserver hkp://keys.gnupg.net --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3 7D2BAF1CF37B13E2069D6956105BD0E739499BDB

# curl -sSL https://get.rvm.io | bash -s stable

# source /etc/profile.d/rvm.sh
```

### ruby

```bash
# rvm install ruby

# ruby --version
ruby 2.4.1p111 (2017-03-22 revision 58053) [x86_64-linux]

# gem --version
2.6.14
```

### 更换 Gem 源

```bash
# gem source -l
*** CURRENT SOURCES ***
https://rubygems.org/

# gem sources --add https://gems.ruby-china.org/ --remove https://rubygems.org/
https://gems.ruby-china.org/ added to sources
https://rubygems.org/ removed from sources

# gem source -l
*** CURRENT SOURCES ***
https://gems.ruby-china.org/
```

### jekyll

```bash
# gem install jekyll bundler

# jekyll -v
jekyll 3.8.1

# useradd jekyll

# chown -R jekyll:jekyll /home/jekyll
```

### 常规创建项目

```bash
# cd /home/jekyll/

# jekyll build

# jekyll new .

# tree /home/jekyll/
/home/jekyll/
├── 404.html
├── about.md
├── _config.yml
├── Gemfile
├── Gemfile.lock
├── index.md
└── _posts
    └── 2018-05-16-welcome-to-jekyll.markdown

# bundle exec jekyll serve --host 0.0.0.0 --port 80 --detach
Configuration file: /home/jekyll/_config.yml
            Source: /home/jekyll
        Destination: /home/jekyll/_site
  Incremental build: disabled. Enable with --incremental
      Generating...
                    done in 0.395 seconds.
  Auto-regeneration: disabled when running server detached.
    Server address: http://0.0.0.0:80/
Server detached with pid '15313'. Run 'pkill -f jekyll' or 'kill -9 15313' to stop the server.
```
预览

![jekyll-1](http://ov30w4cpi.bkt.clouddn.com/jekyll-1.png)

### 使用 minimal mistake

博客使用 GitPage + minimal mistake 框架，测试环境保持版本一致

下载 [minimal mistakes 最新代码](https://github.com/mmistakes/minimal-mistakes)，解压到 /home/jekyll/ 目录

```bash
# cd /home/jekyll/

# vim Gemfile
source "https://gems.ruby-china.org/"
gem "minimal-mistakes-jekyll"

# bundle install
```

修改配置文件

```bash
# vim _config.yml
略
```

## 功能设置

### 1- 博文头图

设置文章页顶部图片

a. 本地图片

```yaml
---
header:
  image: /assets/images/head.jpg
---
正文内容
```

b. 网络图片

```yaml
---
header:
  image: http://some-site.com/assets/images/image.jpg
---
正文内容
```

### 2- 推荐栏预览图

博文结尾会推荐几篇文章，给其设置题图，建议尺寸为 500x300

![jekyll-2](http://ov30w4cpi.bkt.clouddn.com/jekyll-2.png)

a. 默认配图

在 _config.yml 里设置一张默认图

```yaml
teaser: "/assets/images/default.jpg"
```

b. 自定义配图

```yaml
# vim _posts/test.md
---
layout: single
title: "测试预览图"
date: 2018-05-16
header:
  teaser: /assets/images/test.jpg
---
正文内容
```

### 2- 一些小功能

修改 _config.yml

搜索栏

```yaml
search: true
```

每页标题数

```yaml
paginate: 5
```

时区

```yaml
timezone: Asia/Shanghai
```

### 3- 导航

网站横栏导航分类，目前就 “类目”（按分类展示所有博文）、“旧博客”（之前的新浪微博链接）两个功能块

```yaml
# vim _data/navigation.yml
main:
  - title: "类目"
    url: /categories/
  - title: "旧博客"
    url: http://blog.sina.com.cn/zeastion

# mkdir categories

# vim categories/scale_models.md
---
title: "类目汇总"
layout: categories
permalink: /categories/
author_profile: true
---
```

每篇博文头部添加自定义的分类信息

```yaml
---
title: "AE86 实做记录"
categories:
  - 民用模型
---
正文
```

[效果展示](https://zeastion.github.io/categories/)

### 4- 当前博文目录

在每篇博文右侧显示本文目录

```yaml
# vim _posts/test.md
---
toc: true
toc_label: "set my content name"
toc_icon: "align-left"
---
```

效果如下

![jekyll-3](http://ov30w4cpi.bkt.clouddn.com/jekyll-3.png)

小图标可以自定义，素材见 [图标网站](https://fontawesome.com/icons?d=gallery&s=solid&m=free)

## Github Pages 代码升级

升级本地代码到当前版本

```
# git remote add upstream https://github.com/mmistakes/minimal-mistakes.git

# git pull upstream master
```

再删除以下文件及目录


    .editorconfig
    .gitattributes
    .github
    /docs
    /test
    CHANGELOG.md
    minimal-mistakes-jekyll.gemspec
    README.md
    screenshot-layouts.png
    screenshot.png
