太棒了！我把它整理成一个可直接推到 GitHub 的仓库模板，内置：
- GitHub Actions：自动构建多架构镜像并推送 GHCR + Docker Hub
- 简单的前端管理页（/admin），可创建/更新/查询/浏览最近更新的活码
- Docker & Compose 一键运行

你可以直接复制以下文件结构到仓库使用。

项目结构
- live-qr/
  - app/
    - __init__.py
    - main.py
    - static/
      - admin.html
      - admin.js
      - admin.css
  - requirements.txt
  - Dockerfile
  - .dockerignore
  - compose.yml
  - .env.example
  - .gitignore
  - LICENSE
  - README.md
  - .github/
    - workflows/
      - docker.yml

文件内容

1) app/main.py
```python
import os, io, time, sqlite3, string, random, pathlib
from typing import Optional, Literal
from fastapi import FastAPI, HTTPException, Header, Depends, Query
from fastapi.responses import (
    RedirectResponse, HTMLResponse, PlainTextResponse,
    StreamingResponse, FileResponse, JSONResponse
)
from fastapi.staticfiles import StaticFiles
from pydantic import BaseModel
import qrcode

# 可选：支持本地开发自动加载 .env（Docker/Compose 不依赖）
try:
    from dotenv import load_dotenv
    load_dotenv()
except Exception:
    pass

app = FastAPI(title="Live QR", version="1.1.0")

# 环境变量
ADMIN_TOKEN = os.getenv("ADMIN_TOKEN", "change-me")
BASE_URL = os.getenv("BASE_URL", "https://q.example.com")  # 外网访问域名，勿以 / 结尾
DB_PATH = os.getenv("DB_PATH", "/data/codes.db")  # 容器内默认 /data

# 初始化数据库
def init_db():
    db_dir = os.path.dirname(DB_PATH)
    if db_dir:
        os.makedirs(db_dir, exist_ok=True)
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
    id: Optional[str] = None  # 可自定义短码，留空自动生成

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

# 管理：列表（新增，方便后台页）
@app.get("/api/codes", dependencies=[Depends(require_auth)])
def list_codes(
    limit: int = Query(50, ge=1, le=200),
    offset: int = Query(0, ge=0),
    q: Optional[str] = Query(None, description="按 id 或 value 模糊查询")
):
    conn = db_conn()
    cur = conn.cursor()

    base_sql = "FROM codes"
    args = []
    if q:
        base_sql += " WHERE id LIKE ? OR value LIKE ?"
        like = f"%{q}%"
        args.extend([like, like])

    # total
    cur.execute(f"SELECT COUNT(1) {base_sql}", args)
    total = cur.fetchone()[0]

    # items
    cur.execute(f"SELECT id, type, value, hits, updated_at {base_sql} ORDER BY updated_at DESC LIMIT ? OFFSET ?", args + [limit, offset])
    rows = cur.fetchall()
    conn.close()

    items = [
        {
            "id": r[0],
            "type": r[1],
            "value": r[2],
            "hits": r[3],
            "updated_at": r[4],
            "link": f"{BASE_URL}/go/{r[0]}",
            "qr": f"{BASE_URL}/qr/{r[0]}.png"
        } for r in rows
    ]
    return {"total": total, "items": items}

# 管理：删除（可选）
@app.delete("/api/codes/{code_id}", dependencies=[Depends(require_auth)])
def delete_code(code_id: str):
    conn = db_conn()
    cur = conn.cursor()
    cur.execute("DELETE FROM codes WHERE id = ?", (code_id,))
    changes = conn.total_changes
    conn.commit()
    conn.close()
    if changes == 0:
        raise HTTPException(status_code=404, detail="Code not found")
    return {"ok": True, "id": code_id}

# 静态资源与后台页
ROOT_DIR = pathlib.Path(__file__).parent.resolve()
STATIC_DIR = ROOT_DIR / "static"
app.mount("/static", StaticFiles(directory=str(STATIC_DIR)), name="static")

@app.get("/admin")
def admin_page():
    return FileResponse(str(STATIC_DIR / "admin.html"))
```

