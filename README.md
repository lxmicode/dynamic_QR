太好了，这套“活码”已经整理成一个可直接构建发布的小项目，带 Dockerfile、.dockerignore、compose.yml、.env.example 等，一条命令就能打包镜像并跑起来。默认将 SQLite 数据库存到 /data（用卷持久化）。

项目结构
- live-qr/
  - app/
    - __init__.py
    - main.py
  - requirements.txt
  - Dockerfile
  - .dockerignore
  - compose.yml
  - .env.example
  - README.md

文件内容

1) app/main.py
```python
import os, io, time, sqlite3, string, random
from typing import Optional, Literal
from fastapi import FastAPI, HTTPException, Header, Depends
from fastapi.responses import RedirectResponse, HTMLResponse, PlainTextResponse, StreamingResponse
from pydantic import BaseModel
import qrcode

app = FastAPI(title="Live QR", version="1.0.0")

# 环境变量
ADMIN_TOKEN = os.getenv("ADMIN_TOKEN", "change-me")
BASE_URL = os.getenv("BASE_URL", "https://q.example.com")  # 你的外网访问域名，勿以 / 结尾
DB_PATH = os.getenv("DB_PATH", "/data/codes.db")  # 容器内使用 /data 挂载持久化

# 初始化数据库
def init_db():
    os.makedirs(os.path.dirname(DB_PATH), exist_ok=True) if os.path.dirname(DB_PATH) else None
    conn = sqlite3.connect(DB_PATH)
    conn.execute("""
        CREATE TABLE IF NOT EXISTS codes (
            id TEXT PRIMARY KEY,
            type TEXT NOT NULL CHECK(type IN ('url','html','txt')),
            value TEXT NOT NULL,
            hits INTEGER NOT NULL DEFAULT 0,
            updated_at INTEGER NOT NULL
        )
    """)
    conn.commit()
    conn.close()

init_db()

def db_conn():
    conn = sqlite3.connect(DB_PATH, check_same_thread=False)
    # 提高并发读写的鲁棒性
    conn.execute("PRAGMA journal_mode=WAL;")
    conn.execute("PRAGMA synchronous=NORMAL;")
    conn.execute("PRAGMA busy_timeout=3000;")
    return conn

def gen_id(k=8):
    alphabet = string.ascii_lowercase + string.digits
    return ''.join(random.choice(alphabet) for _ in range(k))

# 认证
def require_auth(authorization: Optional[str] = Header(None)):
    if not authorization or not authorization.startswith("Bearer "):
        raise HTTPException(status_code=401, detail="Unauthorized")
    token = authorization.split(" ", 1)[1].strip()
    if token != ADMIN_TOKEN:
        raise HTTPException(status_code=401, detail="Unauthorized")

# Pydantic 模型
class CreateCode(BaseModel):
    type: Literal['url','html','txt']
    value: str
    id: Optional[str] = None  # 可自定义短码，留空则自动生成

class UpdateCode(BaseModel):
    type: Optional[Literal['url','html','txt']] = None
    value: Optional[str] = None

# 生成二维码 PNG
def make_qr_png(data: str, box_size=10, border=4) -> bytes:
    qr = qrcode.QRCode(
        version=None,
        error_correction=qrcode.constants.ERROR_CORRECT_M,
        box_size=box_size,
        border=border,
    )
    qr.add_data(data)
    qr.make(fit=True)
    img = qr.make_image(fill_color="black", back_color="white")
    buf = io.BytesIO()
    img.save(buf, format="PNG")
    return buf.getvalue()

@app.get("/healthz")
def healthz():
    return {"ok": True}

# 动态内容访问（二维码目标链接）
@app.get("/go/{code_id}")
def go(code_id: str):
    conn = db_conn()
    cur = conn.cursor()
    cur.execute("SELECT type, value FROM codes WHERE id = ?", (code_id,))
    row = cur.fetchone()
    if not row:
        conn.close()
        raise HTTPException(status_code=404, detail="Code not found")

    # 计数（可选）
    cur.execute("UPDATE codes SET hits = hits + 1 WHERE id = ?", (code_id,))
    conn.commit()
    conn.close()

    ctype, value = row
    headers = {"Cache-Control": "no-store"}  # 避免缓存，确保活码实时生效

    if ctype == "url":
        return RedirectResponse(url=value, status_code=302, headers=headers)
    elif ctype == "html":
        return HTMLResponse(content=value, headers=headers)
    elif ctype == "txt":
        return PlainTextResponse(content=value, headers=headers)
    else:
        raise HTTPException(status_code=500, detail="Invalid type")

# 二维码图片（固定不变，适合缓存）
@app.get("/qr/{code_id}.png")
def qr_png(code_id: str):
    # 这里的链接要与扫码入口一致
    target = f"{BASE_URL}/go/{code_id}"
    png = make_qr_png(target)
    headers = {
        "Content-Type": "image/png",
        "Cache-Control": "public, max-age=31536000, immutable",
        "Content-Disposition": f'inline; filename="qr-{code_id}.png"',
    }
    return StreamingResponse(io.BytesIO(png), media_type="image/png", headers=headers)

# 管理：创建活码
@app.post("/api/codes", dependencies=[Depends(require_auth)])
def create_code(payload: CreateCode):
    code_id = payload.id or gen_id()
    now = int(time.time())
    conn = db_conn()
    cur = conn.cursor()
    try:
        cur.execute(
            "INSERT INTO codes (id, type, value, updated_at) VALUES (?,?,?,?)",
            (code_id, payload.type, payload.value, now)
        )
        conn.commit()
    except sqlite3.IntegrityError:
        conn.close()
        raise HTTPException(status_code=409, detail="Code id already exists")
    conn.close()
    return {
        "id": code_id,
        "type": payload.type,
        "value": payload.value,
        "link": f"{BASE_URL}/go/{code_id}",
        "qr": f"{BASE_URL}/qr/{code_id}.png"
    }

# 管理：更新活码
@app.put("/api/codes/{code_id}", dependencies=[Depends(require_auth)])
def update_code(code_id: str, payload: UpdateCode):
    if payload.type is None and payload.value is None:
        raise HTTPException(status_code=400, detail="Nothing to update")
    conn = db_conn()
    cur = conn.cursor()
    cur.execute("SELECT 1 FROM codes WHERE id = ?", (code_id,))
    if not cur.fetchone():
        conn.close()
        raise HTTPException(status_code=404, detail="Code not found")

    fields = []
    params = []
    if payload.type is not None:
        fields.append("type = ?")
        params.append(payload.type)
    if payload.value is not None:
        fields.append("value = ?")
        params.append(payload.value)
    fields.append("updated_at = ?")
    params.append(int(time.time()))
    params.append(code_id)

    cur.execute(f"UPDATE codes SET {', '.join(fields)} WHERE id = ?", params)
    conn.commit()
    conn.close()
    return {"ok": True, "id": code_id}

# 管理：查询活码信息
@app.get("/api/codes/{code_id}", dependencies=[Depends(require_auth)])
def get_code(code_id: str):
    conn = db_conn()
    cur = conn.cursor()
    cur.execute("SELECT id, type, value, hits, updated_at FROM codes WHERE id = ?", (code_id,))
    row = cur.fetchone()
    conn.close()
    if not row:
        raise HTTPException(status_code=404, detail="Code not found")
    id_, ctype, value, hits, updated_at = row
    return {
        "id": id_,
        "type": ctype,
        "value": value,
        "hits": hits,
        "updated_at": updated_at,
        "link": f"{BASE_URL}/go/{id_}",
        "qr": f"{BASE_URL}/qr/{id_}.png"
    }
```

