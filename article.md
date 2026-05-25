# 一句话生成微信小游戏：我给 Claude Code 写了个 Skill

> 本文记录了如何用 Claude Code Skill 在 30 秒内生成一套完整的微信小游戏项目骨架，以及我用它做了一个消消乐的全过程。

## 背景

做微信小游戏（Canvas 2D），每次新项目都要写一遍重复的样板代码：

- Canvas 初始化 + DPR 适配
- 游戏循环 + requestAnimationFrame
- 屏幕管理器（首页 → 游戏页 → 结果页）
- 触摸事件处理
- 安全区域适配（刘海屏 + 微信胶囊按钮）
- 绘图工具（圆角矩形、渐变按钮、卡片……）
- 音频管理、分享卡片、倒计时器……

这些代码跟具体游戏逻辑无关，但每次都要写，大概 500-800 行。

于是我给 Claude Code 写了一个 Skill，把这些样板代码模板化。现在只需要一句话，就能生成完整的项目骨架，直接开始写游戏逻辑。

## 安装

```bash
git clone https://github.com/binyuli/wechat-minigame-bootstrap-skill.git ~/.claude/skills/wechat-minigame-bootstrap
```

安装后在 Claude Code 中输入 `/wechat-minigame-bootstrap` 即可触发。

## 30 秒生成项目

在 Claude Code 中输入：

```
/wechat-minigame-bootstrap 帮我创建一个小游戏
```

Skill 会问你 4 个问题：

1. **游戏名称** — 中文名 + 英文名
2. **AppID** — 微信后台获取，本地开发可用 `tourist`
3. **游戏类型** — 单机 / 多人 / 街机 / 棋盘
4. **后端方案** — 无 / 自建 Node.js / 微信云开发

回答后，自动生成完整项目：

```
my-game/
├── project.config.json
├── game.json
├── game.js                 # 入口：Canvas + 游戏循环 + 屏幕管理 + 触摸
└── js/
    ├── config.js           # 设计 Token
    ├── ui/
    │   ├── draw-base.js    # 基础图形
    │   ├── draw-card.js    # 卡片组件
    │   ├── draw-icon.js    # 矢量图标
    │   ├── draw-style.js   # 按钮、文字、进度条
    │   └── animation.js    # 补间动画
    ├── screens/
    │   ├── HomeScreen.js   # 首页
    │   └── GameScreen.js   # 游戏画面
    └── utils/
        ├── audio.js        # 音频管理
        ├── shareImage.js   # 分享卡片
        └── timer.js        # 倒计时
```

用微信开发者工具打开，立刻就能看到首页 + "开始游戏"按钮，点击跳转到游戏画面。

## 实战：做一个消消乐

骨架生成完后，我让 Claude Code 在 `GameScreen.js` 里实现了消消乐的核心玩法：

- 8x8 棋盘，6 种宝石（不同颜色 + 几何图形标识）
- 点选宝石 → 点相邻宝石交换
- 三个及以上同色消除
- 自动下落填充 + 连锁消除
- 计分系统（连击加倍）

整个过程，我只需要说：

> 实现消消乐游戏核心玩法

Claude Code 就在 `GameScreen.js` 里写了完整的匹配消除逻辑、动画系统、触摸交互。大概 400 行代码，涵盖了状态机（IDLE → SWAP → ELIMINATE → FALL → 循环）、匹配检测、重力填充、无解重排。

生成的代码用微信开发者工具打开就能玩，没有任何依赖，不需要 npm install。

## 生成的项目有什么特点？

**零依赖**：纯 ES5 + CommonJS，不依赖任何 npm 包，微信开发者工具直接打开。

**DPR 适配**：自动处理 Retina 屏的高清渲染，`clear()` 函数正确处理了 DPR 缩放恢复。

**安全区域**：自动检测刘海屏 + 微信胶囊按钮位置，所有 UI 从安全区域以下开始绘制。

**绘图工具库**：内置 20+ 绘图函数，支持渐变按钮、阴影卡片、矢量图标、进度条、星星/奖牌等，调用方式统一。

```javascript
// 几行代码画一个渐变按钮
var bounds = draw.gradientButton(ctx, x, y, 200, 50, '开始游戏', {
  gradient: cfg.gradients.primary,
  textColor: '#FFFFFF',
  radius: 25
});

// 碰撞检测
if (draw.hit(touchX, touchY, bounds.x, bounds.y, bounds.w, bounds.h)) {
  // 被点击了
}
```

**屏幕管理**：`switch / push / pop` 三种导航模式，每个 Screen 有完整的生命周期 `init → update/render → destroy`。

## 设计 Token 系统

所有视觉参数集中在 `js/config.js` 中管理：

```javascript
config.colors      // 20+ 颜色变量
config.gradients   // 预设渐变
config.shadows     // 三级阴影 (sm/md/lg)
config.radius      // 圆角尺寸
config.fonts       // 字体预设
config.spacing     // 间距系统
```

改一套颜色就能换主题，不用改任何业务代码。

## 适合做什么类型的游戏？

适合所有 Canvas 2D 小游戏：

- 棋盘类：消消乐、连连看、2048、扫雷
- 卡牌类：24点、记忆翻牌
- 休闲类：弹球、打砖块、接金币
- 益智类：数独、华容道

不适合需要复杂物理引擎或 3D 渲染的游戏。

## 后续开发建议

项目骨架生成后，开发流程：

1. 编辑 `GameScreen.js`，写你的游戏逻辑
2. 需要新页面时，在 `js/screens/` 下新建文件
3. 音效文件放到 `audio/` 目录，用 `require('../utils/audio').sfx('click')` 播放
4. 需要后端时，编辑 `config.js` 的 `apiBase`，启用 `cloud-client.js`

## 总结

这个 Skill 的核心价值是**消除重复劳动**。微信小游戏的样板代码是固定的，每次手写浪费时间，用模板生成后只需要关注游戏逻辑本身。

项目地址：[github.com/binyuli/wechat-minigame-bootstrap-skill](https://github.com/binyuli/wechat-minigame-bootstrap-skill)

欢迎 Star、Fork、提 Issue。