2) app/__init__.py
```python
# empty on purpose
```

3) app/static/admin.html
```html
<!doctype html>
<html lang="zh-CN">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Live QR 管理后台</title>
  <link rel="stylesheet" href="/static/admin.css" />
</head>
<body>
  <header>
    <h1>Live QR 管理后台</h1>
    <div class="kv">
      <label>Base URL</label>
      <input id="baseUrl" placeholder="https://q.example.com" />
      <label>Token</label>
      <input id="token" type="password" placeholder="Bearer Token" />
      <button id="save">保存</button>
    </div>
  </header>

  <main>
    <section>
      <h2>创建活码</h2>
      <div class="form">
        <input id="create-id" placeholder="自定义ID（可空自动生成）" />
        <select id="create-type">
          <option value="url">URL</option>
          <option value="html">HTML</option>
          <option value="txt">TXT</option>
        </select>
        <textarea id="create-value" placeholder="对应内容"></textarea>
        <button id="create-btn">创建</button>
      </div>
      <pre id="create-result"></pre>
    </section>

    <section>
      <h2>更新活码</h2>
      <div class="form">
        <input id="update-id" placeholder="ID" />
        <select id="update-type">
          <option value="">（不改类型）</option>
          <option value="url">URL</option>
          <option value="html">HTML</option>
          <option value="txt">TXT</option>
        </select>
        <textarea id="update-value" placeholder="新的内容（留空则不改）"></textarea>
        <button id="update-btn">更新</button>
      </div>
      <pre id="update-result"></pre>
    </section>

    <section>
      <h2>查询/预览</h2>
      <div class="form">
        <input id="get-id" placeholder="ID" />
        <button id="get-btn">查询</button>
      </div>
      <div id="get-view" class="card hidden">
        <div class="row">
          <img id="get-qr" alt="QR" />
          <div>
            <div><b>ID:</b> <span id="v-id"></span></div>
            <div><b>类型:</b> <span id="v-type"></span></div>
            <div><b>访问:</b> <a id="v-link" target="_blank" rel="noreferrer">落地链接</a></div>
            <div><b>二维码:</b> <a id="v-qr" target="_blank" rel="noreferrer">PNG</a></div>
            <div><b>点击:</b> <span id="v-hits"></span></div>
            <div><b>更新时间:</b> <span id="v-updated"></span></div>
          </div>
        </div>
        <div class="value-box">
          <b>当前内容:</b>
          <pre id="v-value"></pre>
        </div>
      </div>
    </section>

    <section>
      <h2>最近更新</h2>
      <div class="form">
        <input id="search-q" placeholder="搜索（按ID/内容）" />
        <button id="search-btn">刷新</button>
      </div>
      <table id="list-table">
        <thead>
          <tr>
            <th>ID</th>
            <th>类型</th>
            <th>点击</th>
            <th>更新时间</th>
            <th>操作</th>
          </tr>
        </thead>
        <tbody id="list-body"></tbody>
      </table>
      <div id="list-foot"></div>
    </section>
  </main>

  <script src="/static/admin.js"></script>
</body>
</html>
```

