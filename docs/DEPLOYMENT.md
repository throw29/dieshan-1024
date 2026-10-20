# 叠山 · 1024 部署与运维文档

| 项目 | 内容 |
| --- | --- |
| 文档版本 | 1.0 |
| 对应产品版本 | 0.1.0 |
| 更新日期 | 2026-10-20 |
| 部署类型 | 纯静态单文件应用 |
| 后端依赖 | 无 |
| 构建步骤 | 无 |
| 相关文档 | [README](../README.md) · [前端技术](FRONTEND.md) · [后端与数据](BACKEND.md) |

---

## 1. 部署概览

《叠山 · 1024》是纯前端静态页面，生产运行只需要一个文件：

```text
index.html
```

`README.md`、`docs/` 与文档图片不影响游戏运行；若打算同时发布文档，可将整个 `1024/` 目录一起部署。

```text
静态托管 / 文件系统
└── index.html  ──► 浏览器执行全部游戏逻辑
                    └── localStorage 保存本地记录
```

没有构建、数据库、环境变量、API 密钥或服务器端渲染要求。

---

## 2. 本地运行

### 2.1 直接打开

双击 `index.html`：

```text
dieshan-1024\index.html
```

适用场景：

- 个人试玩；
- 离线使用；
- 快速检查视觉与交互。

注意：某些浏览器对 `file://` 下的 `localStorage` 有限制，最佳成绩可能无法保存。需要稳定持久化时，使用本地 HTTP 服务器。

### 2.2 Python 本地服务器

如果系统已安装 Python：

```bash
cd dieshan-1024
python -m http.server 8080
```

打开：

```text
http://127.0.0.1:8080/
```

### 2.3 Node 静态服务器

如已安装 Node.js：

```bash
cd dieshan-1024
npx serve .
```

按照终端提示访问本地地址。

### 2.4 局域网预览

若需要在手机或平板测试，可绑定到局域网地址：

```bash
python -m http.server 8080 --bind 0.0.0.0
```

然后使用电脑的局域网 IP 访问，例如：

```text
http://192.168.1.20:8080/
```

注意：

- 确保防火墙允许该端口的本地访问；
- 只在可信局域网中使用；
- 静态服务器不具备访问控制，不应将开发服务器直接暴露到公网；
- 手机与电脑需在同一网络。

---

## 3. 静态托管部署

### 3.1 部署内容

**最小部署：**

```text
root/
└── index.html
```

**连同文档部署：**

```text
root/
├── index.html
├── README.md
├── CHANGELOG.md
├── THIRD_PARTY_NOTICES.md
└── docs/
```

### 3.2 GitHub Pages

1. 创建一个仓库，将 `index.html` 等内容放入仓库根目录或 `docs/` 目录；
2. 提交并推送到 GitHub；
3. 打开仓库 Settings → Pages；
4. 选择部署分支（例如 `main`）和目录。`index.html` 位于仓库根目录时选择 root；若选择 `/docs`，需要将 `index.html` 一并放入该目录或调整仓库结构；
5. 保存后等待发布；
6. 访问生成的 Pages URL。

示例推送命令：

```bash
cd dieshan-1024
git init
git add .
git commit -m "Release Die Shan 1024 v0.1.0"
git branch -M main
git remote add origin <your-repository-url>
git push -u origin main
```

> 当前目录最初不是 Git 仓库；以上仅为部署参考。如果不需要版本管理，可以只用静态文件上传。

### 3.3 Netlify / Vercel

- **Netlify**：直接把整个 `1024` 文件夹拖入 Netlify Drop，或将仓库连接后设置发布目录为该文件夹；无构建命令。
- **Vercel**：导入仓库，Framework Preset 设为 Other，Build Command 留空，Output Directory 设为项目根目录或 `index.html` 所在目录。
- 也可以将 `index.html` 重命名或上传到任何静态站点的根路径。

### 3.4 Nginx

