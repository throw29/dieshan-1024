# 叠山 · 1024 前端技术文档

| 项目 | 内容 |
| --- | --- |
| 文档版本 | 1.0 |
| 对应产品版本 | 0.1.0 |
| 更新日期 | 2026-10-20 |
| 技术栈 | HTML5 / CSS3 / 原生 JavaScript（ES2020+） |
| 运行形态 | 单文件静态页面，无构建步骤 |
| 相关文档 | [游戏设计](GAME_DESIGN.md) · [设计规范](DESIGN_SYSTEM.md) · [测试](TESTING.md) · [部署](DEPLOYMENT.md) |

---

## 1. 总览

### 1.1 技术选择

本项目刻意选择零依赖、零构建的单文件实现：

- **HTML**：语义化页面结构、可聚焦棋盘、模态弹窗、状态播报区域。
- **CSS**：主题变量、响应式布局、纸质纹理、方块色阶、动画与降级。
- **JavaScript**：游戏状态、移动合并算法、DOM 渲染、键盘/指针输入、本地存储与无障碍状态管理。
- **字体**：将 Cormorant Garamond 的 Latin 子集以内嵌 `WOFF2` 形式写入 CSS；中文使用系统字体栈。
- **数据层**：浏览器 `localStorage`，无网络接口。

选择单文件的原因：

1. 用户可直接双击打开，不需要安装、构建或启动服务；
2. 便于作为一份完整的可移植作品分发；
3. 游戏体量较小，无需引入框架；
4. 没有外部字体/图片/脚本请求，离线也能保持完整视觉。

### 1.2 关键约束

- 所有游戏逻辑必须在浏览器端完成。
- 不依赖后端、账号或数据库。
- 不支持低性能场景的复杂 canvas 渲染；当前采用 DOM 渲染即可满足 4 × 4 / 5 × 5。
- 中文排版依赖操作系统字体，无法在所有平台上保证完全相同的字形。

### 1.3 文件组成

```text
index.html
├── <head>
│   ├── 字符集、视口、主题色、SEO 描述
│   ├── data URI favicon
│   └── <style>
│       ├── 内嵌 Cormorant Garamond @font-face + OFL 许可证注释
│       ├── 设计令牌（CSS 变量）
│       ├── 基础样式与装饰氛围
│       ├── 页面布局与组件
│       ├── 棋盘、方块与动画
│       ├── 弹窗
│       ├── 响应式断点
│       └── 减少动态效果适配
├── <body>
│   ├── 跳过链接
│   ├── 背景氛围层、颗粒层
│   ├── 页面主体
│   ├── 胜利/失败弹窗
│   └── 读屏播报区
└── <script>
    └── IIFE：游戏状态、逻辑、渲染、输入、存储、初始化
```

当前 `index.html` 约 **140 KB**，其中约一半体积来自内嵌字体。页面运行时不会发起任何 HTTP(S) 请求。

---

## 2. 运行架构

```text
          ┌──────────────────────────────────────────────┐
          │                    Browser                   │
          │                                              │
Keyboard ─┼─► handleKeyDown ─┐                           │
Pointer  ─┼─► handlePointer* ─┼─► performMove(direction)  │
Button   ─┼─► changeSize/new ─┘            │              │
          │                                ▼              │
          │                     computeMove(direction)     │
          │                                │              │
          │             ┌──────────────────┼────────────┐  │
          │             ▼                  ▼            ▼  │
          │      DOM transform      合并/删除方块    更新状态 │
          │             │                  │            │  │
          │             └──────► 生成新方块 ◄────────────┘  │
          │                                │              │
          │                                ▼              │
          │                    分数 / 手数 / 山径 / 弹窗    │
          │                                │              │
          │                                ▼              │
          │                         localStorage          │
          └──────────────────────────────────────────────┘
```

整个应用运行在一个立即执行函数（IIFE）中，内部状态不暴露到全局作用域，避免与页面其他脚本或浏览器环境发生命名冲突。

---

## 3. DOM 结构

### 3.1 顶层结构

```html
<body>
  <a class="skip-link">跳至棋盘</a>

  <div class="atmosphere">…背景装饰…</div>

  <div class="page">
    <header class="site-header">…标题、引言…</header>

    <main class="layout">
      <aside class="console">
        <section class="score-ledger">…得分…</section>
        <section class="controls">…尺寸 / 新局…</section>
        <section class="guide">…玩法…</section>
      </aside>

      <section class="game">
        <div class="game-top">…手数 / 目标…</div>
        <div class="board-frame">
          <div id="board" class="board" tabindex="0">
            <div id="boardGrid" class="board-grid"></div>
            <div id="tileLayer" class="tile-layer"></div>
          </div>
        </div>
        <div class="game-under">…山径 / 键盘提示…</div>
        <p class="mobile-note">…移动端提示…</p>
      </section>
    </main>

    <footer class="site-footer">…</footer>
  </div>

  <div id="overlay" class="game-overlay" hidden>…模态卡片…</div>
  <div id="announcer" class="sr-only" aria-live="polite"></div>
  <div class="grain"></div>
</body>
```