4) app/static/admin.css
```css
:root {
  --bg: #0f172a;
  --fg: #e2e8f0;
  --muted: #94a3b8;
  --accent: #22c55e;
  --card: #111827;
  --border: #334155;
}
* { box-sizing: border-box; }
body {
  margin: 0; font: 14px/1.6 -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, "PingFang SC", "Noto Sans CJK", "Microsoft YaHei", sans-serif;
  background: var(--bg); color: var(--fg);
}
header {
  padding: 16px; border-bottom: 1px solid var(--border);
  position: sticky; top: 0; background: rgba(15,23,42,.8); backdrop-filter: blur(6px);
}
h1 { margin: 0 0 8px; font-size: 20px; }
h2 { margin: 24px 0 8px; }
.kv { display: flex; gap: 8px; flex-wrap: wrap; align-items: center; }
.kv input { padding: 8px; border: 1px solid var(--border); border-radius: 6px; background: var(--card); color: var(--fg); min-width: 240px; }
.kv button { padding: 8px 12px; background: var(--accent); color: #002; border: 0; border-radius: 6px; cursor: pointer; }

main { padding: 16px; max-width: 1100px; margin: 0 auto; }
section { margin-bottom: 32px; padding-bottom: 16px; border-bottom: 1px dashed var(--border); }

.form { display: flex; gap: 8px; flex-wrap: wrap; }
.form input, .form select, .form textarea {
  padding: 8px; border: 1px solid var(--border); border-radius: 6px; background: var(--card); color: var(--fg);
}
.form textarea { width: 100%; min-height: 120px; }

button { padding: 8px 12px; background: var(--accent); color: #002; border: 0; border-radius: 6px; cursor: pointer; }

pre {
  margin: 8px 0; padding: 12px; background: #0b1324; color: #d1e7ff;
  border: 1px solid var(--border); border-radius: 6px; overflow: auto;
}

.card { padding: 12px; border: 1px solid var(--border); border-radius: 8px; background: var(--card); }
.hidden { display: none; }
.row { display: flex; gap: 16px; align-items: center; }
.row img { width: 160px; height: 160px; border: 1px solid var(--border); background: white; }

.value-box { margin-top: 12px; }

table { width: 100%; border-collapse: collapse; }
th, td { text-align: left; padding: 8px; border-bottom: 1px solid var(--border); }
td a { color: #93c5fd; text-decoration: none; }
td .danger { background: #ef4444; color: #fff; border-radius: 6px; padding: 6px 10px; }
```

