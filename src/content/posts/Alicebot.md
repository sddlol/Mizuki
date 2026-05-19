---
title: 手把手教你部署 Telegram UserBot 项目 —— Alice
published: 2026-05-19
description: "Alice tg userbot项目搭建指南"
draft: false
author: sddlol
sourceLink: "https://blog.cnmsb.cfd/posts/alicebot/"
---
# 避坑指南：手把手教你部署 Telegram UserBot 项目 —— Alice
## 0. 为什么选择 Alice？
大多数 Telegram 机器人使用的是官方的 Bot API，功能受限。**Alice** 的独特之处在于它使用 **Telegram User 账号**作为 Bot。这种方案在目前市面上非常罕见，且 Alice 已经有了较为成熟的架构。
虽然官方提供的“一键脚本”目前存在一些 Bug（容易卡死），但通过手动安装可以完美解决。本文将教你如何在境外 Linux 服务器上从零开始手动部署 Alice。
## 1. 环境准备
在开始之前，本教程只适用于Ubuntu/Debian。
### 1.1 安装 Docker
```bash
curl -fsSL get.docker.io | bash
# 安装完成后建议启动并设置开机自启
systemctl enable --now docker

```
### 1.2 安装 Node.js (v22)
Alice 的 Runtime 需要较新版本的 Node 环境。
```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt-get install -y nodejs

```
### 1.3 安装 pnpm
```bash
npm install -g pnpm

```
### 1.4 安装 Go (v1.22+)
根据你的服务器架构选择对应的安装命令。
**x86_64 (普通 Intel/AMD 服务器):**
```bash
curl -fsSL https://go.dev/dl/go1.22.linux-amd64.tar.gz | sudo tar -C /usr/local -xzf -

```
**ARM64 (如 Oracle ARM, 树莓派):**
```bash
curl -fsSL https://go.dev/dl/go1.22.linux-arm64.tar.gz | sudo tar -C /usr/local -xzf -

```
**配置 Go 环境变量:**
```bash
echo 'export PATH=$PATH:/local/go/bin' >> ~/.bashrc
source ~/.bashrc

```
## 2. 下载并初始化项目
### 2.1 克隆仓库
```bash
git clone https://github.com/LlmKira/Alice.git
cd Alice/runtime

```
### 2.2 依赖安装与编译
这里是关键步骤，请务必按顺序执行：
 1. **安装依赖：** pnpm install
 2. **批准构建：** pnpm approve-builds（执行后会出现选择界面，建议**全选**）
 3. **重新编译：** pnpm rebuild
## 3. 配置文件修改
### 3.1 环境变量配置
```bash
cp .env.example .env
nano .env

```
在 .env 中填入你的核心信息：
 * **API ID & API Hash：** 去 my.telegram.com/apps 创建应用获取。
 * **Phone：** 填入你要挂载为 Bot 的 Telegram 账号手机号（带国家码，如 86...）。
 * **LLM API KEY：** 填入你的中转或官方 API Key。
### 3.2 核心业务配置
```bash
nano config.toml

```
这个文件的注释非常详细。重点修改 [[llm.endpoints]] 部分。
**示例配置（适配 DeepSeek 和 豆包/Gemini）：**
你可以配置多个模型，一个用于日常聊天，一个用于图像识别（Vision）。
```toml
# 聊天模型配置
[[llm.endpoints]]
name = "deepseek-chat"
base_url = "https://api.ohmygpt.com/v1"
api_key_env = "LLM_API_KEY"
model = "deepseek-v4-flash"

# 图像识别模型配置 (Vision)
[vision]
# 启用图片感知，建议使用多模态模型
model = "gemini-2.0-flash-exp" # 或者换成豆包对应的多模态模型名
base_url = "https://api.ohmygpt.com/v1"
max_per_tick = 5

# 网易云音乐 API (可选)
music_api_base_url = "你的NeteaseCloudMusicApi地址"

```
## 4. 构建 Skill Runner
Alice 的插件/技能运行在 Docker 容器中以保证安全，我们需要手动构建这个镜像：
```bash
# 在 Alice 目录下执行
docker build -t alice-skill-runner:bookworm -f Dockerfile.skill-runner .

```
## 5. 启动 Alice
一切就绪后，启动项目：
```bash
pnpm start

```
**注意：** 第一次启动时，终端会提示你输入 Telegram 的验证码（如果是 User 账号）。输入验证码并确认后，Alice 就会正式上线！
### 💡 小贴士
 * **持久化运行：** 建议使用 screen 或 pm2 来守护进程，防止 SSH 断开后程序停止。
 * **省钱攻略：** 像作者说的那样，日常聊天用 DeepSeek V4 Flash，看图用 Gemini Flash 或豆包，性价比极高。
 * **Bug 反馈：** 虽然 Alice 目前还有些小 Bug，但作者更新很勤快，建议关注 GitHub 仓库动态。
祝大家玩得开心！😎