### 3.2 主要元素职责

| 元素 | ID / Class | 职责 |
| --- | --- | --- |
| 棋盘 | `#board` | 可聚焦的游戏交互区域；监听指针事件、承载网格与方块 |
| 背景网格 | `#boardGrid` | 由 JS 按尺寸生成 `n × n` 个 `.cell` |
| 方块层 | `#tileLayer` | 方块绝对定位容器，与网格共享内边距和间距 |
| 当前得分 | `#score` | 输出当前分数 |
| 最佳成绩 | `#best` | 输出当前棋盘尺寸最佳分数 |
| 得分增量 | `#scoreDelta` | 合并时显示 `+N` |
| 棋盘尺寸按钮 | `[data-size]` | `4` / `5`，通过 `aria-pressed` 标记当前模式 |
| 新局按钮 | `#newGame` | 重置当前棋盘 |
| 手数 | `#moveCount` | 「启 程」或「第 NN 手」 |
| 山径 | `#rail` | 10 个里程碑点：2 ～ 1024 |
| 峰高 | `#peak` | 当前最高方块或登峰文案 |
| 弹窗 | `#overlay` | 胜利 / 失败模态层 |
| 播报区域 | `#announcer` | 屏幕阅读器的动态状态提示 |

棋盘网格与方块层分离，使背景格子在游戏过程中不需要重绘；方块只作为子节点移动到目标位置即可。

---

## 4. 状态模型

### 4.1 应用状态

```js
const state = {
  size: 4,                         // 当前棋盘边长
  bests: { 4: 0, 5: 0 },           // 各尺寸最佳分数
  best: 0,                         // 当前尺寸最佳分数
  score: 0,                        // 当前局分数
  moves: 0,                        // 当前局有效移动次数
  board: [],                       // 长度 size * size 的一维数组，元素为 tile 或 null
  busy: false,                     // 动画 / 异步结算期间锁定输入
  hasWon: false,                   // 本局是否已达成 1024
  over: false,                     // 本局是否已结束
  overlayOpen: false,              // 模态层是否开启
  generation: 0,                   // 对局代际，用于取消过期的异步任务
  tileSeq: 0,                      // 方块自增 ID
  metrics: { cell: 0, gap: 0 },    // 当前像素尺寸与间距
  needsMeasure: false              // 动画期间发生尺寸变化时的待测量标记
};
```

### 4.2 棋盘存储

- `board` 是一个 **一维数组**，长度为 `size * size`，而不是嵌套二维数组。
- 索引公式：

```js
index = row * size + col;
row = Math.floor(index / size);
col = index % size;
```

- 每个有效格子保存一个方块对象；空格为 `null`。
- 移动计算后会创建新的 `board` 数组，再与动画过程同步提交。

### 4.3 方块对象

```js
{
  id: Number,       // 自增唯一 ID
  value: Number,    // 2 的幂，1 / 2 / 4 / 8 ...
  row: Number,      // 当前行
  col: Number,      // 当前列
  el: HTMLElement,  // 方块 DOM（.tile）
  face: HTMLElement // 内部 .tile-face（用于视觉与动画）
}
```

DOM 结构：

```html
<div class="tile" data-value="16" data-digits="2" aria-hidden="true">
  <div class="tile-face">
    <span class="tile-value">16</span>
  </div>
</div>
```

- `.tile` 负责位置与滑动过渡。
- `.tile-face` 负责材质、缩放与合并弹跳。
- `.tile-value` 负责数字排版。

### 4.4 尺寸计算

方块尺寸不使用固定像素，而是根据棋盘实际渲染宽度计算：

```text
cell = (tileLayerWidth - gap * (size - 1)) / size
x    = col * (cell + gap)
y    = row * (cell + gap)
```

其中 `gap` 取自 CSS 自定义属性 `--gap`。测量结果写入 `state.metrics`，同时为每个方块设置 `width`、`height` 与 `transform`。

---

## 5. 游戏逻辑

### 5.1 初始化和新局

`init()` 的流程：

1. 缓存全部 DOM 引用；
2. 从 `localStorage` 读取最佳成绩和棋盘尺寸；
3. 构建山径刻度；
4. 绑定键盘、指针、按钮事件；
5. 更新尺寸按钮和说明；
6. 调用 `startGame({ silent: true, focus: false })`；
7. 注册 `ResizeObserver`；
8. 首次渲染完成。