5) app/static/admin.js
```javascript
const $ = (s) => document.querySelector(s);
const fmtTime = (ts) => new Date(ts * 1000).toLocaleString();

const state = {
  baseUrl: localStorage.getItem('liveqr_base') || (location.origin),
  token: localStorage.getItem('liveqr_token') || '',
};

const headers = () => ({
  'Content-Type': 'application/json',
  'Authorization': `Bearer ${state.token}`,
});

function setSaved() {
  $('#baseUrl').value = state.baseUrl;
  $('#token').value = state.token;
}

async function api(path, opts = {}) {
  const url = new URL(path, state.baseUrl);
  const res = await fetch(url.toString(), {
    ...opts,
    headers: { ...headers(), ...(opts.headers || {}) },
  });
  if (!res.ok) {
    const text = await res.text();
    throw new Error(`HTTP ${res.status}: ${text}`);
  }
  const ct = res.headers.get('content-type') || '';
  return ct.includes('application/json') ? res.json() : res.text();
}

function show(obj) {
  return JSON.stringify(obj, null, 2);
}

async function create() {
  try {
    const payload = {
      id: $('#create-id').value || undefined,
      type: $('#create-type').value,
      value: $('#create-value').value,
    };
    const data = await api('/api/codes', {
      method: 'POST',
      body: JSON.stringify(payload),
    });
    $('#create-result').textContent = show(data);
    $('#get-id').value = data.id;
    await getById();
    await refreshList();
  } catch (e) {
    $('#create-result').textContent = e.message;
  }
}

async function update() {
  try {
    const id = $('#update-id').value.trim();
    if (!id) return alert('请输入 ID');
    const t = $('#update-type').value;
    const v = $('#update-value').value;
    const payload = {};
    if (t) payload.type = t;
    if (v) payload.value = v;
    if (!payload.type && !payload.value) return alert('没有更新内容');
    const data = await api(`/api/codes/${id}`, {
      method: 'PUT',
      body: JSON.stringify(payload),
    });
    $('#update-result').textContent = show(data);
    $('#get-id').value = id;
    await getById();
    await refreshList();
  } catch (e) {
    $('#update-result').textContent = e.message;
  }
}

async function getById() {
  try {
    const id = $('#get-id').value.trim();
    if (!id) return alert('请输入 ID');
    const data = await api(`/api/codes/${id}`);
    $('#get-view').classList.remove('hidden');
    $('#get-qr').src = data.qr;
    $('#v-id').textContent = data.id;
    $('#v-type').textContent = data.type;
    $('#v-link').href = data.link;
    $('#v-qr').href = data.qr;
    $('#v-hits').textContent = data.hits;
    $('#v-updated').textContent = fmtTime(data.updated_at);
    $('#v-value').textContent = data.value;
  } catch (e) {
    alert(e.message);
  }
}

async function refreshList() {
  const q = $('#search-q').value.trim();
  const data = await api(`/api/codes?limit=50${q ? '&q=' + encodeURIComponent(q) : ''}`);
  const tbody = $('#list-body');
  tbody.innerHTML = '';
  for (const it of data.items) {
    const tr = document.createElement('tr');
    tr.innerHTML = `
      <td><a href="${it.qr}" target="_blank" rel="noreferrer">${it.id}</a></td>
      <td>${it.type}</td>
      <td>${it.hits}</td>
      <td>${fmtTime(it.updated_at)}</td>
      <td>
        <a href="${it.link}" target="_blank" rel="noreferrer">落地</a>
        &nbsp;|&nbsp;
        <button class="danger" data-id="${it.id}">删除</button>
      </td>
    `;
    tbody.appendChild(tr);
  }
  tbody.querySelectorAll('button.danger').forEach(btn => {
    btn.addEventListener('click', async () => {
      if (!confirm(`确认删除 ${btn.dataset.id} ?`)) return;
      try {
        await api(`/api/codes/${btn.dataset.id}`, { method: 'DELETE' });
        await refreshList();
      } catch (e) {
        alert(e.message);
      }
    });
  });
  $('#list-foot').textContent = `共 ${data.total} 条，当前显示 ${data.items.length} 条`;
}

function init() {
  setSaved();
  $('#save').addEventListener('click', () => {
    state.baseUrl = $('#baseUrl').value.trim().replace(/\/+$/, '');
    state.token = $('#token').value.trim();
    localStorage.setItem('liveqr_base', state.baseUrl);
    localStorage.setItem('liveqr_token', state.token);
    alert('已保存');
  });

  $('#create-btn').addEventListener('click', create);
  $('#update-btn').addEventListener('click', update);
  $('#get-btn').addEventListener('click', getById);
  $('#search-btn').addEventListener('click', refreshList);

  refreshList();
}

document.addEventListener('DOMContentLoaded', init);
```

6) requirements.txt
```text
fastapi>=0.111
uvicorn[standard]>=0.23
qrcode[pil]>=7.4
python-dotenv>=1.0
```

7) Dockerfile
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

8) .dockerignore
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
node_modules/
```

9) compose.yml
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

10) .env.example
```
# 外部访问域名（不要带斜杠）
BASE_URL=https://q.example.com

# 管理接口 Bearer Token（请改为强口令，>=32位）
ADMIN_TOKEN=please-change-this-super-strong-token
```

11) .gitignore
```
# Python
__pycache__/
*.py[cod]
*.so
*.egg-info/
.venv/
venv/
env/

# Project
data/
*.db
.env
.DS_Store
```

12) LICENSE
```
MIT License

Copyright (c) 2025

Permission is hereby granted, free of charge, to any person obtaining a copy
...
```

13) README.md
```markdown
# Live QR（活码最小可用方案）

二维码固定，后端动态返回 URL/HTML/TXT。支持 Cloudflare 缓存加速二维码图片，FastAPI + SQLite 提供服务。自带后台页面 `/admin` 管理。

## 功能
- 固定二维码：`/qr/{code_id}.png`（immutable，适合长缓存）
- 动态落地：`/go/{code_id}`（禁用缓存）
- 管理接口：创建/更新/查询/列表/删除（Bearer 鉴权）
- 后台页面：`/admin`（本地保存 BaseURL + Token，直接调用接口）
- Docker 一键运行，GitHub Actions 自动构建推送镜像（GHCR + Docker Hub）

