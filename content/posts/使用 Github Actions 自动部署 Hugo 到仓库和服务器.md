---
created: 2024-07-02
updated: 2024-07-03
type: Notes
status: 📥 收集箱
Rating: 
share: true
date: 2024-07-02T22:01:58+08:00
title: 使用 Github Actions 自动部署 Hugo 到仓库和服务器
description: 为了不在本地电脑上折腾 Hugo，而且本地的 git 也总是出故障，能力有限暂时处理不了。所以打算用 Github Actions 自动部署到腾讯云的服务器，同时也可以部署一份到 Github Pages 作为备份。这样就能实现我在本地使用任意的 markdown 编辑器写完笔记，将 md 文件上传到 Github 后，就不用管了~
categories: Hugo
tags:
  - Hugo
  - Github
featured_image: https://images.unsplash.com/photo-1618401479427-c8ef9465fbe1?crop=entropy&cs=tinysrgb&fit=max&fm=jpg&ixid=M3wzNjAwOTd8MHwxfHNlYXJjaHwyfHxnaXRodWJ8ZW58MHx8fHwxNzE5OTUxMzI0fDA&ixlib=rb-4.0.3&q=80&w=1080
---

为了不在本地电脑上折腾 Hugo，而且本地的 git 也总是出故障，能力有限暂时处理不了。

所以打算用 Github Actions 自动部署到腾讯云的服务器，同时也可以部署一份到 Github Pages 作为备份。

这样就能实现我在本地使用任意的 markdown 编辑器写完笔记，将 md 文件上传到 Github 后，就不用管了~

## 准备工作

1. 新建 Github 仓库 `xlglife.github.io` （请自行修改为自己的信息）
2. 创建一个 Github 的 `codespace`（在线的 VScode，本地 git 坏了没法用）
3. 终端运行 `git submodule add https://github.com/AmazingRise/hugo-theme-diary.git themes/diary` 安装主题 （以 `diary` 主题为例）
4. 从 [这里](https://github.com/AmazingRise/hugo-theme-diary/tree/main/exampleSite) 下载 `exampleSite` 文件覆盖根目录

这样 Hugo 程序的源文件就上传好了
![](http://img.xlg.life/images%2F2024%2F07%2F03%2F20240703021259-e802d03d9a08209558e75408ac105182.png)

## 部署到 Github Pages

1. 首先在 [这里](https://github.com/settings/tokens) 生成一个 Github Token（repo 权限）
2. 在仓库的 `Settings` → `Secrets and variables` → `Actions` 中添加一个 `Secret`
	1. Name 填入 `ACTION_ACCESS_TOKEN`
	2. Secret 填入刚才生成的 Token
3. 在仓库的 `Settings` → `Pages` → `Build and deployment` 里选择
	1. Source 选择 `Deploy from a branch`
	2. Branch 选择 `gh-pages`
	3. 记得勾选下面的 `Enforce HTTPS`
4. 最后一步，在仓库根目录下创建文件 `.github/workflows/Deploy.yml`，内容写入：

```yml
name: Build and Deploy

on:
  push:
    branches:
      - main
    # 当推送到 main 分支时触发

jobs:
  build-deploy:
    runs-on: ubuntu-latest
    # 运行在最新版本的 Ubuntu 上

    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          submodules: true # 确保检出子模块
          fetch-depth: 0    # 获取完整历史记录
        # 检出仓库代码并包含子模块

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v3
        with:
          hugo-version: "latest"
          extended: true
        # 设置 Hugo，安装最新版本的 Hugo 扩展版

      - name: Update submodules
        run: git submodule update --init --recursive
        # 初始化并更新子模块

      - name: Build
        run: hugo
        # 运行 Hugo 命令构建静态网站

      - name: Deploy to GitHub Pages
        uses: peaceiris/actions-gh-pages@v4
        with:
          github_token: ${{ secrets.ACTION_ACCESS_TOKEN }}
          publish_branch: gh-pages
          publish_dir: ./public
        # 部署到 gh-pages 分支
        # 使用由 GitHub 自动生成的 ACTION_ACCESS_TOKEN 进行身份验证
        # 将构建后的 ./public 目录中的文件发布到 gh-pages 分支
```

保存提交，然后等待部署完成后，就可以访问 https://xlglife.github.io 看看了

## 部署到服务器

1. ssh 登录到服务器，输入命令 `ssh-keygen -t rsa -b 4096 -C "your_email@example.com"`（改成自己的邮箱）
2. 一路默认回车后，就可以在 `~/.ssh` 目录下看到 `id_rsa`（私钥）和 `id_rsa.pub`（公钥）
3. 首先将公钥里的内容，复制到 `~/.ssh/authorized_keys` 文件中
4. 然后回到 Github 仓库，添加如下 `Secret`
	1. `SERVER_PATH`，服务器上存储静态资源的路径
	2. `SERVER_HOST`，服务器 IP 地址
	3. `SERVER_PORT`，服务器 ssh 端口，一般为 22
	4. `SERVER_USER`，服务器 ssh 用户
	5. `SERVER_KEY`，前面生成的私钥内容填入这里
5. 在 `.github/workflows/Deploy.yml` 后面补充：

```yml
      - name: Deploy to Server
        uses: burnett01/rsync-deployments@7.0.1
        with:
          switches: -avzr --delete
          path: ./public
          remote_path: ${{ secrets.SERVER_PATH }}
          remote_host: ${{ secrets.SERVER_HOST }}
          remote_port: ${{ secrets.SERVER_PORT }}
          remote_user: ${{ secrets.SERVER_USER }}
          remote_key: ${{ secrets.SERVER_KEY }}
        # 通过 ssh 连接服务器，用 rsync 同步文件
        # 服务器信息通过 GitHub Secrets 设置
```

再次保存提交，等待部署完成，就可以访问服务器上的网址了 https://xlg.life

## 补充说明

其实部署到 Github 仓库和服务器，主要是最后一步的 Deploy 不同，我看来很多人写的不同的 Actions 命令，基本上部署到 Github Pages 都是用的 `peaceiris/actions-gh-pages`，而部署到服务器的确是有很多种方法，在这里我也被 ssh key 这个坑阻挠了好久，最终找到了 `burnett01/rsync-deployments` 才搞定。

## 参考资料

- [GitHub - actions/checkout: Action for checking out a repo](https://github.com/actions/checkout)
- [GitHub - peaceiris/actions-hugo: GitHub Actions for Hugo ⚡️ Setup Hugo quickly and build your site fast. Hugo extended, Hugo Modules, Linux (Ubuntu), macOS, and Windows are supported.](https://github.com/peaceiris/actions-hugo)
- [GitHub - peaceiris/actions-gh-pages: GitHub Actions for GitHub Pages 🚀 Deploy static files and publish your site easily. Static-Site-Generators-friendly.](https://github.com/peaceiris/actions-gh-pages)
- [GitHub - Burnett01/rsync-deployments: GitHub Action for deploying code via rsync over ssh](https://github.com/burnett01/rsync-deployments)
- [使用 git action 自动部署 blog](https://yukimio.world/p/hugo-deploy-2307-zh/)