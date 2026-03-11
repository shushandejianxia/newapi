## 本地启动 New API —— 实际执行步骤说明

本文档记录了在本机上**按项目文档要求启动 New API** 时，实际执行过的全部关键操作，方便后续复现或排查问题。

---

### 一、项目与环境确认

- **项目目录**
  - 工作目录：`C:\code\newapi`
  - 目录结构与官方文档一致，包含：`common/`、`controller/`、`relay/`、`web/`、`main.go`、`go.mod` 等。

- **端口占用情况**
  - 使用 PowerShell 检查端口：
    - `3000`：初始为空闲（`FREE 3000`）
    - `5173`：已有进程监听（`LISTEN 5173`），说明本机之前已经启动过前端开发服务器（Vite/Bun）。

---

### 二、环境变量文件配置（`.env`）

- **检查 `.env` 是否存在**
  - 在 `C:\code\newapi` 下检查结果为：`.env` 不存在。

- **创建 `.env` 文件**
  - 参照根目录下的 `.env.example`，创建 `.env` 文件，内容基本保持与示例一致：
    - 大部分配置项保持注释状态；
    - 保留 LinuxDo 的相关默认配置；
    - 未设置 `SQL_DSN`，因此后端默认使用 SQLite 数据库（符合文档中“未配置 SQL_DSN 则默认使用 SQLite” 的行为）。

---

### 三、Go 环境在 Cursor 中的处理

> 现象：你在自己打开的 PowerShell 终端里 `go version` 正常，但 Cursor 启动的 PowerShell 实例一开始识别不到 `go` 命令。

- **查看 PATH 与 `go` 状态**
  - 读取 `$env:Path`，其中包含：
    - `C:\Program Files\Go\bin`
  - 初始 `Get-Command go -All` 报 `CommandNotFoundException`，之后经调整后终端里可以看到：
    - `Application go.exe  Source: C:\Program Files\Go\bin\go.exe`

- **确认 Go 实际安装位置**
  - 检查并直接调用：
    - 路径：`C:\Program Files\Go\bin\go.exe`
    - 命令：`"C:\Program Files\Go\bin\go.exe" version`
    - 输出：`go version go1.26.1 windows/amd64`

- **后续策略**
  - 为避免不同终端之间 PATH 不一致，后续所有 Go 相关命令都统一使用**绝对路径**：
    - `"C:\Program Files\Go\bin\go.exe" mod download`
    - `"C:\Program Files\Go\bin\go.exe" run .`

---

### 四、后端依赖安装（Go Modules）

- **下载 Go 依赖**
  - 在 `C:\code\newapi` 目录执行：

    ```powershell
    "C:\Program Files\Go\bin\go.exe" mod download
    ```

  - 作用：
    - 根据 `go.mod` / `go.sum` 拉取所有后端依赖；
    - 确保后续 `go run .` 不会因缺少依赖包而编译失败。

---

### 五、前端依赖安装与构建（Bun + Vite）

项目前端位于 `web/` 目录，使用 Bun 作为包管理器。

- **确认 Bun 可用**
  - 在 `C:\code\newapi\web` 执行：

    ```powershell
    bun --version
    ```

  - 输出：`1.3.10`，说明 Bun 安装正常且可用。

- **安装前端依赖**
  - 在 `C:\code\newapi\web` 执行：

    ```powershell
    bun install
    ```

  - 结果：
    - 安装 `package.json` 中声明的依赖和开发依赖；
    - 生成/更新 `bun.lockb`；
    - 为构建和开发服务器准备好完整依赖环境。

- **构建前端静态资源**
  - 在 `C:\code\newapi\web` 执行：

    ```powershell
    bun run build
    ```

  - 说明：
    - 实际调用的是 `vite build`；
    - 在 `web/dist` 目录生成生产环境静态资源：
      - `dist/index.html`
      - `dist/assets/...` 各种 JS/CSS/字体等文件。

  - **与后端 `go:embed` 的关系（关键点）**
    - 后端 `main.go` 中存在：

      ```go
      //go:embed web/dist
      var buildFS embed.FS

      //go:embed web/dist/index.html
      var indexPage []byte
      ```

    - 这意味着在运行 `go run .` 时，编译器会尝试将 `web/dist` 目录打包进二进制：
      - 如果 `web/dist` 目录不存在，会报错：
        - `pattern web/dist: no matching files found`
      - 因此**必须先执行一次 `bun run build` 生成 `web/dist`**，后端才能正常启动。