`startGame(options)` 是新局唯一入口，完成：

- 递增 `generation`，使旧动画 / 延迟任务失效；
- 关闭弹窗并解除页面 `inert`；
- 清空状态：分数、手数、胜负标记、棋盘、方块 DOM；
- 重建背景网格；
- 重新测量棋盘；
- 更新得分、最佳成绩、山径、手数、ARIA 文案；
- 在下一帧生成两枚初始方块（保证布局已完成）。

首次打开页面时不主动朗读新局消息，避免干扰用户；之后的显式操作会通过 `aria-live` 播报。

### 5.2 移动与合并算法

入口函数为 `computeMove(direction)`。它是纯计算逻辑：不修改现有状态、不操作 DOM，返回完整移动结果。

#### 第一步：生成处理线

每条线按 **从目标边缘向内** 的顺序提供坐标：

- `left`：每行从左到右；
- `right`：每行从右到左；
- `up`：每列从上到下；
- `down`：每列从下到上。

这样，越靠近目标边缘的方块越先落位。

#### 第二步：压缩与合并

对每条线：

```text
slots = []
for tile of 该线上的所有方块（按上述顺序）:
    last = slots 的最后一格
    if last 存在 且 last 尚未合并 且 last.value == tile.value:
        last.value *= 2
        last.merged = true
        记录一次合并（survivor = last.tile, consumed = tile）
        得分增加 last.value
    else:
        slots.push({ tile, value: tile.value, merged: false })

for index, slot of slots:
    将 slot.tile 放到该线第 index 个目标坐标
```

关键点：

- `merged` 标记保证每个“槽位”本手最多合并一次。
- 合并后新值立即可用于继续压缩，但带 `merged` 标记的槽不能再吸收其它方块。
- 合并的后者会被移除；前者（survivor）在动画结束后更新数值。

#### 第三步：构建结果

`computeMove` 返回：

```js
{
  nextBoard, // 新的一维棋盘数组（只引用存活方块）
  targets,   // Map<tile, { row, col, z }>，包括被合并方块的终点
  merges,    // [{ survivor, consumed, value }]
  gained,    // 本次移动获得的总分
  moved      // 是否存在有效移动
}
```

- `targets` 包含所有原方块（包括即将消失的方块），用于让两张方块先滑动到同一格再合并。
- `moved` 为假时不修改界面状态，只播放无效回弹。

#### 复杂度

- 每条线只遍历一次，总计 O(n²)，对 4 × 4 和 5 × 5 近乎无感。
- 不需要搜索、递归或额外数据结构。

### 5.3 异步移动流程

`performMove(direction)` 是核心异步流程：

```text
1. 若 busy / over / overlayOpen，则忽略输入
2. 调用 computeMove
3. 若 !moved：触发回弹并返回
4. 捕获 generation，设置 busy = true、aria-busy = true
5. 为方块写入目标 transform；合并方块设置 z-index
6. 等待 SLIDE_MS（150ms；减少动态效果时为 0）
7. 校验 generation，防止旧流程写入新局
8. 提交合并：
   - survivor.value 更新
   - 对应方块更新 data-value / 数字文本 / 数字位数
   - 播放合并缩放动画
   - consumed DOM 移除
9. state.board = nextBoard
10. 同步所有存活方块的 row / col / z-index
11. state.moves += 1；state.score += gained
12. 更新得分 DOM、最佳成绩、手数与山径
13. 在随机空位生成一枚新方块
14. 更新峰高与棋盘 ARIA 标签
15. 若最高方块 >= 1024 且本局尚未胜利：
     延迟 OVERLAY_DELAY_MS 后显示胜利弹窗并返回
16. 若已无可移动空间：
     延迟后显示失败弹窗并返回
17. 等待 SETTLE_MS
18. 校验 generation
19. busy = false、移除 aria-busy
20. 若动画期间发生过 resize，则补做一次棋盘测量
```

相关时间常量：

| 常量 | 常规模式 | 减少动态效果模式 | 用途 |
| --- | ---: | ---: | --- |
| `SLIDE_MS` | 150ms | 0ms | 等待方块滑动到位 |
| `SETTLE_MS` | 90ms | 0ms | 提交后短暂冷却 |
| `OVERLAY_DELAY_MS` | 290ms | 40ms | 胜负弹窗出现前的节奏留白 |
| `SWIPE_THRESHOLD` | 24px | 24px | 最小滑动距离 |

### 5.4 新方块生成

`randomTileValue()`：