2) app/__init__.py
```python
# empty on purpose
```

3) requirements.txt
```text
fastapi>=0.111
uvicorn[standard]>=0.23
qrcode[pil]>=7.4
```

4) Dockerfile
```dockerfile
FROM python:3.11-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1

WORKDIR /app

# 先装依赖
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 拷贝代码
COPY app ./app

# 数据目录（挂卷）
VOLUME ["/data"]
ENV DB_PATH=/data/codes.db

EXPOSE 8000

# 以非 root 运行更安全
RUN useradd -m -u 10001 liveqr
USER 10001

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

5) .dockerignore
```
__pycache__/
*.pyc
*.pyo
*.pyd
*.db
.env
.venv/
venv/
env/
build/
dist/
.git/
.gitignore
.DS_Store
data/
```

6) compose.yml
```yaml
services:
  live-qr:
    build: .
    image: live-qr:latest
    container_name: live-qr
    restart: unless-stopped
    ports:
      - "8080:8000"
    environment:
      BASE_URL: ${BASE_URL}
      ADMIN_TOKEN: ${ADMIN_TOKEN}
      DB_PATH: /data/codes.db
    volumes:
      - ./data:/data
```

7) .env.example
```
# 你的外部访问域名（不要带斜杠）
BASE_URL=https://q.example.com