参考配置：

```nginx
server {
    listen 80;
    server_name game.example.com;

    root /var/www/dieshan;
    index index.html;

    charset utf-8;

    location / {
        try_files $uri $uri/ =404;
    }

    location = /index.html {
        add_header Cache-Control "no-cache, must-revalidate";
    }
}
```

启用 HTTPS 后，建议补充：

```nginx
server {
    listen 443 ssl http2;
    server_name game.example.com;

    ssl_certificate     /etc/letsencrypt/live/game.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/game.example.com/privkey.pem;

    root /var/www/dieshan;
    index index.html;

    charset utf-8;

    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "no-referrer" always;
    add_header Permissions-Policy "camera=(), microphone=(), geolocation=()" always;
    add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; font-src 'self' data:; connect-src 'none'; base-uri 'none'; form-action 'none'; frame-ancestors 'none'" always;
    add_header Cache-Control "no-cache, must-revalidate" always;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

根据证书位置和服务器版本调整配置。

> **Nginx 提示**：`add_header` 不会从上级配置继承到已经自行声明了 `add_header` 的 `location`。如果需要为 `/docs/images/*` 单独设置长缓存，请在该 `location` 中重复所有必要的安全头，或使用独立的 server/配置层统一管理。

### 3.5 Apache

将目录放到 DocumentRoot 下，确保默认索引文件包含 `index.html`：

```apache
DirectoryIndex index.html
```

若需要自定义缓存与安全头，可在 `.htaccess` 中添加，或直接在虚拟主机配置中设置。此处不展开完整 Apache 配置。

### 3.6 IIS（Windows）

1. 将 `1024` 文件夹复制到 IIS 站点根目录，如 `C:\inetpub\wwwroot\1024`；
2. 添加网站或将目录映射为虚拟目录；
3. 默认文档添加 `index.html`；
4. 若同时发布文档截图，确保 `.webp` 的 MIME 类型为 `image/webp`；
5. 配置 HTTPS 证书（如需要）。

游戏本身不依赖任何 IIS 模块。

---

## 4. 缓存策略

### 4.1 当前形态

项目的全部功能都在单个 `index.html` 中，没有需要单独缓存的构建产物。

推荐的缓存策略：

| 资源 | 建议 |
| --- | --- |
| `/index.html` | `Cache-Control: no-cache` 或短缓存（如 5 分钟），确保更新及时 |
| `/docs/images/*.webp` | 长缓存（如 30 天），文件变化时改文件名 |
| `/README.md`、`docs/*.md` | 可短缓存，取决于文档发布方式 |
| 字体 | 已内嵌，无独立请求 |

### 4.2 版本化

发布新版本时可选：

- 使用版本化路径：`/v0.1.0/index.html`；
- 在链接后加版本查询串：`/index.html?v=0.1.0`；
- 保留默认的短缓存策略，让用户下次刷新即可获取新版本。

由于没有构建工具自动生成内容哈希，不推荐对 `index.html` 设置一年期不可变缓存。

### 4.3 静态资源

- 页面没有外部图片、字体或 JS/CSS 文件请求，因此不需要复杂的 CDN 缓存规则。
- 文档中的 WebP 预览图若随文档发布，可按普通静态资源缓存。
- favicon 为内联 data URI。

---

## 5. 安全建议

### 5.1 基础安全头

推荐部署以下响应头：

```http
X-Content-Type-Options: nosniff
Referrer-Policy: no-referrer
Permissions-Policy: camera=(), microphone=(), geolocation=()
Content-Security-Policy: default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; font-src 'self' data:; connect-src 'none'; base-uri 'none'; form-action 'none'; frame-ancestors 'none'
```

如启用 HTTPS：

```http
Strict-Transport-Security: max-age=31536000; includeSubDomains
```

### 5.2 关于内联脚本与 CSP

游戏为了单文件交付使用了内联 `<style>` 和 `<script>`，因此示例 CSP 必须包含 `'unsafe-inline'`。

如果需要更严格的 CSP：

1. 将 CSS、JS 提取为外部文件；
2. 为内联资源计算 hash 并加入 `script-src` / `style-src`；
3. 移除 `'unsafe-inline'`；
4. 重新验证字体 data URI 和 SVG 噪点仍被允许。

游戏不需要联网，理论上 `connect-src 'none'` 可以阻断意外网络请求。

### 5.3 其他建议

- 对静态站点启用 HTTPS；
- 不在游戏目录存放 `.env`、私钥或内部文档；
- 静态服务器仅提供文件读取，不应启用目录列表；
- 不添加第三方统计脚本，保持“无网络请求”的隐私承诺；
- 如果未来接入后端，另行审查 CORS、认证、限流与数据校验，详见 [BACKEND.md](BACKEND.md)。

---

## 6. MIME、字符集与编码

必须确保：

```http
Content-Type: text/html; charset=utf-8
```

- HTML 中已声明 `<meta charset="UTF-8">`；
- 中文文案、排版和印章依赖 UTF-8 正确解码；
- 如果服务器以错误编码返回，标题与中文按钮会出现乱码；
- `.webp`：`image/webp`；
- `.md`：通常为 `text/markdown; charset=utf-8`，不强制；
- 字体已内嵌，无需字体 MIME 配置。

---

## 7. 离线能力

### 7.1 本地离线

直接打开 `index.html` 时，因为字体、样式、脚本、图标全部在文件内，断网也能完整游玩。

### 7.2 线上离线

当前**未实现 Service Worker**，因此：

- 已托管页面在断网后不能保证重新打开；
- 浏览器缓存可能让页面短暂可用，但这不是正式离线能力；
- 若要提供 PWA 级离线，请新增：
  - Web App Manifest；
  - Service Worker；
  - 版本化缓存策略；
  - 更新提示与缓存失效机制；
  - 对 `file://` 和 HTTPS 环境分别测试。

### 7.3 中文字体

- 数字和英文使用内嵌字体，不依赖网络；
- 中文使用系统字体栈（楷体 / 宋体等）；
- 不同操作系统的字形不同，这是预期行为，不属于部署故障。

---

## 8. 发布流程

### 8.1 更新文件

当前没有构建流程，直接编辑 `index.html` 即可。

建议修改后执行：

1. 更新 `CHANGELOG.md`；
2. 如需，更新文档与预览图；
3. 在本地 `file://` 和 HTTP 环境各打开一次；
4. 进行功能回归（见 [TESTING.md](TESTING.md)）；
5. 确认无控制台错误；
6. 上传覆盖线上的 `index.html`；
7. 根据缓存策略清理 CDN 或浏览器缓存。

### 8.2 版本号

游戏本身没有显式版本号 UI，版本由文档维护。建议使用语义化版本：

- `MAJOR`：规则或架构不兼容变化；
- `MINOR`：新增功能、模式；
- `PATCH`：样式、文案或修复。

### 8.3 回滚

静态站点回滚简单：

1. 保留上一版本 `index.html` 备份；
2. 如新版本有问题，将备份文件覆盖回去；
3. 清理缓存或增加新的缓存指纹；
4. 验证页面和本地记录正常。

本地存储的数据结构在版本之间保持兼容；若未来修改存储键，应按版本迁移策略处理。

---

## 9. 运维与监控

### 9.1 当前运维需求

- 无服务器进程；
- 无数据库；
- 无定时任务；
- 无密钥轮换；
- 无用户数据存储；
- 只需保证静态文件可访问。

### 9.2 建议监控

即便没有后端，也可以：

- 使用静态托管自带可用性监控；
- 监控 HTTPS 证书到期；
- 检查 CDN / DNS 配置；
- 保留上一版本的部署包；
- 定期验证根路径返回 `index.html` 且状态码为 200。

### 9.3 日志和用户数据

- 游戏本身不上报日志或分析事件；
- 静态主机可能记录访问 IP 和 User-Agent，这部分由托管平台策略决定；
- 如果对隐私敏感，发布时应明确静态服务器日志策略。

---

## 10. 常见故障排查

| 现象 | 可能原因 | 处理 |
| --- | --- | --- |
| 双击后浏览器空白 | 浏览器禁用 JavaScript 或文件损坏 | 打开控制台查看错误；尝试通过 HTTP 服务加载 |
| 中文显示异常或字体不同 | 系统缺少楷体 / 宋体 | 更换具有对应字体的系统，或调整字体栈 |
| 数字字体回退 | 内嵌字体被浏览器阻止 | 检查 CSP / 文件是否完整；正常浏览器应能加载 data URI 字体 |
| 最佳成绩刷新后消失 | 使用 `file://`、隐私模式、禁用 localStorage 或清除站点数据 | 改用 HTTP(S) 访问；检查浏览器存储设置 |
| 页面提示存储异常但游戏可玩 | localStorage 不可用 | 属预期降级，不影响当前对局 |
| 手机棋盘滑动页面 | 用户在棋盘外滑动 | 页面其余区域可正常滚动；棋盘内滑动用于游戏 |
| 游戏在窄屏显得很长 | 杂志式刊头 + 游戏区 | 属响应式设计；棋盘会随后出现，可纵向滚动 |
| 部署后 CSS/JS 不生效 | 服务器错误重写或 CSP 阻止内联资源 | 检查 HTML 是否完整、CSP 是否允许内联脚本/样式 |
| 页面总是旧版本 | 缓存 | 清理 CDN 缓存或配置 `index.html` 短缓存 |
| WebP 预览图无法显示 | 服务器缺少 MIME | 添加 `image/webp`，或改用 PNG |
| 弹窗遮罩未模糊 | 浏览器不支持 `backdrop-filter` | 视觉降级，不影响功能 |

---

## 11. 备份与迁移

### 11.1 备份内容

至少备份：

- `index.html`；
- `README.md` 与 `docs/`（如需保留文档）；
- 字体许可信息（已内嵌于 `index.html`，但仍建议保留 `THIRD_PARTY_NOTICES.md`）；
- `CHANGELOG.md`。

### 11.2 迁移

因为无构建与环境配置，迁移只需复制目录。若需要保留玩家最佳成绩，本地存储随浏览器/源变化，不能通过复制文件迁移；如需要跨设备迁移，未来必须另加同步后端或导出功能。

---

## 12. 部署检查清单

上线前：

- [ ] `index.html` 已通过本地 `file://` 和 HTTP 打开。
- [ ] 页面加载正常，控制台无错误。
- [ ] 字体正常显示，无外部请求。
- [ ] 键盘 / 滑动 / 鼠标拖动操作正常。
- [ ] 4 × 4 与 5 × 5 均可用。
- [ ] 1024 胜利、继续、失败和重开流程正确。
- [ ] 最佳成绩与尺寸在刷新后保留。
- [ ] 移动端无横向溢出。
- [ ] `Content-Type` 为 UTF-8 HTML。
- [ ] 如启用 CSP，内联样式/脚本和 data URI 字体被允许。
- [ ] HTTPS 证书有效。
- [ ] `index.html` 使用合适的缓存策略，更新后能被用户获取。
- [ ] 已记录版本号和变更日志。
- [ ] 已备份上一版本。

---

## 13. 相关文档

- [README](../README.md)
- [游戏设计文档](GAME_DESIGN.md)
- [前端技术文档](FRONTEND.md)
- [后端与数据文档](BACKEND.md)
- [设计规范](DESIGN_SYSTEM.md)
- [测试文档](TESTING.md)

---

<p align="center"><sub>部署与运维 · 纯静态 · 无后端</sub></p>