```text
roll = Math.random()
roll < 0.08              → 1
0.08 <= roll < 0.88      → 2
roll >= 0.88             → 4
```

`spawnRandomTile(animate)`：

1. 扫描 `state.board` 收集空位索引；
2. 若没有空位，返回 `null`；
3. 在空位中等概率抽取一个；
4. 计算 `row` / `col`；
5. 调用 `createTile()` 创建节点与模型；
6. 写入 `state.board`。

生成后新方块会播放 `.tile--new` 的缩放淡入动画。

### 5.5 可行移动判定

`movesAvailable()`：

- 只要存在任意空位，即返回 `true`；
- 否则检查每个格子右侧与下侧的相邻方块，只要存在同值即返回 `true`；
- 都不满足时返回 `false`。

因为每次有效移动都会至少产生一个空位，生成新方块后最多只填满一个位置，所以失败判定安排在新方块生成之后。

### 5.6 胜利与失败

- 胜利条件：`getMaxTile() >= 1024` 且 `!state.hasWon`。
- 胜利后 `hasWon = true`，本局不会重复显示胜利弹窗。
- 继续游戏会关闭弹窗，恢复输入，并重新检查是否存在可行移动。
- 失败条件：生成新方块后 `movesAvailable() === false`，设置 `state.over = true` 并显示失败弹窗。

### 5.7 分数与最佳成绩

`renderScore(previousScore, gained)`：

- 将最新分数写入 `#score`；
- 播放分数数字的轻微上跳；
- 若本次获得分数，则显示 `+N` 并播放上升淡出动画；
- 如果新分数高于当前尺寸的最佳成绩：
  - 更新内存中的 `state.best` / `state.bests[size]`；
  - 更新 `#best`；
  - 调用 `savePreferences()` 写入本地存储。

### 5.8 新局和切换尺寸

- `changeSize(nextSize)`：
  - 只接受 4 或 5；
  - 当前尺寸相同则直接返回；
  - 更新尺寸、最大成绩、按钮状态、本地偏好；
  - 调用 `startGame()`。
- 点击当前已激活的尺寸不会重启对局。
- 动画过程中切换尺寸是安全的：
  - `startGame()` 递增 `generation`；
  - 旧异步流程在下一个 `await` 后检测代数不一致并退出；
  - 新局 DOM 不会被旧流程覆盖。

---

## 6. 动画与渲染

### 6.1 方块移动

方块元素使用绝对定位，并在内联样式中设置：

```css
transform: translate3d(x, y, 0);
transition: transform 0.15s cubic-bezier(0.22, 0.7, 0.2, 1);
```

- 使用 `translate3d` 让浏览器走合成层，减少布局开销。
- 方块尺寸和内边距由 JS 测量后写入。
- 合并双方会同时移动到同一目标位置：
  - 被合并方 `z-index: 2`；
  - 存活方 `z-index: 3`；
  - 过渡结束后移除被合并方，并将存活方数值更新。

### 6.2 新方块和合并反馈

- `.tile--new .tile-face`：`tileLand` 动画，从 0.36 倍缩放出现并轻微回弹。
- `.tile--merged .tile-face`：`tileMerge` 动画，放大到 1.13 倍后恢复。
- 1024 方块额外有 `goldSheen` 循环流光。
- 得分数字播放 `numberBump`，增量数字播放 `deltaRise`。
- 无效方向播放 `boardNudge`。

### 6.3 测量与响应式重排

`measureBoard()`：

1. 读取 `#tileLayer` 的实际尺寸；
2. 读取 CSS 变量 `--gap`；
3. 计算 `cell`；
4. 写入 `--cell` 供字号计算使用；
5. 给棋盘添加 `.no-motion`，避免重排时触发滑动；
6. 重设每个方块的宽高与 transform；
7. 强制读取 `offsetWidth` 提交布局；
8. 移除 `.no-motion`。

`ResizeObserver` 监听棋盘尺寸变化：

- 非动画期间：通过 `requestAnimationFrame` 防抖测量；
- 动画期间：仅设置 `state.needsMeasure = true`，移动流程结束后补测，避免打断当前动画。

### 6.4 减少动态效果

```css
@media (prefers-reduced-motion: reduce) {
  *,
  *::before,
  *::after {
    animation-duration: 0.001ms !important;
    animation-delay: 0ms !important;
    transition-duration: 0.001ms !important;
  }

  .tile { transition: none !important; }
}
```

JavaScript 在载入时读取 `matchMedia("(prefers-reduced-motion: reduce)")` 并相应设置等待时长，确保不会只“视觉关闭”动画却在逻辑上等待 150ms。

---

## 7. 输入系统

### 7.1 键盘

