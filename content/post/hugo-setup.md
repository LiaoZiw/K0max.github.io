+++
title = 'Hugo GitHub主页搭建与踩坑记'
date = 2024-03-20T15:58:10+08:00
draft = false
toc = true
+++
# 前言

阅读本文前应该已经了解git的使用,基本命令行操作, 且已经安装了Hugo

由于网络上众多文章发布时间较早, Hugo 和GitHub 产生了部分更新导致有一定出入,遂为此文

本文精简了官方文档的可选操作,故每一步基本都是必要操作

# GitHub 相关操作

首先新建名为 `username.github.io` 的repo, `username` **必须**为你的 GitHub 用户名, 不得有误

到`GitHub->Settings->Developer settings->Personal access token->Tokens(classic)`生成access token

**注意:** access token仅会展示一次,一定要保存到本地

2023年某月开始GitHub不再允许通过username和password进行git push, 转为使用username和access token.

# Hugo 本地仓库

新建文件夹 `reponame` 并 `cd` 进入(后续将该目录简称为`repo`), 然后 `hugo new site sitename`, `sitename` 随意. 进行`git init`初始化

## 安装主题(theme)

```sh
git submodule add https://github.com/theNewDynamic/gohugo-theme-ananke.git themes/ananke
```
此时在`repo/themes`下会出现一个名为`ananke`的文件夹

## 修改配置(hugo.toml)

打开`repo/hugo.toml`(若无则新建), **注意:** 在2023.2 Hugo官方将`config.toml`迁移到了`hugo.toml`, 故网络上的文章提到的`config.toml` 现在即为`hugo.toml`.

```toml
baseURL = 'https://username.github.io'
languageCode = 'zh-CN'
title = "Choose a name by yourself"
theme = 'ananke'
```

## 新建一条post

```sh
hugo new content post/newpost.md
```
**注意:** 虽然Hugo官方doc给的是 `posts/newpost.md` 但是(部分/大部分?)主题并不识别 `posts` 而是 `'post`

此时会在`repo/content/post`下创建文件,修改该.md文件. 文件开头被加号包起来的东西叫作 `frontmatter`

**注意:** 记得修改`draft` 或者 `date`, 否则等下部署的时候也不会出现任何post

## 本地部署测试

```sh
hugo server
```
然后打开浏览器访问`http://localhost:1313`.

`hugo`或`hugo server`都会查看`repo/content/post`下的文件,然后在`repo/public`下生成相应的.html

本地进行`git commit`

# 部署到GitHub

## GitHub操作

到你`username.github.io`下, `Settings->Pages`, 找到`Build and Deployment`, 将`Source`从`Deploy from a branch`改为`GitHub Actions`

## 本地操作

新建`repo/.github/workflows/hugo.yaml`,填入下面的内容
```yaml
# Sample workflow for building and deploying a Hugo site to GitHub Pages
name: Deploy Hugo site to Pages

on:
  # Runs on pushes targeting the default branch
  push:
    branches:
      - main

  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:

# Sets permissions of the GITHUB_TOKEN to allow deployment to GitHub Pages
permissions:
  contents: read
  pages: write
  id-token: write

# Allow only one concurrent deployment, skipping runs queued between the run in-progress and latest queued.
# However, do NOT cancel in-progress runs as we want to allow these production deployments to complete.
concurrency:
  group: "pages"
  cancel-in-progress: false

# Default to bash
defaults:
  run:
    shell: bash

jobs:
  # Build job
  build:
    runs-on: ubuntu-latest
    env:
      HUGO_VERSION: 0.124.0
    steps:
      - name: Install Hugo CLI
        run: |
          wget -O ${{ runner.temp }}/hugo.deb https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.deb \
          && sudo dpkg -i ${{ runner.temp }}/hugo.deb          
      - name: Install Dart Sass
        run: sudo snap install dart-sass
      - name: Checkout
        uses: actions/checkout@v4
        with:
          submodules: recursive
          fetch-depth: 0
      - name: Setup Pages
        id: pages
        uses: actions/configure-pages@v4
      - name: Install Node.js dependencies
        run: "[[ -f package-lock.json || -f npm-shrinkwrap.json ]] && npm ci || true"
      - name: Build with Hugo
        env:
          # For maximum backward compatibility with Hugo modules
          HUGO_ENVIRONMENT: production
          HUGO_ENV: production
        run: |
          hugo \
            --gc \
            --minify \
            --baseURL "${{ steps.pages.outputs.base_url }}/"          
      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public

  # Deployment job
  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```
记得修改上面的hugo version(好像不改也没事)

添加远程仓库, origin作为别名(当然也可以用别的奇怪的别名,origin只是一种范例而非关键字)
```sh
git remote add origin https://github.io/username.github.io
```sh
推送到远程仓库
```sh
git push origin main
```
会弹出来验证信息,要求输入用户名和密码(名义上是密码,但是应该输入access token, 如果输入密码会提示使用密码验证已经不再支持)

# 为什么本地测试没有post显示?

可能有以下几种情况: (post 指代`repo/content/post/youpost.md`)
1. post的frontmatter中的date是明天
2. post中`draft=true`, 应改为 `draft=false`. 或者采用`hugo server -D` 也可将草稿内容部署到网站
3. 你使用了`hugo new content posts/post.md` 而非 `hugo new content post/post.md`, 主题可能是寻找`post`文件夹而非`posts`文件夹生成.html文件的(Hugo官方doc和theme总有一种脱节的美). 或者到theme文件夹里修改`config.yaml` 将 `mainSections` 改成 `posts`

