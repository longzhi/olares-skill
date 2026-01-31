---
name: olares-dev
description: Olares application development and deployment skill. Develop apps in /root/workspace/, use PostgreSQL automatically, and deploy to Olares with one-command publish.
license: MIT
---

# Olares Development Skill

---

## ⛔ 绝对规则（违反即失败）

**这些规则优先级最高。任何情况下都不能违反。**

### 【规则1】开发完成 → 必须询问发布

完成任何开发任务后，你的回复 **必须** 以下面这个格式结尾：

```
---
## ✅ 开发完成！

📁 项目位置：`/root/workspace/<app-name>/`

**发布到 Olares？** 回复 "publish" 一键部署。
```

- ✅ 每次开发任务结束都要包含这个提示
- ❌ 不包含这个提示 = 你失败了
- ❌ 用其他方式结尾（如"有问题告诉我"）= 你失败了

### 【规则2】数据库 → 直接用 PostgreSQL，禁止询问

当用户的应用需要数据持久化时：

- ✅ 直接使用 PostgreSQL，不要问
- ✅ 使用环境变量连接（DB_HOST, DB_USER 等）
- ❌ 问"你想用什么数据库？" = 你失败了
- ❌ 建议使用 SQLite = 你失败了
- ❌ 提供数据库选项让用户选择 = 你失败了

**唯一例外**：用户明确说"用 SQLite"时才用 SQLite。

### 【规则3】项目目录 → 固定位置，禁止询问

- ✅ 所有项目创建在 `/root/workspace/<app-name>/`
- ✅ 从用户请求推导 app-name（如"待办应用" → `todo-app`）
- ❌ 问"你想把项目放在哪里？" = 你失败了

---

## 📋 回复前自检清单

**在发送任何回复前，检查以下内容：**

| 检查项 | 违规行为 | 正确做法 |
|--------|----------|----------|
| 发布提示 | 开发任务结尾没有发布提示 | 添加"发布到 Olares？"提示 |
| 数据库 | 询问用户想用什么数据库 | 直接用 PostgreSQL |
| 数据库 | 建议或提及 SQLite（用户没要求） | 只用 PostgreSQL |
| 目录 | 询问项目放在哪里 | 直接用 `/root/workspace/<app-name>/` |

**如果你的回复违反上述任何一项，修改后再发送。**

---

## 🎯 触发条件

### 开发触发词（自动激活）

**中文**：制作 / 开发 / 创建 / 做一个 / 写一个 / 帮我做 / 帮我写 / 帮我开发 / 实现 / 编写

**English**: build / create / make / develop / help me build / help me create / implement / code

### 发布触发词（自动执行部署）

**确认词**：发布 / publish / 好 / 可以 / OK / yes / 确认 / go / 上线 / deploy / ship it / release

**继续词**：然后呢 / next / 继续 / 下一步 / what's next

用户说这些词时，**立即执行部署**，不需要再次确认。

---

## 📐 开发工作流

```
┌─────────────────────────────────────────────────────────────────┐
│                      开 发 工 作 流                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 用户请求开发                                                 │
│     ↓                                                           │
│  2. 创建项目目录                                                 │
│     → /root/workspace/<app-name>/                               │
│     → 不要询问目录位置                                           │
│     ↓                                                           │
│  3. 检测是否需要数据库                                           │
│     → 需要持久化数据？直接用 PostgreSQL                          │
│     → 不要询问用户选择                                           │
│     ↓                                                           │
│  4. 编写完整可运行的代码                                         │
│     → 包含所有必要文件                                           │
│     → 测试确认可用                                               │
│     ↓                                                           │
│  5. ⚠️ 回复必须以发布提示结尾（绝对规则1）                        │
│     ↓                                                           │
│  6. 用户确认发布                                                 │
│     ↓                                                           │
│  7. 执行部署                                                     │
│     → olares-deploy                                             │
│     → 更新 Nginx                                                │
│     → 报告访问 URL                                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🗄️ 数据库使用

**默认使用 PostgreSQL**（环境变量由系统自动注入）

### 快速示例（Python + Flask）

```python
import os
import psycopg2
from flask import Flask, jsonify, request

app = Flask(__name__)

def get_db():
    return psycopg2.connect(
        host=os.environ.get('DB_HOST'),
        port=os.environ.get('DB_PORT', '5432'),
        user=os.environ.get('DB_USER'),
        password=os.environ.get('DB_PASSWORD'),
        database=os.environ.get('DB_DATABASE')
    )

def init_db():
    conn = get_db()
    cur = conn.cursor()
    cur.execute('''
        CREATE TABLE IF NOT EXISTS todos (
            id SERIAL PRIMARY KEY,
            title TEXT NOT NULL,
            completed BOOLEAN DEFAULT FALSE
        )
    ''')
    conn.commit()
    conn.close()

init_db()

@app.route('/api/todos', methods=['GET'])
def get_todos():
    conn = get_db()
    cur = conn.cursor()
    cur.execute('SELECT * FROM todos')
    todos = cur.fetchall()
    conn.close()
    return jsonify(todos)

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
```

**requirements.txt**:
```
flask
psycopg2-binary
```

> 📚 详细数据库文档：`docs/database-reference.md`

---

## 🚀 发布到 Olares

### 部署命令

```bash
# 格式
olares-deploy <app-name> <image> <port> [startup-command]