- `handleKeyDown` 负责监听。
- 支持方向键和 `WASD`（大小写兼容）。
- 带有 `Ctrl` / `Meta` / `Alt` 的按键忽略。
- 弹窗打开时游戏键盘逻辑停止，由 `handleOverlayKeyDown` 接管 `Esc` 和 Tab 焦点循环。
- 对方向键调用 `preventDefault()`，避免页面滚动。
- 输入被 `state.busy` 锁定时静默忽略。

### 7.2 指针与触摸

- 在棋盘上监听 `pointerdown`、`pointerup`、`pointercancel`、`lostpointercapture`。
- 指针按下时记录 `{ x, y, pointerId }` 并尝试 `setPointerCapture()`。
- 指针抬起时计算 `deltaX` / `deltaY`：
  - 小于 24px：视为点击，不移动；
  - 横向位移绝对值较大：左/右；
  - 否则：上/下。
- 仅响应主鼠标键（左键）。
- 点击或滑动棋盘会让棋盘获得焦点，便于键盘用户继续操作。

### 7.3 输入锁与并发保护

- `busy` 阻止动画中的新输入。
- `overlayOpen` 阻止弹窗下方的游戏输入。
- `over` 阻止已结束对局的输入。
- `generation` 是处理并发重置的核心：
  - 每次新局、切换尺寸都会递增；
  - 异步流程在每个关键阶段后校验代数；
  - 代数不一致时直接返回，不再修改 DOM 或状态。

---

## 8. 本地数据与偏好

实现函数：

- `readPreferences()`：读取并校验本地数据；
- `savePreferences()`：写入最佳成绩和尺寸；
- 全部操作包裹在 `try / catch` 中，存储不可用时不阻断游戏。

存储键：

| 键 | 值 | 说明 |
| --- | --- | --- |
| `dieshan.best.v1` | JSON | `{ "4": number, "5": number }` |
| `dieshan.size.v1` | 字符串 | `"4"` 或 `"5"` |

加载时的容错：

- 尺寸解析为整数，仅接受 4 或 5；
- 最佳成绩 JSON 解析失败时回退为 `0`；
- 负数、`NaN` 或非数字值被过滤；
- 最佳成绩使用 `Math.floor()` 规整为整数。

更完整的数据层说明与可选后端设计见 [BACKEND.md](BACKEND.md)。

---

## 9. 弹窗、焦点与状态管理

### 9.1 打开弹窗

`showOverlay(kind)`：

1. 设置 `state.busy = true` 和 `state.overlayOpen = true`；
2. 根据 `kind`（`win` / `lose`）更新：
   - 印章文字（峰 / 止）
   - 标签（山 · 成 / 山 · 止）
   - 标题、正文、得分、最高方块
   - 按钮显隐与样式
3. 显示固定定位遮罩；
4. 将 `.page` 设置为 `inert`，禁止背景交互；
5. 为 `<body>` 添加 `.has-overlay` 阻止页面滚动；
6. 下一帧将焦点移到主要按钮；
7. 通过 `announce()` 播报结果。

### 9.2 焦点圈定

- 弹窗打开时，背景页面不可交互，且不可通过 Tab 进入。
- `handleOverlayKeyDown` 在首尾按钮间循环 Tab 焦点。
- 胜利弹窗中，`Esc` 等同于点击「继续叠山」。
- 关闭弹窗后移除 `inert`，并按操作类型恢复棋盘焦点或保持新游戏布局。

### 9.3 关闭弹窗

`closeOverlay(options)`：

- 隐藏模态层；
- 清除 `overlayOpen`；
- 移除 `inert` 和页面滚动锁定；
- 按需将焦点恢复到棋盘。

注意：关闭弹窗不等于允许继续输入。胜利继续、失败等状态由文档中定义的流程显式处理。

---

## 10. 可访问性实现

| 能力 | 实现 |
| --- | --- |
| 键盘可玩 | 棋盘 `tabindex="0"`，全局方向键处理 |
| 跳过导航 | `.skip-link` 跳到棋盘 |
| 操作语义 | 按钮使用原生 `<button>`，尺寸按钮使用 `aria-pressed` |
| 状态名称 | 棋盘 `aria-label` 随尺寸、最高方块、得分更新 |
| 操作说明 | `aria-describedby` 关联隐藏说明 |
| 快捷键描述 | `aria-keyshortcuts` |
| 动态播报 | `#announcer` 设置 `aria-live="polite"` / `aria-atomic="true"` |
| 动画状态 | 移动期间 `aria-busy="true"` |
| 模态对话框 | `role="dialog"`、`aria-modal="true"`、`aria-labelledby`、`aria-describedby` |
| 背景隔离 | `.page.inert = true` |
| 焦点可见 | 自定义 `:focus-visible` 样式 |
| 减少动效 | `prefers-reduced-motion` + JS 时长降级 |
| 装饰元素 | 全部设置 `aria-hidden="true"`，避免读屏噪音 |

