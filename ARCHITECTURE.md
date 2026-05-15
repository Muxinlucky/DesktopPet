# ARCHITECTURE

VibePet 技术白皮书 -- 面向开发者和贡献者。

## 技术栈

| 层 | 技术 |
|---|---|
| 运行时 | Tauri v2 (Rust 后端, WebView2 前端) |
| 前端 | Vite + TypeScript, 零框架 |
| 渲染 | 纯 DOM + CSS `steps()` 精灵图动画 |
| 日历 | lunar-javascript (节气、农历) |
| 音频 | HTML5 Audio API |
| API | OpenAI 兼容 chat completions 格式 |
| 构建 | Cargo (Rust) + Vite (前端) |
| 安装包 | NSIS (可自定义安装目录) |

## 架构决策

### 为什么坚持无框架纯 DOM 渲染

桌面宠物的核心需求极其简单：一张精灵图 + 几个 UI 面板。引入 React/Vue/Pixi/Three 会带来不必要的包体积和运行时开销。纯 DOM + CSS 动画足以胜任，且调试直观、启动极快。

### 为什么不用 Canvas/WebGL

精灵图动画通过 CSS `background-position` + `steps()` 即可实现逐帧播放，无需 Canvas 绘图上下文。DOM 元素天然支持事件捕获、CSS 变换和硬件加速，对桌面宠物场景完全够用。

## 项目结构

```
src/
  pet/                  -- 宠物窗口（透明、无边框、置顶）
    index.html          -- 入口 HTML + 所有面板结构
    pet.ts              -- PetEngine、拖拽、生物钟、气泡、菜单、API、GitHub、代办
    pet.css             -- 宠物窗口全部样式（毛玻璃面板）
    spritesheet.webp    -- 默认精灵图（兜底）
  assets/
    audio/              -- 音效（pop, boing, bubble, bell, crunch）
  config/               -- 配置面板窗口（默认隐藏）
    index.html
    config.ts
    style.css
  lunar-javascript.d.ts -- 日历库类型声明

src-tauri/
  src/
    main.rs             -- Tauri 入口
    lib.rs              -- 插件注册、命令处理、自启动
    pet_import.rs       -- zip 导入、宠物清单、文件管理
  tauri.conf.json       -- 窗口配置、权限、打包设置
  capabilities/
    default.json        -- Tauri 能力权限
  icons/                -- 应用图标（通过 tauri icon 生成）
```

## 多窗口设计

- **`src/pet/`** -- 透明无边框置顶宠物窗口。包含 `pet.ts`（PetEngine + 拖拽）、`pet.css`、`spritesheet.webp`。
- **`src/config/`** -- 默认隐藏的配置面板窗口。包含 `config.ts`、`style.css`。

窗口在 `src-tauri/tauri.conf.json` 中注册，标签分别为 `pet` 和 `config`。Vite 通过 `vite.config.ts` 中的 `rollupOptions.input` 同时构建两个入口。

## PetEngine 精灵图引擎 (`src/pet/pet.ts`)

核心类，驱动所有精灵图动画。使用 `SpriteConfig` 定义帧尺寸和状态到行的映射。`injectKeyframes()` 动态为每个状态创建 CSS `@keyframes`；`applyState()` 将动画应用到精灵元素。

默认回退机制：精灵图的每一行 = 一个状态，每行帧数假设相等。`pet.json` 清单中无需提供状态/帧映射。

```
idle            -> row 0, 6 frames
running-right   -> row 1, 8 frames
waving          -> row 3, 4 frames
jumping         -> row 4, 5 frames
review (think)  -> row 8, 6 frames
...
```

默认图集：8 列 x 9 行，每格 192x208。

## 拖拽系统

基于阈值的状态机：

- **Mousedown** -- "潜伏"阶段，宠物微压（scaleY 拉伸）
- **Mousemove 超过 5px 阈值** -- 进入拖拽模式，切换奔跑动画，通过 `appWindow.startDragging()` 将拖拽控制权交给操作系统
- **Mouseup** -- 松手时播放落地挤压动画，或仅点击时生成粒子

回退机制：当操作系统吞掉 mouseup 事件时（`buttons === 0` 而 `isMouseDown` 为真），自动重置状态。

关键标志位：

