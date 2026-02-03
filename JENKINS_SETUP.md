# Jenkins + Docker + GitHub 部署指南

本指南帮助你在 Docker 中运行的 Jenkins 上，从 GitHub 拉取本项目并完成构建与部署。

---

## 一、前置条件确认

- 服务器上已用 Docker 安装并运行 Jenkins
- 项目已推送到 GitHub（建议使用 SSH 或 Personal Access Token）
- 服务器可访问 GitHub（若在国内可考虑代理或镜像）

---

## 二、Jenkins Docker 常用命令（回顾）

```bash
# 查看 Jenkins 容器是否在运行
docker ps | grep jenkins

# 若未运行，典型启动示例（按你实际镜像与端口调整）
docker run -d --name jenkins \
  -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  jenkins/jenkins:lts
```

---

## 三、Jenkins 基础配置

### 3.1 安装必要插件

1. 打开 Jenkins：`http://你的服务器IP:8080`
2. **系统管理** → **插件管理** → **可选插件**
3. 搜索并安装：
   - **Git**
   - **Pipeline**（Pipeline 与 Pipeline: Stage View）
   - **NodeJS Plugin**（用于 Node/npm/pnpm 构建）
   - **Docker Pipeline**（若需要在流水线里用 Docker）

安装后如提示重启，请重启 Jenkins。

### 3.2 配置 Git

- **系统管理** → **全局工具配置** → **Git**
- 若宿主机已装 Git，可填 **Path to Git executable**，例如：`/usr/bin/git`
- 若用 Jenkins 自带 Git，保持默认即可

### 3.3 配置 Node.js（用于 Next.js 构建）

1. **系统管理** → **全局工具配置** → **NodeJS**
2. 点击 **新增 NodeJS**，取名如 `Node 20`
3. 勾选 **自动安装**，选择需要的版本（如 20.x LTS）
4. 保存

### 3.4 添加 GitHub 凭据

**方式 A：使用 GitHub 用户名 + Personal Access Token（推荐）**

1. GitHub → **Settings** → **Developer settings** → **Personal access tokens** 创建 Token
2. 勾选权限：至少 `repo`
3. Jenkins：**系统管理** → **凭据** → **系统** → **全局凭据** → **添加凭据**
   - 类型：**Username with password**
   - 用户名：GitHub 用户名
   - 密码：粘贴 **Token**（不是登录密码）
   - ID：如 `github-token`，流水线里会用到

**方式 B：使用 SSH 密钥**

1. 在服务器或 Jenkins 容器内生成 SSH 密钥，公钥添加到 GitHub **Deploy keys** 或账号 **SSH keys**
2. Jenkins 凭据里添加 **SSH Username with private key**，私钥粘贴进去，ID 如 `github-ssh`

---

## 四、创建流水线任务（从 GitHub 拉代码并构建）

### 4.1 新建任务

1. **新建 Item**
2. 输入任务名称（如 `demo_jenkins`）
3. 选择 **Pipeline**
4. 确定

### 4.2 配置 Pipeline

在任务配置页：

1. **General**（可选）：
   - 勾选 **GitHub 项目**，项目 URL 填：`https://github.com/你的用户名/demo_jenkins`（或你的仓库地址）

2. **Pipeline** 区域（请向下滚动到页面下方的「Pipeline」区块）：
   - **Definition**（定义 / 流水线定义）：必须选 **Pipeline script from SCM**（从 SCM 获取流水线脚本）。  
     ⚠️ 若选的是「Pipeline script」，则不会出现 Script Path，因为那是手写脚本模式。
   - **SCM**：选 **Git**
   - **Repository URL**：
     - HTTPS：`https://github.com/你的用户名/demo_jenkins.git`
     - SSH：`git@github.com:你的用户名/demo_jenkins.git`
   - **Credentials**：选择你在 3.4 步添加的凭据
   - **分支**：`*/main` 或 `*/master`（按你默认分支）
   - **Script Path**（脚本路径）：
     - 位置：在「分支」输入框**下方**，同一 Pipeline 区块里。
     - 若没看到：先确认 Definition 选的是 **Pipeline script from SCM** 且 SCM 选的是 **Git**，然后在该区块内**向下滚动**，Script Path 通常在 Git 配置的最下面。
     - 中文界面可能显示为：**脚本路径**。
     - 填写：`Jenkinsfile`（表示使用仓库根目录的 Jenkinsfile）。