说明：数字方块本身装饰了游戏的视觉状态，但棋盘整体提供当前最高方块、得分和操作说明；这是该视觉型益智游戏的基础无障碍折中。

---

## 11. CSS 架构

完整的视觉规范另见 [DESIGN_SYSTEM.md](DESIGN_SYSTEM.md)。此处记录与实现直接相关的结构。

### 11.1 设计令牌

核心变量：

```css
:root {
  --paper: #ece3d0;
  --paper-high: #f5eddb;
  --paper-deep: #dccdb2;
  --ink: #20251f;
  --ink-soft: #565a4f;
  --ink-faint: #6c6859;
  --cinnabar: #a53c28;
  --cinnabar-deep: #7c2a1c;
  --gold: #c8a159;
  --line: rgba(32, 37, 31, 0.17);
  --font-cn: "STKaiti", "KaiTi", "Kaiti SC", …;
  --font-song: "STSong", "Songti SC", "Noto Serif SC", …;
  --font-num: "Cormorant Garamond", "Bodoni MT", Didot, Georgia, …;
}
```

### 11.2 层级

| z-index | 层 |
| ---: | --- |
| 0 | 背景氛围（等高线、日晕） |
| 2 | 页面内容 |
| 3–4 | 棋盘内框、角花 |
| 1–3 | 棋盘网格、方块层、合并临时层级 |
| 60 | 胜负模态层 |
| 80 | 纸张颗粒覆盖层 |

### 11.3 响应式断点

| 条件 | 布局变化 |
| --- | --- |
| `width > 1100px` | 完整双栏布局，大标题刊头 |
| `961px ≤ width ≤ 1100px` | 双栏收紧，棋盘缩窄 |
| `width ≤ 960px` | 刊头纵向堆叠；信息区分为两栏，棋盘在下方居中 |
| `width ≤ 700px` | 信息区单栏；棋盘近全宽；隐藏长篇指南，显示移动端操作提示 |
| `width ≤ 430px` | 进一步压缩标题、战绩标签与目标装饰；隐藏引言正文 |
| `width ≥ 961px` 且 `height ≤ 960px` | 短屏桌面模式：缩小刊头，按视口高度限制棋盘，确保完整可见 |

棋盘尺寸使用 CSS 变量 `--board-size`：

```css
--board-size: min(608px, calc(100vh - 270px));
```

该值只用于短屏桌面模式，使游戏区域尽量保持在首屏。

### 11.4 方块色阶

方块通过 `data-value` 选择色阶：

| 数值 | 视觉 |
| --- | --- |
| 1–4 | 纸白、砂色，墨字 |
| 8 | 暖赭，深墨字 |
| 16–32 | 淡苔绿至松绿 |
| 64 | 深松绿，米白字 |
| 128–512 | 朱砂至暗绛红，米白字 |
| 1024 | 鎏金渐变 + 流光，深墨字 |
| > 1024 | 深墨底 + 金色字，属于“越峰”状态 |

数字字号通过 `data-digits` 调整，避免 3–6 位数在方块内溢出；超过 6 位时按 6 位规则处理。

### 11.5 字体

- **中文展示**：`STKaiti / KaiTi / Kaiti SC`，用于「叠山」与引文，带来手写楷体的笔意。
- **中文正文**：`STSong / Songti SC / Noto Serif SC / SimSun`，形成书刊感。
- **数字与英文**：内嵌的 Cormorant Garamond 可变字体（字重 300–700），并提供 `Bodoni MT / Didot / Georgia / Times New Roman` 回退。
- 使用 `font-variant-numeric: lining-nums tabular-nums` 保证数字对齐稳定。

### 11.6 动画

| 名称 | 用途 |
| --- | --- |
| `revealUp` | 页面载入错峰上浮 |
| `numberBump` | 得分跳动 |
| `deltaRise` | `+N` 上浮淡出 |
| `tileLand` | 新方块出现 |
| `tileMerge` | 合并弹跳 |
| `boardNudge` | 无效移动 |
| `goldSheen` | 1024 金属流光 |
| `veilIn` | 弹窗遮罩 |
| `cardIn` | 弹窗卡片上浮 |
| `sealIn` | 朱砂印章落下 |

所有动画均为 CSS 实现，无需第三方动效库。

---

## 12. 函数索引

以下函数全部位于 `index.html` 的 IIFE 中，按职责分组。