```typescript
let isMouseDown = false;
let hasStartedDragging = false;
let isDraggingInProgress = false;
```

`startDragging()` 是异步的 Tauri API，使用 `.then()/.catch()` 而非 `await`，配合 `isDraggingInProgress` 防重入守卫。

## 生物钟系统

通过 Tauri `cursorPosition` API 轮询全局光标位置，监测用户活动：

| 空闲时长 | 行为 |
|---|---|
| 5 分钟 | 切换到"思考"状态 3 秒 |
| 10 分钟 | 进入"等待"状态（趴下睡觉） |
| 鼠标回归 | 跳起惊喜唤醒 |

启动时宠物先挥手打招呼 5 秒，然后进入待机。

## 气泡语录系统

基于当前日期、时间和农历的上下文感知气泡：

**节气：** 冬至 -- 提醒吃饺子或汤圆；立春 -- 新开始祝福

**节日：** 12月13日 -- 国家公祭日；母亲节(5月第2个周日) -- 提醒打电话；春节(农历正月初一) -- 新年祝福；中秋(农历八月十五) -- 月饼提醒

**时间：** 23:00 -- 深夜健康提醒（触发"思考"状态 + 8 秒气泡）

**默认：** 从精选语料库随机抽取温暖问候。

气泡使用 `width: max-content` 自动适配宽度，CSS 三角尾部，平滑淡入淡出过渡。互斥锁防止多个气泡重叠。

## 动态性格引擎

六种内置性格预设，每种有独特语气和说话风格：

| 预设 | 风格 |
|---|---|
| 傲娇猫 | 外冷内热，偷偷关心 |
| 阳光少女 | 活力满满，爱用感叹号 |
| 毒舌朋友 | 吐槽精准，但没有恶意 |
| 温柔前辈 | 暖心耐心，轻声细语 |
| 中二少年 | 中二台词，戏剧独白 |
| 禅系柴犬 | 佛系哲学，惜字如金 |

另有自定义性格面板，可自行编写性格提示词。

## 混合对话模式

两种对话引擎，智能路由：

- **基础模式** -- 从 Hitokoto API 获取随机语句，轻量无配置
- **觉醒模式** -- 连接任意 OpenAI 兼容 LLM API，动态性格对话

觉醒模式中，首次点击触发完整 LLM 回复（组装系统提示词，含性格 + 时间上下文）。后续点击回落到 Hitokoto 语录，10 分钟冷却后可再次触发 LLM。兼顾 API 成本与"首次接触"的仪式感。

## API 配置

毛玻璃设置面板，支持连接任意 OpenAI 兼容 API 端点：

- **Endpoint** -- 自动修正 URL：缺失 `/v1` 时自动补全，缺失 `/chat/completions` 时自动追加
- **API Key** -- 存储在 localStorage，密码输入框
- **Model** -- 可配置模型名（默认 `gpt-3.5-turbo`）

"保存"按钮会发送一个最小 POST 请求（`max_tokens: 1`）验证端点、Key 和模型的连通性。绿色表示成功，红色表示配置有误。

## GitHub 提交监控

关联 GitHub 账号后：

- 定时轮询 GitHub Events API
- 检测自上次检查以来的新 `PushEvent`
- 发现新提交时显示祝贺气泡
- "测试"按钮可先验证用户名是否存在

## 智能代办系统

霓虹主题面板，两种条目类型：

- **备忘** -- 永久文字记录，存于 localStorage，支持复制到剪贴板和标记完成
- **提醒** -- 倒计时定时器，时间到后触发循环提示音和视觉脉冲

自然语言时间解析：输入"30 分钟后提醒我站起来活动"自动识别。配置 API 后，模糊文本会回退到 LLM 解析。

## 音效系统

五种音效，基于 HTML5 Audio API：

| 音效 | 触发场景 |
|---|---|
| Pop | 点击宠物 / 生成粒子 |
| Boing | 唤醒跳跃 / 落地冲击 |
| Bubble | UI 交互 |
| Bell | 专注计时结束 |
| Crunch | 文件拖入 / 提醒闹钟 |

音频资源通过 Vite 的 `new URL()` 导入机制打包，确保开发和生产环境路径正确。

## SVG 去白边滤镜

自定义 SVG 滤镜链消除精灵图边缘的白色锯齿：

