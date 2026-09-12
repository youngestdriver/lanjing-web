# 蓝鲸答题助手 — Web / PWA 与桌面版

浏览器客户端与本地 Express 代理（`server.js`），同一份代码打包为 Windows / macOS / Linux 桌面版
（托盘单文件，内嵌 Bun 编译的服务二进制）。

- 启动：`npm ci && npm start`（默认 http://127.0.0.1:3000）
- 测试：`npm test`；语法检查：`npm run check`
- 规则引擎（题型分类）来自主仓 youngestdriver/lanjing_test 的 `lanjing-bank` 包，
  以 git 依赖固定在 `package.json` 里
- 发布：推 main 自动发版（见 `.github/workflows/release.yml`），产物为
  Windows 托盘单文件 ×2 / macOS .app+dmg / Linux 单文件 ×2
- 题库包不在本仓发布，见主仓发布页

面向蓝鲸微课考试流程的第三方学习客户端。本仓是其中的 Web/PWA 实现，同时承载桌面打包壳；
原生 iOS 与原生安卓实现见主仓 youngestdriver/lanjing_test。

> [!IMPORTANT]
> 本项目会把进入考试、提交答案、标记题目和交卷等操作发送到上游服务，可能直接改变账号中的真实考试记录。只应在获得授权的账号和场景中使用，不要把它用于违规答题、未授权访问或公开服务。

## 网络路径

```text
浏览器 / PWA ──> 本地 Express API ──> https://test.lanjingweike.com
```

Web 版必须启动本仓的 `server.js`；客户端本身不直接访问上游。

## 快速开始

环境要求：

- Node.js 22 或更高版本
- 能访问上游测试服务的网络环境

```bash
npm ci
npm start
```

浏览器打开：

```text
http://127.0.0.1:3000
```

可通过 `PORT` 修改端口：

```bash
PORT=43127 npm start
```

服务默认绑定所有网卡接口（`0.0.0.0`），登录后可在“我的 > 局域网访问”关闭，或用 `HOST` 环境变量指定绑定地址（例如仅本机访问）：

```bash
HOST=127.0.0.1 npm start
```

启动时服务会打印本机可用的局域网访问地址（如 `http://192.168.1.5:3000`）并提示风险；`TRUSTED_HOSTS`（逗号分隔）可额外允许指定主机名，如 `TRUSTED_HOSTS=my-mac.local`。

服务启动后可以用无副作用的状态接口确认运行情况：

```bash
curl --fail http://127.0.0.1:3000/api/status
```

未登录时的正常响应为：

```json
{
  "loggedIn": false,
  "hasSavedSession": false
}
```

## 功能

响应式单页应用，可安装为 PWA，无前端构建步骤。覆盖登录、会话恢复、考试列表、开始或继续考试、逐题作答、题目标记、答题卡、交卷和结果展示：

| 能力 | 说明 |
|---|---|
| 登录后首页 | “考试列表 / 练习 / 我的”三个一级入口；桌面为左侧导航，移动端为底部导航 |
| 单选题 | 点击选项后立即判定并提交 |
| 多选题 | 选择多个选项后确认，按完整集合判定并上报 |
| 历史作答 | 恢复状态及用户之前选择的选项 |
| 自动切题 | 用户可配置，默认关闭；手动导航会取消待执行跳转 |
| 列表与失败恢复 | 主动刷新、空态、加载重试及放弃考试后的陈旧记录抑制 |
| 答案上报失败 | 保留本地选择、显示未同步状态并允许重试 |
| 会话持久化 | 进程级全局 Cookie，写入权限为 `0600` 的本地文件 |
| 练习刷题 | 首次使用直连蓝鲸平台爬取全部机考题库到本地（`.local/practice`，逐卷进度与断点续爬），按 大类→题型细分 两级分类离线刷题，组合题材料渲染、按大类随机顺序；“我的”页提供 更新题库/删除题库/爬取日志导出 |
| 云端会话同步 | 可选启用 CookieCloud 同步：登录后自动上传、启动时拉取、设置中手动同步（协议与官方扩展兼容） |

主题、自动下一题、Cookie 云端同步和退出登录集中在“我的”，局域网访问开关仅 Web 提供。Web 额外提供桌面浏览器入口、响应式布局、触控和键盘答题以及可安装的 PWA 应用壳。

## 本地 API

前端通过同源的 `/api` 路由访问本地 Express 代理（`GET /api/status`、`POST /api/login`、`GET /api/exams`、`POST /api/exams/:id/enter`、`GET /api/exams/:id/questions`、`POST /api/exams/:id/answer`、`POST /api/exams/:id/mark`、`GET /api/exams/:id/states`、`POST /api/exams/:id/submit`、`POST /api/logout`、`GET /bank/*`）。

