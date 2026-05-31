---
title: 使用 Obsidian 和 Quartz 在 GitHub Pages 上发布个人博客
draft: false
tags:
  - Obsidian
  - Quartz
---
本文主要介绍如何结合 Obsidian 的本地笔记管理功能和 Quartz 的静态网站生成能力，将你的个人知识库免费发布并托管在 GitHub Pages 上。

![[Pasted image 20260413220449.png]]
## 为什么选择这种方式？

* **完全免费**：利用 GitHub Pages 进行托管，无需支付服务器或托管平台费用。
* **掌控数据**：所有的 Markdown 文件都在你的本地 Obsidian 库中，数据完全属于你。
* **高度定制**：[Quartz](http://quartz.jzhao.xyz/) 是一个功能强大的静态网站生成器（Static Site Generator），支持深色模式、图谱视图、双向链接等 Obsidian 核心特性。

---

## 准备工作

在开始之前，请确保你已经安装并注册了以下工具：
1. **[Node.js](https://nodejs.org/en/)** 
2. **[Git](https://git-scm.com/)**
3. **[GitHub](https://github.com/)** 账号
4. **[Obsidian](https://obsidian.md/)**

---

## 步骤一：克隆并配置 Quartz

Quartz v4 是目前推荐使用的版本，它是基于 Node.js 重新构建的。

1. 打开终端。
2. 将 Quartz 仓库克隆到本地目录。你可以直接在终端中运行：
   ```bash
   git clone https://github.com/jackyzha0/quartz.git
   cd quartz
   ```
3. 运行以下命令安装所需的依赖包：
   ```bash
   npm i
   ```

## 步骤二：连接到 Obsidian Vault

Quartz 通过读取其目录下的 `content` 文件夹来生成静态网页。

* **推荐做法**：你可以直接将整个 `quartz` 文件夹在 Obsidian 中打开作为一个 Vault；或者在 Obsidian 中仅打开 `quartz/content` 文件夹。
* 这样，你在 Obsidian 中对 `content` 文件夹内笔记的任何创建和修改，都会自动同步到 Quartz 需要构建的内容中。

## 步骤三：本地预览网站

在将网站发布到互联网之前，你可以在本地实时预览效果：

1. 在 `quartz` 目录的终端中运行：
   ```bash
   npx quartz build --serve
   ```
2. 命令执行后，在浏览器中访问 `http://localhost:8080` 即可查看你的个人博客。你在 Obsidian 中所做的修改通常也会在浏览器中实时刷新。
	![[Pasted image 20260411165907.png|620]]

## 步骤四：配置并部署到 GitHub Pages

1. 登录你的 GitHub 账号，创建一个新的空仓库（Repository），例如命名为 `my-personal-blog`。**不要**初始化 README 或 .gitignore。
2. 将本地的 Quartz 项目与你的 GitHub 仓库关联（在 `quartz` 目录下运行，将链接替换为你自己的仓库地址）：
   ```bash
   git remote remove origin
   git remote add origin https://github.com/你的用户名/my-personal-blog.git
   ```
3. Quartz 提供了一个非常方便的命令来同步并发布你的网站。运行：
   ```bash
   npx quartz sync
   ```
   这个命令会自动将你的本地笔记 commit 并 push 到 GitHub。
4. **在 GitHub 上设置 Pages**：
- 打开你的 GitHub 仓库页面。
- 进入 **Settings** -> **Pages**。
- 在 **Build and deployment** 下的 **Source** 下拉菜单中，选择 **GitHub Actions**。
- Quartz 的代码中已经包含了一个自动部署的 Action 工作流配置。只要配置好，以后每次你运行 `npx quartz sync` 提交更改时，GitHub Pages 就会自动构建并更新你的博客。

---

## 总结

通过上述步骤，你就搭建好了一个强大且免费的个人博客！之后你只需要两步：
1. 在 Obsidian 中尽情写作（文件存放在 `content` 目录下）。
2. 在终端运行 `npx quartz sync` 进行发布。

> 参考来源： [How to publish Obsidian notes with Quartz on GitHub Pages - Nicole van der Hoeven](https://notes.nicolevanderhoeven.com/How+to+publish+Obsidian+notes+with+Quartz+on+GitHub+Pages)