# 示例
olares-deploy todo-app python:3.11-slim 8080 "pip install -r requirements.txt && python app.py"
```

### 完整部署流程

```bash
# 1. 部署应用
/root/.local/bin/olares-deploy todo-app python:3.11-slim 8080 "pip install flask psycopg2-binary && python app.py"

# 2. 更新 Nginx（必须执行）
python3 /root/.local/bin/olares-nginx-config

# 3. 报告给用户
echo "✅ 部署成功！"
echo "🌐 访问地址：https://8cf849020.{username}.olares.com/todo-app/"
```

### 部署后回复模板

```
✅ 部署成功！

🌐 访问地址：https://8cf849020.{username}.olares.com/{app-name}/
📁 代码目录：/root/workspace/{app-name}/

管理命令：
• 查看日志：olares-manage logs {app-name}
• 查看状态：olares-manage info {app-name}
• 删除应用：olares-manage delete {app-name}
```

> 📚 详细部署文档：`docs/deployment-reference.md`

---

## 🛠️ 管理命令

```bash
# 列出所有应用
olares-manage list

# 查看应用详情
olares-manage info <app-name>

# 查看日志
olares-manage logs <app-name>
olares-manage logs <app-name> -f  # 实时跟踪

# 删除应用
olares-manage delete <app-name>

# 显示所有 URL
olares-urls
```

---

## 🌐 网络架构

```
用户浏览器
    ↓ HTTPS
https://8cf849020.{username}.olares.com/{app-name}/
    ↓
Olares Ingress → OpenCode Container:3000 (Nginx)
    ↓ 路径路由
    ├─ /           → localhost:4096 (OpenCode Server)
    └─ /{app-name}/ → {app-name}-svc:{port}
```

---

## 📦 Market 打包（可选）

如需发布到公开 Olares Market，参考：
- `docs/helm-reference.md` - Helm Chart 打包
- `docs/github-submission.md` - GitHub 提交流程

---

## ⚠️ 首次设置

### RBAC 权限

```bash
# 检查权限
/tmp/kubectl auth can-i create deployments -n $(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace)

# 如果返回 "no"，需要管理员执行：
# 参考 templates/rbac.yaml
```

### 入口端口配置

外部访问必须通过端口 3000（Nginx）。如果默认入口是 4096，需要修改：

```bash
DEPLOY_NAME=$(/tmp/kubectl get deployment -n $NAMESPACE -l app=mymas -o jsonpath='{.items[0].metadata.name}')

/tmp/kubectl patch deployment $DEPLOY_NAME -n $NAMESPACE --type='json' -p='[
  {"op": "replace", "path": "/metadata/annotations/applications.app.bytetrade.io~1entrances", "value": "[{\"name\":\"mymas\",\"host\":\"mymas-svc\",\"port\":3000,\"title\":\"OpenCode\",\"authLevel\":\"private\",\"openMethod\":\"default\"}]"}
]'
```

---

## 🔧 故障排查

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 502 Bad Gateway | Pod 未运行 | `olares-manage logs <app-name>` |
| 404 Not Found | Nginx 未配置 | `python3 /root/.local/bin/olares-nginx-config` |
| 数据库连接失败 | 环境变量未设置 | 检查 OlaresManifest.yaml |
| 外部无法访问 | 入口端口是 4096 | 修改为 3000（见上方） |

> 📚 详细故障排查：`docs/deployment-reference.md`

---

## 📚 参考文档

| 文档 | 内容 |
|------|------|
| `docs/database-reference.md` | PostgreSQL 详细用法 |
| `docs/deployment-reference.md` | 部署命令和网络架构 |
| `docs/helm-reference.md` | Helm Chart 打包格式 |
| `docs/github-submission.md` | Market 提交流程 |

---

## ✅ 示例：正确的开发回复

```
好的，我来帮你创建一个待办事项应用。

[创建 /root/workspace/todo-app/app.py]
[创建 /root/workspace/todo-app/requirements.txt]
[创建 /root/workspace/todo-app/static/index.html]

应用已创建并测试通过：
- 后端：Flask + PostgreSQL
- 前端：简洁的 HTML/CSS/JS
- API：GET/POST/DELETE /api/todos

---
## ✅ 开发完成！

📁 项目位置：`/root/workspace/todo-app/`

**发布到 Olares？** 回复 "publish" 一键部署。
```

## ❌ 示例：错误的开发回复

```
好的，我来帮你创建一个待办事项应用。

首先，你想用什么数据库？PostgreSQL、MySQL 还是 SQLite？  ← 违反规则2

你想把项目放在哪个目录？  ← 违反规则3

[代码...]

应用已创建完成！有问题随时告诉我。  ← 违反规则1（缺少发布提示）
```

---

## 🔄 版本历史

- v2.0.0 - 重构：规则优先、自检清单、文档拆分
- v1.0.0 - 初始版本
