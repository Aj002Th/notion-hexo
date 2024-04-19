---
categories: 实践记录
tags:
  - blog
  - 工具
description: ''
permalink: build-blog-hugo/
title: 'hugo: 零成本自建技术博客'
cover: /images/98cc79c7e2f1e62617fcb83906e7075c.jpg
date: '2024-04-18 19:51:00'
updated: '2024-04-19 19:00:00'
---

### 方案选择


github pages 提供博客站点 + hugo 提供博客模板 + github actions 自动构建和发布


### part 1 : Github Pages


官网：[GitHub Pages](https://pages.github.com/)


选择 github pages 的优点就是 github 使用频率高，并且作为全球最大的代码托管平台，也很稳定，即使是免费提供使用也不用担心跑路的问题，缺点当然就是对于国内的网络访问不算友好


步骤如下：

1. 新建 Repositories 用于存储博客内容，注意：名称格式为：`<用户名>.github.io`
2. 将博客网页代码上传至仓库中
3. 访问博客，博客站点访问地址为：**`https://<`**`用户名>`**`.github.io/`**

### part 2 : hugo


官网：[The world’s fastest framework for building websites | Hugo (gohugo.io)](https://gohugo.io/)


hugo 是使用 go 语言实现的静态博客网站生成器，可将 md 文章内容转换为美观的博客网站代码


步骤如下：

1. 安装 git 用于将博客网页上传 Github, 安装 hugo 用于生成博客网页（最好安装extended版本）
2. 运行 `hugo new site <文件夹名>`，创建站点，创建结束后也会有后续使用流程的指示

	```javascript
	1. Change the current directory to D:\AAAData\Code\Blog\blog.
	2. Create or install a theme:
	   - Create a new theme with the command "hugo new theme <THEMENAME>"
	   - Install a theme from https://themes.gohugo.io/
	3. Edit hugo.toml, setting the "theme" property to the theme name.
	4. Create new content with the command "hugo new content <SECTIONNAME>\<FILENAME>.<FORMAT>".
	5. Start the embedded web server with the command "hugo server --buildDrafts".
	```

3. 安装主题，我选用的是 LoveIt，运行 `git clone https://github.com/dillonzq/LoveIt.git themes/LoveIt`
4. 依据主题配置说明，例如 LoveIt 主题的 [https://hugoloveit.com/](https://hugoloveit.com/)，配置 hugo.toml
5. 在 content 文件夹下编写 md 文章，直接新建 md 文件当然也是可以的，但是推荐运行 `hugo new content <FILENAME>` 来生成博客文章 md 文件，可以自动添加文章头部的默认元数据配置
6. 运行 hugo 指令生成静态博客网站代码，生成结果位于 public 文件夹中
7. 将 public 文件夹中的内容上传至 Github Pages 所对应的仓库

### part 3 : Github Actions


github actions 是 github 自带的 CICD 系统，可以直接搭配使用


目前我们的博客其实是可以正常使用了，但是每次修改完内容后都需要手动运行 hugo 命令生成静态网页文件，然后再手动将 public 文件夹下的内容上传至 github pages 所对应的仓库，较为繁琐，可以说用 github actions 来自动化这部分流程


我们最终需要两个 github 仓库，一个我们称之为 博客源仓库，这里存储博客的内容和 hugo 的配置等，另一个是 github pages 所对应的的仓库


最终的效果是：我们在本地的站点中写博客，然后推送至 博客源仓库，触发博客源仓库的 github actions，自动生成静态网页文件并推送至 github pages 仓库


步骤如下：

1. 添加 github actions 配置

	配置需要放置在仓库目录 `.github/workflows` 下，以 `.yml` 为后缀


	下面是一个例子，可以按需修改


	```yaml
	name: deploy
	
	on:
	    push:
	    workflow_dispatch:
	    schedule:
	        # Runs everyday at 8:00 AM
	        - cron: "0 0 * * *"
	
	jobs:
	    build:
	        runs-on: ubuntu-latest
	        steps:
	            - name: Checkout
	              uses: actions/checkout@v2
	              with:
	                  submodules: true
	                  fetch-depth: 0
	
	            - name: Setup Hugo
	              uses: peaceiris/actions-hugo@v2
	              with:
	                  hugo-version: "latest"
	
	            - name: Build Web
	              run: hugo
	
	            - name: Deploy Web
	              uses: peaceiris/actions-gh-pages@v3
	              with:
	                  PERSONAL_TOKEN: ${{ secrets.PERSONAL_TOKEN }}
	                  EXTERNAL_REPOSITORY: Aj002Th/Aj002Th.github.io
	                  PUBLISH_BRANCH: master
	                  PUBLISH_DIR: ./public
	                  commit_message: ${{ github.event.head_commit.message }}
	```

2. 创建 Token，需要 workflows 和 repo 权限

	上述配置中使用到了一个变量 `secrets.PERSONAL_TOKEN` ，因为我们需要从博客仓库推送到外部 GitHub Pages 仓库，需要特定权限，要在 GitHub 账户下 `Setting - Developer setting - Personal access tokens` 下创建一个 Token，至少需要 workflows 和 repo 权限

3. 配置 Token 到博客源仓库

	配置后复制生成的 Token（需要注意的是：Token 只会出现一次），然后在我们博客源仓库的 `Settings - Secrets and variables - Actions` 中添加 `PERSONAL_TOKEN` 环境变量为刚才的 Token，这样 GitHub Action 就可以获取到 Token 了

4. 推送代码

### 最终成果

1. 在本地的站点仓库中使用 md 编辑器写文章，并通过 hugo serve 命令预览生成的静态网页效果
2. 完成文章内容后 git push 到 github 上的博客源仓库，同时触发 github actions 构建和生成最新的静态网页代码并发布到 github pages 所在仓库中
3. 可以访问 github pages 查看最新博客内容