---

### 六、启动后端服务

- **第一次启动尝试（失败原因）**
  - 在尚未构建前端、`web/dist` 不存在的情况下执行：

    ```powershell
    cd C:\code\newapi
    "C:\Program Files\Go\bin\go.exe" run .
    ```

  - 报错信息：
    - `main.go:37:12: pattern web/dist: no matching files found`
  - 原因：
    - `go:embed web/dist` 需要真实存在的 `web/dist` 目录；
    - 这是当前代码结构下的必然要求，并非运行环境问题。

- **构建前端后重新启动后端**
  - 在 `web/` 执行 `bun run build` 生成 `web/dist` 之后，再回到项目根目录执行：

    ```powershell
    cd C:\code\newapi
    "C:\Program Files\Go\bin\go.exe" run .
    ```

  - 以“前台命令 + 后台进程”模式运行（在工具中将其放入后台，避免阻塞其它操作）。

- **验证监听状态**
  - 使用 PowerShell 检查端口：

    ```powershell
    Get-NetTCPConnection -LocalPort 3000 -State Listen -ErrorAction SilentlyContinue
    ```

  - 结果：
    - 稍等一段时间后，看到输出类似：
      - `LISTEN 3000 PID=xxxx`
    - 说明后端服务已成功启动并监听在：
      - `http://localhost:3000`

---

### 七、前端开发服务器状态

- 初始检测时：
  - 端口 `5173` 已有进程监听：
    - `LISTEN 5173 PID=9072`

- 结合 `web/package.json` 的 `scripts` 配置：

  ```json
  {
    "scripts": {
      "dev": "vite",
      "build": "vite build",
      "preview": "vite preview",
      ...
    },
    "proxy": "http://localhost:3000"
  }
  ```

- 这意味着：
  - 你本机已经通过 `bun run dev`（或同等命令）启动了 Vite 前端开发服务器；
  - 前端访问地址为：
    - `http://localhost:5173`
  - 前端通过代理将 API 请求转发到后端：
    - `http://localhost:3000`

---

### 八、最终运行状态概览

- **后端服务**
  - 启动命令（在项目根目录执行）：

    ```powershell
    "C:\Program Files\Go\bin\go.exe" run .
    ```

  - 监听地址：
    - `http://localhost:3000`

- **前端服务**
  - 你本机已有 Vite/Bun 开发服务器在 5173 端口监听；
  - 典型启动命令（位于 `web/` 目录）：

    ```powershell
    bun install      # 首次或依赖变更时执行
    bun run build   # 确保 web/dist 存在（后端 embed 用）
    bun run dev     # 前端开发服务器
    ```

  - 访问地址：
    - `http://localhost:5173`
  - 所有前端调用的 API，会通过代理转发到后端：
    - `http://localhost:3000`

---

### 九、从零复现的一套最简命令（总结版）

如果你在另一台机器上想完全复现当前环境，按下面两组命令执行即可：

1. **后端（窗口 1）**

   ```powershell
   cd C:\code\newapi

   # 第一次拉依赖
   "C:\Program Files\Go\bin\go.exe" mod download

   # 启动后端
   "C:\Program Files\Go\bin\go.exe" run .
   ```

2. **前端（窗口 2）**

   ```powershell
   cd C:\code\newapi\web

   bun install      # 只需首次执行
   bun run build    # 生成 web/dist，供 go:embed 使用
   bun run dev      # 启动前端开发服务器
   ```

至此，本地开发环境就完整跑起来了：  
前端访问 `http://localhost:5173`，后端服务 `http://localhost:3000`。  
