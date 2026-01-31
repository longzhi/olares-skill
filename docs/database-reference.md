# PostgreSQL 数据库参考文档

> 这是详细参考文档。核心规则见主 SKILL.md

## 环境变量

Olares 自动注入以下环境变量：

| 变量 | 说明 | 示例值 |
|------|------|--------|
| `DB_HOST` | PostgreSQL 主机 | `citus-master-svc.user-system-xxx` |
| `DB_PORT` | 端口 | `5432` |
| `DB_USER` | 用户名 | `myapp_xxx_user` |
| `DB_PASSWORD` | 密码 | `XJBVYOFh7Wr...` |
| `DB_DATABASE` | 数据库名 | `myapp_db_xxx` |

## Python 示例

### 使用 psycopg2

```python
import os
import psycopg2
from psycopg2.extras import RealDictCursor

def get_db_connection():
    return psycopg2.connect(
        host=os.environ.get('DB_HOST'),
        port=os.environ.get('DB_PORT', '5432'),
        user=os.environ.get('DB_USER'),
        password=os.environ.get('DB_PASSWORD'),
        database=os.environ.get('DB_DATABASE'),
        cursor_factory=RealDictCursor
    )
```

### 使用 SQLAlchemy

```python
import os
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

DATABASE_URL = (
    f"postgresql://{os.environ['DB_USER']}:{os.environ['DB_PASSWORD']}"
    f"@{os.environ['DB_HOST']}:{os.environ.get('DB_PORT', '5432')}"
    f"/{os.environ['DB_DATABASE']}"
)

engine = create_engine(DATABASE_URL, pool_pre_ping=True)
SessionLocal = sessionmaker(bind=engine)
```

## Node.js 示例

```javascript
const { Pool } = require('pg');

const pool = new Pool({
  host: process.env.DB_HOST,
  port: parseInt(process.env.DB_PORT || '5432'),
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  database: process.env.DB_DATABASE,
});
```

## Go 示例

```go
import (
    "fmt"
    "os"
    "github.com/jackc/pgx/v5/pgxpool"
)

func connectDB() (*pgxpool.Pool, error) {
    connStr := fmt.Sprintf(
        "postgres://%s:%s@%s:%s/%s",
        os.Getenv("DB_USER"),
        os.Getenv("DB_PASSWORD"),
        os.Getenv("DB_HOST"),
        os.Getenv("DB_PORT"),
        os.Getenv("DB_DATABASE"),
    )
    return pgxpool.New(context.Background(), connStr)
}
```

## Schema 隔离（重要）

每个应用必须使用独立的 schema，不要使用 public：

```python
class AppDatabase:
    def __init__(self, app_name: str):
        self.schema = app_name.replace('-', '_')
        self._init_schema()
    
    def _init_schema(self):
        conn = psycopg2.connect(...)
        cur = conn.cursor()
        cur.execute(f'CREATE SCHEMA IF NOT EXISTS {self.schema}')
        conn.commit()
        conn.close()
    
    def get_connection(self):
        conn = psycopg2.connect(
            ...,
            options=f'-c search_path={self.schema}'
        )
        return conn
```

## OlaresManifest.yaml 配置

```yaml
middleware:
  postgres:
    username: myapp_user
    password: {{ .Values.postgres.password }}
    databases:
      - name: myapp_db
        distributed: false
```

## deployment.yaml 环境变量

```yaml
env:
  - name: DB_HOST
    value: "{{ .Values.postgres.host }}"
  - name: DB_PORT
    value: "{{ .Values.postgres.port }}"
  - name: DB_USER
    value: "{{ .Values.postgres.username }}"
  - name: DB_PASSWORD
    value: "{{ .Values.postgres.password }}"
  - name: DB_DATABASE
    value: "{{ .Values.postgres.databases.myapp_db }}"
  - name: DATABASE_URL
    value: "postgresql://{{ .Values.postgres.username }}:{{ .Values.postgres.password }}@{{ .Values.postgres.host }}:{{ .Values.postgres.port }}/{{ .Values.postgres.databases.myapp_db }}"
```

## 故障排查

**连接被拒绝：**
```bash
echo $DATABASE_URL
python -c "import psycopg2; psycopg2.connect('$DATABASE_URL')"
```

**认证失败：**
- 检查 OlaresManifest.yaml 中的 middleware.postgres 配置
- 确认用户名密码与配置匹配

**数据库不存在：**
- 确保 .Values.postgres.databases.<name> 与代码中使用的一致
- 数据库在应用安装时自动创建