### 12.1 初始化与 DOM

| 函数 | 作用 |
| --- | --- |
| `init()` | 应用入口：缓存 DOM、读取偏好、绑定事件、构建界面、启动首局、注册尺寸观察 |
| `cacheDom()` | 集中获取并保存页面中需要操作的节点 |
| `observeResize()` | 监听棋盘尺寸变化，处理动画期间的测量延后 |
| `scheduleMeasure()` | 用 `requestAnimationFrame` 防抖触发测量 |
| `buildGrid()` | 根据 `state.size` 生成背景网格 |
| `measureBoard()` | 计算格宽、间距，更新方块像素尺寸与位置 |
| `positionTile(tile, row, col)` | 设置单个方块的宽高和 transform |
| `tileTransform(row, col)` | 根据坐标生成 `translate3d(...)` 字符串 |

### 12.2 对局生命周期

| 函数 | 作用 |
| --- | --- |
| `startGame(options)` | 开始新局，重置状态与界面，生成两枚初始方块 |
| `changeSize(nextSize)` | 切换棋盘大小并开始新局 |
| `updateSizeControls()` | 同步尺寸按钮、说明和最佳成绩模式标签 |
| `readPreferences()` | 读取并校验本地最佳成绩 / 尺寸 |
| `savePreferences()` | 写入本地偏好 |
| `createTile(value, row, col, animate)` | 创建方块模型与 DOM |
| `applyTileValue(tile, value)` | 更新方块数值及对应展示属性 |
| `randomTileValue()` | 按概率返回 1 / 2 / 4 |
| `spawnRandomTile(animate)` | 在随机空位生成新方块 |
| `buildRail()` | 创建 2～1024 的山径刻度 |

### 12.3 移动与状态

| 函数 | 作用 |
| --- | --- |
| `computeMove(direction)` | 纯计算移动结果：压缩、合并、得分与目标位置 |
| `performMove(direction)` | 驱动动画、提交合并、计分、生成和胜负检查的异步流程 |
| `movesAvailable()` | 判断棋盘是否仍有空位或可合并相邻对 |
| `getMaxTile()` | 返回当前棋盘最大值 |
| `renderScore(previousScore, gained)` | 更新当前分数、得分动效和最佳成绩 |
| `updateMoves()` | 更新手数文案 |
| `updateRail(maxTile)` | 更新山径进度与峰高文案 |
| `updateBoardLabel()` | 更新棋盘的无障碍描述 |
| `nudgeBoard()` | 播放无效移动回弹 |

### 12.4 弹窗与输入

| 函数 | 作用 |
| --- | --- |
| `showOverlay(kind)` | 打开胜利 / 失败弹窗，设置内容与焦点 |
| `closeOverlay(options)` | 关闭弹窗、解除背景隔离、按需恢复焦点 |
| `continueAfterWin()` | 关闭胜利弹窗，继续对局或转入失败 |
| `handleKeyDown(event)` | 全局键盘输入 |
| `handleOverlayKeyDown(event)` | 弹窗内 `Esc` 与 Tab 焦点循环 |
| `directionForKey(event)` | 将按键映射为方向 |
| `handlePointerDown(event)` | 记录滑动起点并捕获指针 |
| `handlePointerUp(event)` | 计算滑动方向并触发移动 |
| `clearPointer()` | 清理指针状态 |

### 12.5 工具与辅助

| 函数 | 作用 |
| --- | --- |
| `wait(ms)` | Promise 化延时 |
| `replayClass(element, className, removeAfter)` | 重新触发一次性动画 class |
| `announce(message)` | 通过 ARIA live 区域播报动态状态 |

---

## 13. 性能与兼容性

### 13.1 性能

- 最大方块节点数量仅为 25，移动计算为 O(n²)。
- 方块滑动使用 transform，避免触发布局；不重绘背景网格。
- 棋盘背景和颗粒纹理由 CSS 渐变与内联 SVG 噪点实现，无需图片请求。
- 尺寸测量由 `ResizeObserver` 触发，并用 `requestAnimationFrame` 防抖。
- 字体内嵌减少了外部请求和网络字体加载时间，代价是 HTML 文件体积略大。
- 没有内存缓存或历史快照，状态释放简单。

### 13.2 浏览器兼容

目标为近两年的 Chrome / Edge / Firefox / Safari。

使用到的现代能力：

- Pointer Events；
- `ResizeObserver`；
- CSS 自定义属性、`clamp()`、`min()`；
- CSS `aspect-ratio`；
- `inert`、`aria-*`；
- `backdrop-filter`（仅有视觉增强，不支持时可降级）；
- `prefers-reduced-motion`；
- ES2020 语法（可选链未强制使用，核心为 async/await、Map 等）。

