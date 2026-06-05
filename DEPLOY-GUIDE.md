# 部署「历史的回音」技能到 GitHub — 操作指南

## 第一步：在 GitHub 网页上创建仓库

1. 打开 https://github.com/new
2. Repository name 填写：`historical-chronological-dialogue-simulator`
3. Description 填写：`历史的回音 — 编年体跨时代对话模拟器（WorkBuddy Skill）`
4. 选择 **Public**
5. **不要**勾选 "Add a README file"、".gitignore"、"License"（已包含在本地）
6. 点击 **Create repository**

## 第二步：在本地终端执行推送命令

仓库创建完成后，打开 **Git Bash** 或 **PowerShell**，执行以下命令：

```bash
cd D:\workbuddydata\2026-06-05-16-54-06\historical-chronological-dialogue-simulator-repo

# 添加远程仓库
git remote add origin https://github.com/liup03317-netizen/historical-chronological-dialogue-simulator.git

# 推送到 GitHub
git push -u origin main
```

> 如果提示认证，请使用 GitHub Personal Access Token（不是密码）作为密码。
> 生成方法：GitHub → Settings → Developer settings → Personal access tokens → Generate new token

## 第三步：验证仓库

推送完成后，访问以下地址确认：
https://github.com/liup03317-netizen/historical-chronological-dialogue-simulator

## 其他人安装此技能

仓库公开后，其他人可通过以下方式安装：

### 方式一：克隆安装（推荐）

```bash
git clone https://github.com/liup03317-netizen/historical-chronological-dialogue-simulator.git \
  ~/.workbuddy/skills/historical-chronological-dialogue-simulator
```

### 方式二：下载 ZIP 安装

1. 访问 https://github.com/liup03317-netizen/historical-chronological-dialogue-simulator
2. 点击 **Code → Download ZIP**
3. 解压到 `~/.workbuddy/skills/historical-chronological-dialogue-simulator/`

### 方式三：WorkBuddy 技能链接（如支持）

```
/install-skill https://github.com/liup03317-netizen/historical-chronological-dialogue-simulator
```