`enter`、`answer`、`mark` 和 `submit` 都可能改变上游状态。其中 `submit` 会结束当前考试，前端的“放弃考试”也使用这条提交路径，不是单纯删除本地记录。

请求、响应、错误处理和上游映射详见 [docs/web-api.md](docs/web-api.md)。

## 测试

```bash
npm ci
npm run check
npm test
npm run test:browser
```

`npm test` 使用 Node 内置测试框架验证解析器和答题纯逻辑；`npm run test:browser` 通过本机 Chrome 验证完整浏览器交互（需要系统已安装 Google Chrome，或通过 `CHROME_PATH` 指定兼容的 Chromium 可执行文件）。测试都不访问真实上游。

## 发布

每次 push 到 `main` 自动发布（版本号自动取最新 tag 的 patch +1），也可手动触发，生成产物并创建带附件的 GitHub Release。桌面发布产物为免安装版（无需安装 Node.js）：

| 平台 | 产物 | 说明 |
|---|---|---|
| Windows | `LanjingQuiz-windows-x64.exe` / `LanjingQuiz-windows-arm64.exe` | 单文件，托盘图标（打开浏览器 / 退出） |
| macOS | `LanjingQuiz-macOS.dmg`（Universal） | 菜单栏图标（打开浏览器 / 退出） |
| Linux | `LanjingQuiz-linux-x64` / `LanjingQuiz-linux-arm64` | 单文件，`chmod +x` 后运行，终端 Ctrl+C 停止 |

托盘/菜单栏图标提供：打开浏览器、设置 Cookie 服务器（CookieCloud 配置，保存后自动同步）、退出。

首次使用：启动后自动打开浏览器；数据保存在本机（Windows `%LOCALAPPDATA%\LanjingQuiz\data`、macOS `~/Library/Application Support/LanjingQuiz`、Linux 程序旁 `.local`）。服务端口固定 3000，被占用时启动失败（Windows 托盘会提示）。未签名提示：macOS 首次打开需右键“打开”；Windows SmartScreen 选“更多信息 → 仍要运行”。

## 安全与限制

> [!WARNING]
> Web 后端使用进程级全局 Cookie 和缓存，只适合本机单用户运行。服务默认绑定所有网卡接口并允许局域网访问（可在“我的 > 局域网访问”关闭），同时拒绝非白名单 Host、跨源和非 JSON 的写请求，但仍没有多用户会话隔离、TLS、CSRF token 或限流。同一局域网的设备都会共享这份会话，请只在可信网络使用，不要通过反向代理把它暴露到公网，也不要共享 `.local/` 和包含会话信息的终端日志。

- 上游地址目前固定为 `https://test.lanjingweike.com`，没有 `.env` 或运行时切换配置。
- PWA 只缓存应用壳和静态资源；登录、试卷、答题与结果流程都依赖网络和上游服务，不能离线答题。
- 后端会把上游 Cookie 明文保存到忽略跟踪的 `.local/session_cookies.txt`；目录权限为 `0700`，文件权限为 `0600`，退出登录会删除该文件。
- `server.js` 的 Cookie 和考试缓存是进程级单例，因此不能作为多用户后端部署。
- 设置保存于 `.local/settings.json`（权限 `0600`）。
- CookieCloud 同步：UUID 与密码需与浏览器扩展一致才能互通；加密在客户端完成，服务端只保存密文，未知算法或解密失败一律不应用；读取接口永不下发密码。退出登录不会清除云端 blob，其他设备保持登录。
- 上游接口和 HTML 结构不属于本仓库控制范围；页面、字段或认证流程变化都可能导致解析失败。
- `tools/login-demo.js` 是历史调试工具，可能直接访问上游，不应作为普通启动命令或无人值守测试执行。

## 项目结构

```text
.
├── lib/                    # 考试页、成绩页与会话解析器；CookieCloud 加密协议
├── public/
│   ├── js/                 # 浏览器应用与可测试答题逻辑
│   ├── index.html          # 单页应用结构
│   ├── styles.css
│   └── sw.js               # Service Worker
├── test/                   # Node 单元测试与浏览器回归
├── tools/login-demo.js     # 敏感历史调试工具
├── desktop/                # 桌面壳（macOS Swift 菜单栏 / Windows C# 托盘）
├── scripts/                # 桌面打包与静态资源包构建脚本
├── assets/                 # 应用与托盘图标、dmg 背景
├── docs/web-api.md         # 本地 API 与上游映射
├── server.js               # 静态服务、会话和 API 代理
├── desktop-entry.js
└── package.json
```

## 许可与免责声明

本仓库当前没有提供开源许可证。公开可见不代表自动授予复制、修改、分发或商业使用权。

本项目仅用于学习、研究和经授权的测试。使用者应自行确认账号权限、平台规则、当地法律和操作后果；维护者不对未授权使用、考试记录变更或由上游服务变化造成的损失负责。