较旧浏览器可能出现的降级：

- 弹窗背景无法完全不可聚焦（`inert` 未支持时）；
- 棋盘尺寸变化不会即时重排（无 `ResizeObserver` 时仍可依赖窗口 resize 事件）；
- 模糊背景、字体渲染差异。

### 13.3 安全与隐私

- 无网络请求，不上传用户数据。
- 动态数字通过 `textContent` 写入，不使用用户可控内容拼接 HTML。
- 棋盘尺寸与生成值均来自内部受控代码。
- 本地存储数据可能被用户或同源脚本修改；当前实现仅读取最佳成绩与尺寸，影响范围限于本游戏界面。
- 无 `eval`、无动态脚本、无第三方追踪。

---

## 14. 修改与扩展指南

### 14.1 修改胜利目标

当前目标值集中在 `WINNING_VALUE = 1024`，但仅修改此常量**不足以**完整切换目标，还需同步更新：

- 顶部「合至 1024」
- 引言与页脚中的目标文案
- 竖印文字「一〇二四」
- 胜利弹窗默认文案
- README 与文档
- 山径刻度数组 `RAIL_VALUES`（当前生成 2 的 1 到 10 次方）
- 1024 专属 CSS 选择器 / 越峰判断
- 游戏标题及元信息

建议将目标值抽取为配置对象，同时生成相应 UI 文案，以支持主题变体。

### 14.2 调整生成概率

修改 `randomTileValue()` 中的阈值即可。需要注意阈值总和为 1，并同步更新设计文档和测试用例。

当前实现：

```js
if (roll < 0.08) return 1;
if (roll < 0.88) return 2;
return 4;
```

### 14.3 增加棋盘尺寸

当前 UI 和存储显式支持 4 与 5。若要增加 6 × 6：

1. 在 HTML 中新增尺寸按钮 `[data-size="6"]`；
2. 扩展尺寸校验（`readPreferences`、`changeSize`）；
3. 扩展 `state.bests` 初始化与读取；
4. 确认 6 × 6 数字字号和布局仍可读；
5. 更新移动端断点和最大棋盘样式；
6. 增加对应测试。

核心移动算法本身基于 `state.size`，不限定为 4 或 5。

### 14.4 调整动画时长

- 逻辑等待常量：`SLIDE_MS`、`SETTLE_MS`、`OVERLAY_DELAY_MS`；
- CSS 动画：搜索 `@keyframes` 与 `.tile` 的 `transition`；
- 修改后同步更新测试和设计文档。

### 14.5 主题配色

全部核心颜色通过 `:root` 变量控制。方块色阶因视觉差异较大，直接定义在 `.tile[data-value="…"]` 中；如果要做完整主题化，建议将色阶也提升为变量。

### 14.6 添加当前对局存档

当前只保存最佳成绩和尺寸。若需要刷新后继续上一局：

1. 序列化状态中的 `size`、`score`、`moves`、`hasWon`、`over` 与棋盘数值矩阵；
2. 存入新的 versioned key，例如 `dieshan.game.v1`；
3. 启动时校验尺寸、方块值、空位和最大方块；
4. 恢复 DOM 与棋盘时禁用过渡，避免重新加载时出现位移动画；
5. 对局结束或主动新局时清理存档。

### 14.7 添加撤销

必须保存移动前的完整棋盘、得分和手数快照；恢复时重建方块 DOM 并重算山径。建议只保留一步撤销，避免状态膨胀。

---

## 15. 已知技术限制

- 单文件带来较高的文件耦合；逻辑函数集中在一个 IIFE 内，未模块化。
- 内嵌字体以 Base64 形式增加 HTML 体积，但换来了完全离线与无外部请求。
- 中文标题依赖系统字体，在缺少楷体的系统上会回退到更普通的宋体或无衬线字体。
- 当前对局不持久化，刷新或关闭页面会丢失进度。
- 无自动化测试脚本随仓库发布；测试报告见 [TESTING.md](TESTING.md)。
- 未集成 Service Worker，离线性来自“单文件自身包含资源”，而非 PWA 缓存。
- 未提供对局回放、网络同步或服务端校验。

---

## 16. 相关文档

- [README](../README.md)
- [游戏设计文档](GAME_DESIGN.md)
- [后端与数据文档](BACKEND.md)
- [设计规范](DESIGN_SYSTEM.md)
- [测试文档](TESTING.md)
- [部署与运维](DEPLOYMENT.md)
- [第三方声明](../THIRD_PARTY_NOTICES.md)

---

<p align="center"><sub>前端文档 · 叠山 0.1.0</sub></p>