```
feMorphology (erode 1.5px) -> feGaussianBlur (0.5) -> feColorMatrix (alpha harden)
```

收缩可见 alpha 边界、模糊边缘、然后钳制 alpha 通道，在任何背景上产生干净的精灵图边缘。

## 眼睛跟踪

根据光标相对于窗口中心的位置水平翻转精灵图，产生宠物注视效果。应用在 `#pet-sprite`（而非容器）上，避免与物理挤压/拉伸变换冲突。

## 粒子效果

点击宠物生成 2-3 个随机 SVG 粒子（爱心、笑脸、星星、狗脸、眨眼、心形眼），上浮并淡出。所有粒子都是内联 SVG data URI，无外部图片，无 emoji。

## 物理反馈

交互状态的视觉反馈：

- **提起** -- `scaleY(1.05) scaleX(0.95)` 拉伸
- **落地** -- `scaleY(0.9) scaleX(1.1)` 挤压

平滑过渡：`cubic-bezier(0.25, 0.8, 0.25, 1)`。

## 音量控制

胶囊形 Windows 11 Fluent Design 音量面板：

- 毛玻璃背景 + 模糊效果
- 动态蓝色进度条
- 从宠物下方滑出
- 鼠标移开自动收起
- 音量通过 localStorage 持久化

## 文件拖入回收站

将文件拖放到宠物身上即可移入回收站。每次拖放播放"咀嚼"动画和音效。

## 右键菜单

原生 Tauri 菜单：

- 动画演示子菜单 -- 预览任意状态
- 自定义代办 -- 打开代办/提醒面板
- 专注计时器子菜单 -- 预设时长
- 音量控制
- 对话模式子菜单：基础闲聊(Hitokoto) / 觉醒模式(6种性格 + 自定义) / 接入 API / 关联 GitHub
- 设置子菜单：开机自启动 / 窗口置顶 / 导入宠物(.zip)
- 退出（播放"失败"动画后退出）

## 开机自启动

通过右键菜单设置启用。使用 Tauri autostart 插件注册系统启动项。

## 窗口置顶

通过右键菜单切换窗口 z 轴顺序。状态持久化到 localStorage，跨会话记忆。

## 退出告别

退出时播放"失败"动画 3 秒，期间锁定所有交互，然后优雅退出。

## 构建命令

```bash
npm run dev            # 仅 Vite 开发服务器（前端）
npm run tauri dev      # 完整 Tauri + Vite 开发（桌面应用）
npm run build          # TypeScript 检查 + Vite 生产构建
npm run tauri build    # 完整 Tauri 发布构建（生成安装包）
npx vite build         # 仅前端构建（跳过 tsc）
cargo check            # 检查 Rust 编译（在 src-tauri/ 下）
npx tauri icon <png>   # 从正方形 PNG 生成应用图标
```

## 构建产物

运行 `npm run tauri build` 后：

- **NSIS 安装包**: `src-tauri/target/release/bundle/nsis/`
- **可执行文件**: `src-tauri/target/release/vibe-pet.exe`

## 宠物包格式

`.zip` 文件，包含：

```
pet.json              -- 清单
spritesheet.webp      -- 精灵图
```

**pet.json schema:**

```json
{
  "id": "my-pet",
  "displayName": "My Pet",
  "description": "A custom pet",
  "spritesheetPath": "spritesheet.webp",
  "version": "1.0.0"
}
```

引擎使用默认回退：精灵图每行 = 一个状态，帧数假设相等。清单中无需状态/帧映射。

## Tauri 能力权限

`src-tauri/capabilities/default.json` 授予两个窗口 `core:default` + `core:window:allow-start-dragging`。

## 设计约束

- **无 UI 框架。** 所有渲染为纯 DOM + CSS。
- **无 Canvas/WebGL。** 精灵图动画仅使用 `background-position` + `steps()`。
- **粒子效果**必须是原生 DOM 元素 + CSS 动画，完成后通过 `.remove()` 移除。
- **pet.json schema**（`id`, `displayName`, `description`, `spritesheetPath`, `version`）-- 无状态/帧映射，代码必须有默认回退（行即状态假设）。

## 持久化

所有用户偏好（API 配置、对话模式、性格、音量、GitHub 用户名、窗口置顶、代办）通过 localStorage 跨会话持久化。

## 许可证

MIT
