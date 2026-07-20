# FastAPI + PyWebView 桌面应用开发与打包指南

从零开始构建一个 Python 桌面应用：FastAPI 提供后端 API，PyWebView 提供原生窗口界面，Nuitka 打包成独立 exe。

## 架构概览

```
┌─────────────────────────────────────────────┐
│              PyWebView 窗口                  │
│  ┌─────────────────────────────────────────┐ │
│  │           前端 (HTML/JS/CSS)            │ │
│  │      加载 http://127.0.0.1:8000        │ │
│  └─────────────────────────────────────────┘ │
└─────────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────┐
│           FastAPI (uvicorn)                  │
│  - REST API                                 │
│  - 静态文件服务                              │
│  - WebSocket (可选)                         │
└─────────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────┐
│              数据层                          │
│  - SQLite (SQLModel)                        │
│  - 本地文件系统                              │
└─────────────────────────────────────────────┘
```

## 项目结构

```
project/
├── core/
│   ├── models.py       # SQLModel 数据模型
│   ├── database.py     # 数据库连接
│   └── paths.py        # 路径处理（开发/打包兼容）
├── frontend/
│   └── index.html      # 前端页面
├── assets/
│   └── icon.ico        # 应用图标
├── data/
│   └── app.db          # SQLite 数据库
├── server.py           # FastAPI 入口
├── ui.py               # PyWebView 入口
├── build_nuitka.py     # Nuitka 打包脚本
└── pyproject.toml
```

## 1. 环境搭建

```bash
# 创建项目
mkdir my-app && cd my-app

# 初始化 uv
uv init
uv add fastapi uvicorn sqlmodel pywebview httpx

# 开发依赖
uv add --dev ruff pytest
```

## 2. FastAPI 后端

### server.py

```python
from fastapi import FastAPI
from fastapi.staticfiles import StaticFiles
from pathlib import Path

app = FastAPI()

# 静态文件服务（前端）
frontend_dir = Path(__file__).parent / "frontend"
app.mount("/", StaticFiles(directory=str(frontend_dir), html=True), name="frontend")

# API 示例
@app.get("/api/search")
async def search(q: str):
    # 你的业务逻辑
    return {"results": []}
```

### 路径兼容（关键）

开发和打包后的路径不同，需要统一处理：

```python
# core/paths.py
import sys
from pathlib import Path

if getattr(sys, "frozen", False):
    # Nuitka 打包后
    BASE_DIR = Path(sys.executable).parent
else:
    # 开发环境
    BASE_DIR = Path(__file__).parent.parent

DATA_DIR = BASE_DIR / "data"
ASSETS_DIR = BASE_DIR / "assets"
```

## 3. PyWebView 桌面窗口

### ui.py

```python
import sys
import threading
import time
from pathlib import Path

def find_available_port(default: int = 8000) -> int:
    """检测端口可用性"""
    import socket
    try:
        with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
            s.bind(("127.0.0.1", default))
            return default
    except OSError:
        import random
        return random.randint(8000, 8999)

def run_server(port: int):
    """后台运行 FastAPI"""
    import uvicorn
    from server import app
    uvicorn.run(app, host="127.0.0.1", port=port, reload=False)

if __name__ == "__main__":
    import webview

    port = find_available_port()

    # 启动后端（守护线程）
    server_thread = threading.Thread(target=run_server, args=(port,), daemon=True)
    server_thread.start()
    time.sleep(1)  # 等待服务器启动

    # 创建窗口
    icon_path = str(Path(__file__).parent / "assets" / "icon.ico")
    window = webview.create_window(
        "My App",
        f"http://127.0.0.1:{port}",
        width=1000,
        height=700,
    )

    # 启动 GUI（阻塞）
    webview.start(icon=icon_path)
```

### 关键点

- **端口检测**：避免端口冲突，随机分配可用端口
- **守护线程**：后端线程随主程序退出
- **icon 参数**：设置窗口图标（Windows 需要 ICO 格式）

## 4. Nuitka 打包

### build_nuitka.py

