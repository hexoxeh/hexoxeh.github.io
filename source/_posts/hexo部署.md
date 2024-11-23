title: hexo 部署
author: 亦 漩
tags:
  - hexo
categories:
  - hexo
date: 2024-11-23 07:43:00
---
[Hexo](https://hexo.io/zh-cn) 是一个基于 Node.js 的快速、简洁且高效的静态博客框架。它支持使用 Markdown（或其他标记语言）撰写文章，并结合丰富的主题模板，在几秒钟内生成优雅的静态网页。

本文将详细介绍如何通过 Docker 部署 Hexo，配置主题与语言设置，并使用 GitHub Actions 实现自动化部署，帮助您快速搭建高效的博客系统。

### 1. 使用 Docker 安装 Hexo

#### 1.1 [安装 Docker](https://docs.docker.com/engine/install/binaries/#install-daemon-and-client-binaries-on-linux) 环境
请确保主机已安装 Docker 与 Docker Compose。如未安装，可通过以下命令快速安装(二进制安装，仅支持 Systemd 类型的发行版)

``` bash
bash <(curl -sSL https://dwz.cn/XOJj0Njx) -i docker
bash <(curl -sSL https://dwz.cn/XOJj0Njx) -i compose
```

安装完成后，使用以下命令验证 Docker 和 Docker Compose 是否正确安装：

``` bash
docker --version
docker-compose --version
```

#### 1.2 构建 Hexo 镜像

> 1. 创建 Hexo 工作目录并编写 Dockerfile 文件：

``` bash
mkdir -p /home/hexo/app
cat <<'EOF'  > /home/hexo/app/Dockerfile
FROM node:iron-alpine3.20

ENV LANG en_US.UTF-8

RUN sed -i 's#https://dl-cdn.alpinelinux.org#https://mirror.nju.edu.cn#g' /etc/apk/repositories \
 && apk --no-cache add bash git openssh vim expect

RUN npm config set registry https://repo.nju.edu.cn/repository/npm/ \
 && npm install -g hexo-cli hexo-server

COPY docker-entrypoint.sh /

WORKDIR /home/hexo/.hexo

RUN addgroup -S hexo -g 2000 \
 && adduser -S hexo -u 2000 -G hexo \
 && chmod +x /docker-entrypoint.sh \
 && chown hexo:hexo /docker-entrypoint.sh \
 && chown -Rf hexo:hexo /home/hexo/.hexo

USER hexo

ENTRYPOINT ["/docker-entrypoint.sh"]
EOF
```

> 2. 创建 docker-entrypoint.sh

该脚本用于初始化 Hexo 环境并启动服务：

``` bash
cat <<'EOF'  > /home/hexo/app/docker-entrypoint.sh
#!/bin/sh

if [ "$#" -gt 0 ]; then
    exec "$@"
else
    if [ ! -f _config.yml ]; then
        hexo init .
    fi

    if [ ! -d node_modules ]; then
        npm install --save
    fi

    hexo clean
    exec hexo server
fi
EOF
```

> 3. 创建 docker-compose.yml

此文件定义了 Hexo 容器的服务配置：

``` bash
cat <<'EOF'  > /home/hexo/docker-compose.yml
name: hexo

services:
  hexo:
    image: alpine/hexo:latest
    build:
      context: ./app
      dockerfile: Dockerfile
    restart: always
    hostname: hexo
    container_name: hexo
    ports:
      - "4000:4000"
    stdin_open: true
    tty: true
    user: 2000:2000
    stop_grace_period: 1s
    volumes:
      - /etc/localtime:/etc/localtime
      - ./data/hexo:/home/hexo/.hexo
EOF
```

> 4. 构建镜像并启动服务

```bash
mkdir -p /home/hexo/data/hexo
chown -Rf 2000:2000 /home/hexo/data
docker-compose up -d --build
```

- 启动成功后，打开浏览器访问 http://127.0.0.1:4000 即可查看 Hexo 首页。

### 2. Hexo 配置

Hexo 提供了多种主题选择，这里以 [Fluid 主题](https://github.com/fluid-dev/hexo-theme-fluid)为例

#### 2.1 配置[主题](https://hexo.io/themes)

> 下载并应用主题

```bash
ghp=https://ghp.ci/
git clone ${ghp}https://github.com/fluid-dev/hexo-theme-fluid /home/hexo/data/hexo/themes/fluid
chown -Rf 2000:2000 /home/hexo/data/hexo/themes
sed -i "s/theme:.*/theme: fluid/" /home/hexo/data/hexo/_config.yml
```

#### 2.2  配置语言和文章链接格式

> [设置语言](https://hexo.fluid-dev.com/docs/guide/#语言配置)为中文

```bash
sed -i "s/language:.*/language: zh-CN" /home/hexo/data/hexo/_config.yml
```

> [配置文章 URL 格式](https://hexo.io/zh-cn/docs/configuration#网址)

```bash
sed -i "s@permalink:.*@permalink: :title/@" /home/hexo/data/hexo/_config.yml
```

### 3. 使用 GitHub Actions 实现自动化部署

#### 3.1 创建仓库

在 GitHub 上新建名为`hexoxeh.github.io`的仓库, 前缀`hexoxeh`为 github 用户名

![](/images/hexo-1.png)![](/images/hexo-2.png)

#### 3.2 配置 Actions 工作流

创建工作流文件，`master`分支提交时触发，执行构建将`public`目录提交到`gh-pages`分支

```basn
mkdir -p /home/hexo/data/hexo/.github/workflows
cat <<'EOF'  > /home/hexo/data/hexo/.github/workflows/deploy.yml
name: Build and Deploy

on:
  push:
    branches:
      - master

permissions:
  contents: write

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout 🛎️
        uses: actions/checkout@v4

      - name: Use Node.js 20 📦
        uses: actions/setup-node@v4
        with:
          node-version: "20"

      - name: Install and Build 🔧 
        run: |
          npm install
          npm run build

      - name: Deploy 🚀
        uses: JamesIves/github-pages-deploy-action@v4.6.9
        with:
          BRANCH: gh-pages
          FOLDER: public
EOF
```

#### 3.3 提交代码到 GitHub

> 1. 创建Git忽略规则文件`.gitignore`

```bash
cat <<'EOF'  > /home/hexo/data/hexo/.gitignore
node_modules/
.deploy_git/
public/
script/
*.log
db.json
yarn.lock
package-lock.json
EOF
```

> 2. 推送代码到 Github

```bash
cd /home/hexo/data/hexo
git init
git add --all
git commit -m "first commit"
git branch -M master
git remote add origin https://github.com/hexoxeh/hexoxeh.github.io.git
git push -u origin master
```

> 3. 验证

![](/images/hexo-3.png)![](/images/hexo-4.png)![](/images/hexo-5.png)

### 4. 配置 Github Page

![](/images/hexo-6.png)

- https://hexoxeh.github.io

### 5. 配置在线编辑文章

修改主题模板为文章末尾添加`a`标签，点击后直接跳转到对应文章编辑页面直接修改提交 Markdown 文件，等待几分钟即可实时更新博客内容

> 1. 获取文章编辑页面URL地址

![](/images/hexo-7.png)

> 2. 修改主题源码

编辑`fluid`主题模板 `/home/hexo/data/hexo/themes/fluid/layout/post.ejs` 第51行添加`<div style="text-align: center"><a href="https://github.com/hexoxeh/hexoxeh.github.io/edit/master/source/<%- page.source %>" target="_blank">编辑文章✏</a></div>`

![](/images/hexo-8.png)

更改`https://github.com/hexoxeh/hexoxeh.github.io`为实际仓库地址及`master` 分支名，再次提交后效果如下

![](/images/hexo-9.png)

![](/images/hexo-10.png)

---

教程完成后，您将拥有一个高度自动化、可在线编辑的 Hexo 博客。访问 https://hexoxeh.github.io 查看最终效果！