# GitHub Market 提交指南

> 用于发布应用到公开 Olares Market

## 提交流程

```
1. Fork 官方仓库
2. 添加应用 Chart
3. 创建 Pull Request
4. GitBot 自动验证
5. 审核合并
6. 应用上线 Market
```

## 官方仓库

https://github.com/beclab/apps

## PR 标题格式

```
[PR_TYPE][FolderName][version]Title Content

示例：
[NEW][myapp][0.1.0]My awesome application
[UPDATE][myapp][0.2.0]Fixed bugs
[SUSPEND][myapp][0.1.0]Temporarily suspend
[REMOVE][myapp][0.1.0]Remove application
```

| 类型 | 说明 |
|------|------|
| `NEW` | 新应用提交 |
| `UPDATE` | 更新现有应用 |
| `SUSPEND` | 暂停分发 |
| `REMOVE` | 永久删除 |

## owners 文件

在 chart 根目录创建 `owners` 文件，每行一个 GitHub 用户名：

```
your-github-username
another-contributor
```

## 资源要求

| 类型 | 格式 | 大小限制 | 尺寸 | 必需 |
|------|------|----------|------|------|
| 图标 | PNG/WEBP | ≤512KB | 256x256 | ✅ |
| 截图 | JPEG/PNG/WEBP | ≤8MB | 1440x900 | 推荐 |
| 特色图 | JPEG/PNG/WEBP | ≤8MB | 1440x900 | 可选 |

## 验证状态

| 标签 | 含义 |
|------|------|
| `PR type` | 标题格式正确 |
| `waiting to submit` | 有问题需修复 |
| `waiting to merge` | 检查通过 |
| `merged` | 已合并 |
| `closed` | 有无法修复的问题 |

## 更新应用

1. 版本号必须大于当前版本
2. 更新 Chart.yaml 和 OlaresManifest.yaml 的版本
3. 使用 `[UPDATE]` PR 类型

## 暂停应用

1. 在 chart 目录创建 `.suspend` 文件
2. 使用 `[SUSPEND]` PR 类型
3. 版本号必须与当前版本一致

## 删除应用

1. 清空 chart 目录所有文件
2. 创建 `.remove` 文件
3. 使用 `[REMOVE]` PR 类型
4. ⚠️ 不可逆操作

## 常见问题

**"Not an owner" 错误：**
- 检查 `owners` 文件是否包含你的 GitHub 用户名
- 用户名区分大小写

**"Version conflict" 错误：**
- Chart.yaml 和 OlaresManifest.yaml 版本必须一致
- UPDATE 时版本必须大于当前版本

**PR 被关闭：**
- 创建新 PR（不要重新打开）
- 修复 PR 评论中提到的问题

## 参考资源

- Market 仓库: https://github.com/beclab/apps
- 开发文档: https://docs.olares.com/developer/
- 提交指南: https://docs.olares.com/developer/develop/submit/