# 强口令，建议至少 32 位随机字符串
ADMIN_TOKEN=please-change-this-super-strong-token
```

8) README.md
```markdown
# Live QR（活码最小可用方案）

二维码固定不变，后端按短码动态返回 URL/HTML/TXT。Cloudflare 缓存加速二维码图片，FastAPI + SQLite 提供服务。

## 功能
- 固定二维码图片：`/qr/{code_id}.png`（长期缓存）
- 动态落地：`/go/{code_id}`（禁止缓存，即时生效）
- 管理接口：创建、更新、查询（Bearer Token 保护）
- 轻量存储：SQLite（容器内 `/data/codes.db`）

## 快速开始

### 1) Docker 本地
```bash
cp .env.example .env
# 修改 .env 里的 BASE_URL 与 ADMIN_TOKEN

docker compose up -d --build
# 访问 http://localhost:8080/healthz
```

或使用 docker run:
```bash
docker build -t live-qr:latest .
docker run -d --name live-qr \
  -p 8080:8000 \
  -e BASE_URL=https://q.example.com \
  -e ADMIN_TOKEN=your-strong-token \
  -e DB_PATH=/data/codes.db \
  -v $(pwd)/data:/data \
  live-qr:latest
```

### 2) 创建活码
```bash
curl -X POST http://localhost:8080/api/codes \
  -H "Authorization: Bearer YOUR_ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"type":"url","value":"https://www.example.com"}'
```

返回示例：
```json
{
  "id":"a1b2c3d4",
  "type":"url",
  "value":"https://www.example.com",
  "link":"https://q.example.com/go/a1b2c3d4",
  "qr":"https://q.example.com/qr/a1b2c3d4.png"
}
```

### 3) 更新为 HTML / TXT（二维码不变）
```bash
curl -X PUT http://localhost:8080/api/codes/a1b2c3d4 \
  -H "Authorization: Bearer YOUR_ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"type":"html","value":"<h1>Hello 活码</h1><p>这是一段动态 HTML</p>"}'

curl -X PUT http://localhost:8080/api/codes/a1b2c3d4 \
  -H "Authorization: Bearer YOUR_ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"type":"txt","value":"这是一段纯文本"}'
```

### 4) 访问
- 二维码图片：`https://q.example.com/qr/a1b2c3d4.png`
- 扫码落地：`https://q.example.com/go/a1b2c3d4`

## Cloudflare 配置要点

- DNS：`q.example.com` 指向你的服务，橙色云开启
- SSL/TLS：Full (strict)，Always Use HTTPS 开启
- Cache Rules：
  1. Host = `q.example.com` AND Path matches `/qr/*`
     - Cache: Eligible / Cache Everything
     - Edge TTL: 1 year
     - Respect origin: On（服务已下发 immutable）
  2. Host = `q.example.com` AND Path matches `/go/*`
     - Cache: Bypass / No cache
- WAF：可限制 `/api/*` 仅办公 IP；或依赖 Bearer 鉴权

## 环境变量
- `BASE_URL`（必填）：外网访问域名，例如 `https://q.example.com`
- `ADMIN_TOKEN`（必填）：管理接口 Bearer Token
- `DB_PATH`（可选）：默认 `/data/codes.db`，建议使用卷持久化

## 接口速览
- `GET /healthz`
- `GET /go/{code_id}` 动态返回 URL/HTML/TXT
- `GET /qr/{code_id}.png` 二维码 PNG（长期缓存）
- `POST /api/codes` 创建活码（需 Bearer）
- `PUT /api/codes/{code_id}` 更新活码（需 Bearer）
- `GET /api/codes/{code_id}` 查询活码（需 Bearer）

## 生产建议
- 将 DB 目录挂载到独立磁盘或网络卷，做好备份
- `ADMIN_TOKEN` 使用高强度随机值（>=32 字符）
- 在 Cloudflare/入口层限制管理接口的来源 IP
- 高并发只读 OK；若写入（计数）压力大，可按需：
  - 关闭实时计数或按批量/异步上报
  - 迁移到外部数据库（例如 Postgres）
```

怎么用
- 复制以上文件到同一个目录（live-qr/）。
- 本地跑：docker compose up -d --build。
- 或构建镜像：docker build -t live-qr:latest .，再 docker run。
- 将 BASE_URL 设置成你的外网域名（与二维码内编码一致）。

需要我帮你把这一套做成 GitHub 仓库（含 Actions 自动构建镜像/推送 GHCR/Docker Hub）吗？也可以顺手加上简单的前端后台页。