3. 保存

这样 Jenkins 会从 GitHub 拉取代码，并执行仓库根目录的 `Jenkinsfile`。

---

## 五、Jenkinsfile 说明（仓库内已提供）

项目根目录的 `Jenkinsfile` 会：

1. 拉取 GitHub 代码
2. 使用 Node 环境
3. 安装依赖（npm 或 pnpm）
4. 执行 `npm run build`（Next.js 构建）
5. 可选：把构建产物归档或部署到指定目录

首次运行前请在 Jenkinsfile 里确认：

- `nodejs('Node 20')` 与你在 **全局工具配置** 里配置的 Node 名称一致
- 若使用 pnpm，需在 Jenkins 中安装 pnpm 或脚本里用 `npm install -g pnpm`

---

## 六、触发方式

### 6.1 手动触发

在任务页点击 **立即构建**。

### 6.2 轮询 SCM（定时检查 GitHub 是否有新提交）

在任务配置 → **Pipeline** 上方或 **构建触发器** 中：

- 勾选 **Poll SCM**
- 日程表填：`H/5 * * * *`（每 5 分钟检查一次）

### 6.3 GitHub Webhook（推荐，提交即构建）

1. GitHub 仓库 → **Settings** → **Webhooks** → **Add webhook**
2. **Payload URL**：`http://你的Jenkins公网IP或域名:8080/github-webhook/`
3. **Content type**：`application/json`
4. **Which events**：Just the push event（或按需选择）
5. 保存

6. Jenkins 任务配置 → **构建触发器** → 勾选 **GitHub hook trigger for GITScm polling**

注意：若 Jenkins 在内网，需用内网穿透或在同一台机上的 Nginx 转发，确保 GitHub 能访问到 Payload URL。

---

## 七、部署到服务器目录（可选）

若希望构建完成后把 Next.js 输出部署到服务器某目录并启动：

1. 在 Jenkinsfile 里增加步骤：把 `out`（静态导出）或 `.next` + `package.json` 等拷贝到目标目录
2. 在目标目录执行 `npm run start` 或使用 pm2：
   - 安装 pm2：`npm install -g pm2`
   - 在 Jenkinsfile 里用 `pm2 restart demo_jenkins` 或 `pm2 start npm --name "demo_jenkins" -- start`

注意：Jenkins 通常以 Jenkins 用户运行，部署目录需对该用户有写权限；若用 pm2，需保证 Jenkins 能调用到 pm2（同一用户或配置好环境）。

---

## 八、常见问题

| 问题 | 处理思路 |
|------|----------|
| 拉不到 GitHub 代码 | 检查凭据、仓库地址、分支名；国内服务器检查网络/代理 |
| 找不到 Node/npm | 全局工具配置中 Node 名称与 Jenkinsfile 中 `nodejs('...')` 一致；或改用系统已安装的 Node |
| 构建失败：权限不足 | 部署目录权限、Jenkins 用户权限 |
| Webhook 不触发 | Payload URL 是否可被 GitHub 访问；Jenkins 是否勾选 “GitHub hook trigger” |

---

## 九、下一步

1. 在 Jenkins 中按第四节创建 Pipeline 任务，Script Path 指向 `Jenkinsfile`
2. 点击 **立即构建** 做一次完整构建
3. 查看 **Console Output** 排查任何错误
4. 再按第六节配置轮询或 Webhook，实现提交后自动构建

如需把构建产物自动部署到指定目录并用 pm2 启动，可说明你的目录和端口，我可以按你的环境改一版 Jenkinsfile 示例。
