# wechat-minigame-bootstrap

> AI 编程助手 Skill — 一句话生成完整的微信小游戏项目骨架

## 这是什么？

这是一个通用的 AI 编程助手 Skill，安装后你只需要对 AI 说一句"帮我创建一个小游戏"，就能自动生成一套完整的微信小游戏 Canvas 2D 项目，包含：

- 游戏入口 + 游戏循环 + 屏幕管理器
- DPR 适配 + 安全区域处理
- Canvas 2D 绘图工具库（圆角矩形、渐变按钮、卡片、图标、进度条等）
- 补间动画系统
- 音频管理（BGM + SFX）
- 分享卡片生成
- 倒计时工具
- 后端 API 客户端（可选）
- 自建服务器代码（可选）

生成的代码全部使用 ES5 + CommonJS，无需构建工具，微信开发者工具直接打开就能跑。

## 适合的 AI 编程助手

任何支持自定义 Skill / Agent / 系统提示词的 AI 编程助手都能用：

- Claude Code / Trae / Qoder / CodeBuddy 等
- 各类 Claude 套壳/增强客户端
- 其他支持 Markdown 指令的 AI 编程工具（直接贴 SKILL.md 到提示词）

## 安装

```bash
# 克隆仓库
git clone https://github.com/binyuli/wechat-minigame-bootstrap-skill.git
# 或 Gitee（国内更快）
git clone https://gitee.com/binyuli/wechat-minigame-bootstrap-skill.git
```

把 `SKILL.md` 文件放入你所用工具的 Skill/Agent 目录即可：

- **Claude Code**: `~/.claude/skills/wechat-minigame-bootstrap/`
- **Trae / Qoder / CodeBuddy**: 放入对应工具的 Skill 目录
- **其他工具**: 把 SKILL.md 内容贴到系统提示词中

安装后对 AI 说"帮我创建一个微信小游戏"即可触发。

## 使用方法

### 1. 触发 Skill

对 AI 编程助手说：

```
帮我创建一个微信小游戏
```

### 2. 回答几个问题

Skill 会依次问你：

| 问题 | 说明 |
|------|------|
| 游戏名称 | 中文名 + 英文名，如"消消乐 / match-crush" |
| AppID | 在 mp.weixin.qq.com 获取，本地开发可用 `tourist` |
| 游戏类型 | 单机益智 / 多人 / 街机 / 棋盘 |
| 后端方案 | 无（纯本地）/ 自建 Node.js / 微信云开发 |

### 3. 获得完整项目

回答完毕后，Skill 自动生成所有文件：

```
your-game/
├── project.config.json    # 微信项目配置
├── game.json              # 游戏配置（横竖屏等）
├── game.js                # 入口：Canvas 初始化 + 游戏循环
├── .gitignore
└── js/
    ├── config.js          # 设计 Token（颜色、渐变、阴影、字体）
    ├── cloud-client.js    # 后端 API 客户端
    ├── ui/
    │   ├── draw.js        # 绘图工具统一导出
    │   ├── draw-base.js   # 基础图形（圆角矩形、碰撞检测、清屏）
    │   ├── draw-card.js   # 卡片组件（渐变 + 阴影）
    │   ├── draw-icon.js   # 矢量图标（箭头、奖杯、计时器等）
    │   ├── draw-style.js  # 样式组件（按钮、文字、进度条、星星）
    │   └── animation.js   # 补间动画 + 缓动函数
    ├── screens/
    │   ├── HomeScreen.js  # 首页（标题 + 开始按钮）
    │   └── GameScreen.js  # 游戏画面（占位，等你写逻辑）
    └── utils/
        ├── audio.js       # 背景音乐 + 音效
        ├── shareImage.js  # 动态分享卡片
        └── timer.js       # 倒计时器
```

### 4. 在微信开发者工具中打开

1. 打开微信开发者工具
2. 导入项目 → 选择生成的目录
3. 填入 AppID
4. 立刻就能看到首页，点击"开始游戏"跳转到游戏画面

### 5. 开始写你的游戏逻辑

只需编辑 `js/screens/GameScreen.js`，在 `init` 中初始化状态，在 `update` 中更新逻辑，在 `render` 中绘制画面。

## 内置绘图 API 速查

所有绘图工具通过 `GameGlobal.G.draw` 访问：

```javascript
var draw = GameGlobal.G.draw;
var cfg = GameGlobal.G.config;

// 清屏
draw.clear(ctx, W, H, cfg.colors.bg);

// 圆角矩形
draw.roundRect(ctx, x, y, w, h, 10, '#FFFFFF');

// 渐变按钮（自动处理阴影）
var bounds = draw.gradientButton(ctx, x, y, 200, 50, '点击', {
  gradient: cfg.gradients.primary,
  textColor: '#FFFFFF',
  radius: 25
});

// 居中文字
draw.centerText(ctx, 'Hello', W/2, H/2, 'bold 24px sans-serif', '#333');

// 碰撞检测
if (draw.hit(touchX, touchY, bounds.x, bounds.y, bounds.w, bounds.h)) {
  // 被点击了
}
```

## 核心设计模式

- **全局状态 `GameGlobal.G`** — config、draw、屏幕管理器全局可访问
- **屏幕生命周期** — `constructor → init → update/render → destroy`
- **按钮在 render 中创建 bounds，在 touch 中 hit test** — 位置永远和渲染同步
- **DPR 缩放只处理一次** — init 中 scale(dpr, dpr)，后续全用逻辑像素
- **安全区域** — UI 元素从 `G.safeArea.top` 以下开始绘制

## 成功案例

**凑24** — 基于 Skill 模板架构开发的 24 点数学卡牌游戏，已上线微信小游戏平台。支持关卡模式、实时对战（WebSocket）、排行榜。

![凑24小程序码](docs/qr-24game.jpg)

微信搜索"凑24"或扫码体验。

## 环境要求

- 任意支持 Skill/Agent 的 AI 编程助手
- [微信开发者工具](https://developers.weixin.qq.com/miniprogram/dev/devtools/download.html)
- 微信小游戏 AppID（或使用 `tourist` 模式本地开发）

## 仓库地址

https://github.com/binyuli/wechat-minigame-bootstrap-skill

国内用户推荐通过 Gitee 或 Codeup 克隆，速度更快。

## License

MIT