```python
import shutil
import subprocess
import sys
from pathlib import Path

ROOT = Path(__file__).parent

def build(mode: str = "release"):
    build_dir = ROOT / "build"
    if build_dir.exists():
        shutil.rmtree(build_dir)

    is_debug = mode == "debug"

    cmd = [
        sys.executable, "-m", "nuitka",
        "--standalone",
        f"--windows-console-mode={'force' if is_debug else 'disable'}",
        "--company-name=MyCompany",
        "--product-name=MyApp",
        "--product-version=1.0.0",
        f"--output-dir={ROOT / 'build'}",
        "--output-filename=MyApp.exe",
        f"--windows-icon-from-ico={ROOT / 'assets' / 'icon.ico'}",
        "--assume-yes-for-downloads",
        # 包含数据文件
        f"--include-data-dir={ROOT / 'frontend'}=frontend",
        f"--include-data-dir={ROOT / 'assets'}=assets",
        # 包含依赖
        "--include-package=fastapi",
        "--include-package=uvicorn",
        "--include-package=starlette",
        "--include-package=sqlmodel",
        # 排除不需要的
        "--nofollow-import-to=playwright",
        "--nofollow-import-to=PIL",
        # 入口文件
        str(ROOT / "ui.py"),
    ]

    # 包含数据库（如果存在）
    db_path = ROOT / "data" / "app.db"
    if db_path.exists():
        cmd.insert(-1, f"--include-data-dir={ROOT / 'data'}=data")

    if not is_debug:
        cmd.append("--python-flag=-O")

    subprocess.run(cmd, check=True)

if __name__ == "__main__":
    mode = sys.argv[1] if len(sys.argv) > 1 else "release"
    build(mode)
```

### 打包命令

```bash
# Release 构建
python build_nuitka.py release

# Debug 构建（保留控制台）
python build_nuitka.py debug
```

### 输出结构

```
build/
└── MyApp.dist/
    ├── MyApp.exe          # 主程序
    ├── frontend/          # 前端文件
    ├── assets/            # 资源文件
    ├── data/              # 数据库
    ├── _internal/         # Python 运行时和依赖
    └── ...
```

## 5. 常见问题

### 路径问题

打包后 `__file__` 不再可靠，必须用 `sys.executable`：

```python
# 错误
Path(__file__).parent / "data"

# 正确
if getattr(sys, "frozen", False):
    BASE_DIR = Path(sys.executable).parent
else:
    BASE_DIR = Path(__file__).parent.parent
```

### 端口被占用

pywebview 打开窗口时后端可能还没启动：

```python
# 加个延迟
time.sleep(1)  # 等待 uvicorn 启动
webview.start()
```

### 图标不显示

- Windows 必须用 ICO 格式
- 确保 `--windows-icon-from-ico` 路径正确
- pywebview 的 `icon` 参数需要 v6.0+ 支持

### 数据库迁移

打包后数据库路径要跟随程序目录：

```python
from core.paths import BASE_DIR

DB_PATH = BASE_DIR / "data" / "app.db"
```

## 6. 开发工作流

```bash
# 1. 开发阶段
uv run python server.py          # 单独启动 API
uv run python ui.py              # 启动桌面客户端

# 2. 测试
uv run pytest

# 3. 打包
python build_nuitka.py release

# 4. 分发
# 压缩 build/MyApp.dist/ 为 zip 发布
```

## 7. GitHub Release 自动化

```yaml
# .github/workflows/release.yml
name: Build and Release

on:
  push:
    tags: ['v*']

jobs:
  build-windows:
    runs-on: windows-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v4
      - run: uv sync
      - run: python build_nuitka.py release
      - uses: actions/upload-artifact@v4
        with:
          name: windows-build
          path: build/MyApp.dist/

  release:
    needs: build-windows
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
      - uses: softprops/action-gh-release@v2
        with:
          files: windows-build/*
```

## 技术栈

| 组件 | 选型 | 说明 |
|------|------|------|
| 后端框架 | FastAPI | 异步、自动文档、类型安全 |
| ASGI 服务器 | Uvicorn | 高性能、生产就绪 |
| 桌面窗口 | PyWebView | 轻量、原生 WebView |
| 数据库 | SQLite + SQLModel | 零配置、ORM 友好 |
| 打包工具 | Nuitka | 编译为原生代码、体积小 |
| HTTP 客户端 | httpx | 异步支持、现代 API |

## License

MIT