## 快速开始
```bash
cp .env.example .env
# 修改 BASE_URL 和 ADMIN_TOKEN

docker compose up -d --build
# 打开 http://localhost:8080/admin
```

## REST API
- `GET /healthz`
- `GET /go/{code_id}` 动态返回 URL/HTML/TXT
- `GET /qr/{code_id}.png` 二维码 PNG（长期缓存）
- `POST /api/codes` 创建（需 Bearer）
- `PUT /api/codes/{code_id}` 更新（需 Bearer）
- `GET /api/codes/{code_id}` 查询（需 Bearer）
- `GET /api/codes?limit=&offset=&q=` 列表（需 Bearer）
- `DELETE /api/codes/{code_id}` 删除（需 Bearer）

## Cloudflare 建议
- DNS：`q.example.com` → 你的服务（橙色云开启）
- SSL/TLS：Full (strict)，Always Use HTTPS 开启
- Cache Rules：
  1) Host=`q.example.com` AND Path=`/qr/*` → Cache Everything，Edge TTL 1y，Respect origin: On
  2) Host=`q.example.com` AND Path=`/go/*` → Bypass
- WAF：限制 `/api/*` 来源 IP 或仅内网访问

## 构建与运行
- 本地：
  ```bash
  uvicorn app.main:app --host 0.0.0.0 --port 8000
  # 需先导出环境变量或使用 python-dotenv
  ```
- Docker：
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

## 安全与生产建议
- 使用高强度 `ADMIN_TOKEN`（≥32字符）
- 后台/API 最好只在内网访问或 WAF 限制 IP
- DB 目录挂载持久化并有备份；高写入需求可迁移到外部 DB
- 如需更高并发计数，建议异步/批量上报或关闭实时计数

## 许可
MIT
```

14) .github/workflows/docker.yml
```yaml
name: Build and Push Docker Image

on:
  push:
    branches: [ "main" ]
    tags: [ "v*", "release-*" ]
  workflow_dispatch:

env:
  IMAGE_NAME: live-qr

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write    # 推送 GHCR 需要
      id-token: write
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Login to Docker Hub
        if: secrets.DOCKERHUB_USERNAME && secrets.DOCKERHUB_TOKEN
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Docker metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: |
            ghcr.io/${{ github.repository_owner }}/${{ env.IMAGE_NAME }}
            docker.io/${{ secrets.DOCKERHUB_USERNAME || 'docker' }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=ref,event=tag
            type=sha
            type=raw,value=latest,enable={{is_default_branch}}

      - name: Build and push
        uses: docker/build-push-action@v6
        with:
          context: .
          file: ./Dockerfile
          push: true
          platforms: linux/amd64,linux/arm64
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
```

使用 GH Actions（推送 GHCR + Docker Hub）
- 在 GitHub 仓库 Settings → Secrets and variables → Actions 添加：
  - DOCKERHUB_USERNAME
  - DOCKERHUB_TOKEN（Docker Hub 的 Access Token）
- 推送分支 main 会出镜像：
  - ghcr.io/<你的 GitHub 用户或 org>/live-qr:latest
  - docker.io/<你的 Docker Hub 用户>/live-qr:latest
- 推送 tag（如 v1.0.0）会额外生成相应 tag 镜像
- 注意：首次在 GHCR 需要把包可见性设置为 public（Packages → 选择包 → Change visibility）

怎么用
- 初始化仓库：把以上文件放进新仓库，git push
- 配置 Actions Secrets（Docker Hub 可选）
- docker compose up -d --build
- 打开 http://你的域名/admin，填 BaseURL 和 Token 直接管理

需要我帮你把它发布到一个实际的 GitHub 仓库，并预置 Actions、说明和版本标签吗？我也可以根据你的域名和镜像命名做细节定制（比如 org 名称、镜像前缀、UI 主题等）。
