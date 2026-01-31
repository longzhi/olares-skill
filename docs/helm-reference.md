# Helm 打包参考文档

> 用于发布到 Olares Market 的正式打包方式

## 目录结构

```
myapp/
├── Chart.yaml              # Helm chart 元数据（必需）
├── OlaresManifest.yaml     # Olares 配置（必需）
├── owners                  # 维护者 GitHub 用户名（Market 必需）
├── templates/
│   ├── deployment.yaml
│   └── service.yaml
├── values.yaml             # 默认值（可选）
└── README.md
```

## Chart.yaml

```yaml
apiVersion: v2
name: myapp                    # 小写，字母数字和连字符
version: 0.1.0                 # Chart 版本
appVersion: "1.0.0"            # 应用版本
description: "应用描述"
keywords:
  - productivity
home: https://github.com/user/myapp
icon: https://example.com/icon.png
```

## OlaresManifest.yaml

```yaml
terminusManifest.version: '0.7.1'
terminusManifest.type: app

metadata:
  name: myapp                  # 必须与 Chart.yaml name 匹配
  title: My Application
  description: "应用描述"
  icon: https://example.com/icon.png
  version: 0.1.0               # 必须与 Chart.yaml version 匹配
  categories:
    - Productivity

permission:
  appData: true
  userData:
    - Home
    - Documents

spec:
  requiredMemory: 128Mi
  requiredDisk: 512Mi
  requiredCpu: 100m
  limitedMemory: 512Mi
  limitedCpu: 500m
  supportArch:
    - amd64
    - arm64

entrances:
  - name: myapp-web
    title: My App
    host: myapp-svc            # 对应 service.yaml
    port: 3000
    authLevel: private
    openMethod: default

middleware:
  postgres:
    username: myapp_user
    password: {{ .Values.postgres.password }}
    databases:
      - name: myapp_db
        distributed: false
```

## deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
  namespace: {{ .Release.Namespace }}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      containers:
        - name: {{ .Release.Name }}
          image: "your-registry/image:tag"
          ports:
            - containerPort: 3000
          env:
            - name: DATABASE_URL
              value: "postgresql://{{ .Values.postgres.username }}:{{ .Values.postgres.password }}@{{ .Values.postgres.host }}:{{ .Values.postgres.port }}/{{ .Values.postgres.databases.myapp_db }}"
          volumeMounts:
            - name: app-data
              mountPath: /app/data
      volumes:
        - name: app-data
          hostPath:
            type: DirectoryOrCreate
            path: {{ .Values.userspace.appData }}/{{ .Release.Name }}
```

## service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}-svc
  namespace: {{ .Release.Namespace }}
spec:
  type: ClusterIP
  selector:
    app: {{ .Release.Name }}
  ports:
    - port: 3000
      targetPort: 3000
```

## 系统注入变量

| 变量 | 说明 |
|------|------|
| `.Values.bfl.username` | 当前用户名 |
| `.Values.userspace.Home` | 用户主目录 |
| `.Values.userspace.appData` | 应用数据根目录 |
| `.Values.postgres.host` | PostgreSQL 主机 |
| `.Values.postgres.port` | PostgreSQL 端口 |
| `.Values.postgres.username` | 数据库用户名 |
| `.Values.postgres.password` | 数据库密码（自动生成） |
| `.Values.postgres.databases.<name>` | 数据库名 |

## 调试命令

```bash
# 语法检查
helm lint myapp/

# 渲染模板（调试）
helm template myapp myapp/ --debug

# 打包
helm package myapp/
```
