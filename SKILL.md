# WeChat Mini Game Bootstrap

Scaffold a complete WeChat mini game (Canvas 2D) project from scratch. Generates all boilerplate files so you can start building gameplay immediately.

## When to Use

Trigger when the user:
- Wants to create a new WeChat mini game project
- Says "new game", "create game", "init game", "scaffold game"
- Mentions starting a WeChat mini game from zero

**Do NOT trigger** for WeChat Mini Program (小程序 / WXML/WXSS) — this skill is for Canvas-based mini games (小游戏) only.

## Prerequisites

Before scaffolding, ask the user:

1. **Game name** (Chinese + English, e.g. "24点游戏" / "24-game")
2. **AppID** — from [mp.weixin.qq.com](https://mp.weixin.qq.com) backend. If not ready, use `"tourist"` for local dev.
3. **Game type** — single-player puzzle / multiplayer / arcade / board / other (affects template selection)
4. **Backend choice** — self-hosted (Node.js + MySQL) / WeChat cloud / none (local only)
5. **Project directory** — where to create the project (default: current directory)

## File Structure

Generate this structure:

```
project.config.json
game.json
game.js                  # Entry point: canvas init, game loop, screen manager, touch handlers
.gitignore
js/
├── config.js            # Design tokens (colors, gradients, shadows, fonts)
├── cloud-client.js      # Backend API client (HTTP + WebSocket)
├── ui/
│   ├── draw.js          # Drawing primitives (re-exports from sub-modules)
│   ├── draw-base.js     # roundRect, strokeRoundRect, gradientRoundRect, hit, clear, overlay
│   ├── draw-card.js     # shadowCard, gradientCard
│   ├── draw-icon.js     # drawIcon (arrow-left, gamepad, trophy, timer, etc.)
│   ├── draw-style.js    # gradientButton, text, centerText, sectionTitle, progressBar, divider
│   └── animation.js     # Tween + easing (linear, easeInOut, easeOut, easeIn)
├── screens/
│   ├── HomeScreen.js    # Landing page with game title + start button
│   └── GameScreen.js    # Main gameplay screen (placeholder)
└── utils/
    ├── audio.js         # BGM + SFX management (wx.createInnerAudioContext)
    ├── shareImage.js    # Dynamic share card generation (offscreen canvas)
    └── timer.js         # Countdown timer utility
```

If **self-hosted backend** chosen, also generate:

```
server/
├── package.json
├── .env.example
├── index.js             # Express + ws + mysql2 server
├── db.js                # MySQL connection pool
├── routes/
│   └── game.js          # Game API routes (score, rank, room)
├── ws-handlers/
│   └── battle.js        # WebSocket battle logic
└── migrate.js           # Data migration script
```

## Template Files

### project.config.json

```json
{
  "description": "{{GAME_NAME_CN}}",
  "packOptions": {
    "ignore": [
      { "type": "folder", "value": "server" },
      { "type": "folder", "value": "docs" },
      { "type": "folder", "value": "tests" },
      { "type": "folder", "value": ".claude" },
      { "type": "file", "value": "README.md" },
      { "type": "file", "value": ".gitignore" },
      { "type": "file", "value": "project.config.json.example" }
    ],
    "include": []
  },
  "setting": {
    "urlCheck": false,
    "es6": true,
    "enhance": true,
    "compileHotReLoad": true,
    "postcss": false,
    "compileWorklet": false,
    "minified": false,
    "uglifyFileName": false,
    "uploadWithSourceMap": true,
    "packNpmManually": false,
    "packNpmRelationList": [],
    "useCompilerPlugins": false
  },
  "compileType": "game",
  "libVersion": "3.3.4",
  "appid": "{{APPID}}",
  "projectname": "{{PROJECT_NAME}}",
  "condition": {},
  "editorSetting": {}
}
```

### game.json

```json
{
  "deviceOrientation": "portrait",
  "showStatusBar": true,
  "networkTimeout": {
    "request": 5000,
    "connectSocket": 5000
  }
}
```

### game.js

```javascript
/**
 * {{GAME_NAME_CN}} - Main Entry Point
 */

var config = require('./js/config');
var draw = require('./js/ui/draw');

var G = {
  sm: null,
  config: config,
  draw: draw,
  canvas: null,
  ctx: null,
  W: 0,
  H: 0,
  safeArea: { top: 0, bottom: 0, left: 0, right: 0 }
};

module.exports = G;
GameGlobal.G = G;

function init() {
  var canvas = wx.createCanvas();
  var ctx = canvas.getContext('2d');
  var info = wx.getSystemInfoSync();
  var dpr = info.pixelRatio;

  G.canvas = canvas;
  G.ctx = ctx;
  G.dpr = dpr;
  G.W = info.windowWidth;
  G.H = info.windowHeight;

  // Safe area: notch + WeChat capsule button
  G.safeArea = { top: 0, bottom: 0, left: 0, right: 0 };
  if (info.safeArea) {
    G.safeArea.top = info.safeArea.top || 0;
    G.safeArea.bottom = G.H - (info.safeArea.bottom || G.H);
  }
  try {
    var menuBtn = wx.getMenuButtonBoundingClientRect();
    G.safeArea.top = Math.max(G.safeArea.top, menuBtn.bottom + 6);
  } catch (e) {
    if (G.safeArea.top < 44) G.safeArea.top = 44;
  }

  G.config.W = G.W;
  G.config.H = G.H;

  canvas.width = G.W * dpr;
  canvas.height = G.H * dpr;
  ctx.scale(dpr, dpr);

  // Screen manager
  G.sm = {
    current: null,
    stack: [],
    switch: function(name, params) {
      if (this.current && this.current.destroy) this.current.destroy();
      var Screen = require('./js/screens/' + name);
      this.current = new Screen();
      this.current.init(params || {});
    },
    push: function(name, params) {
      if (this.current) this.stack.push(this.current);
      var Screen = require('./js/screens/' + name);
      this.current = new Screen();
      this.current.init(params || {});
    },
    pop: function() {
      if (this.current && this.current.destroy) this.current.destroy();
      this.current = this.stack.pop() || null;
    }
  };

  // Touch handlers
  wx.onTouchStart(function(e) {
    if (G.sm.current && G.sm.current.onTouchStart) {
      var t = e.touches[0];
      G.sm.current.onTouchStart(t.clientX, t.clientY);
    }
  });
  wx.onTouchEnd(function(e) {
    if (G.sm.current && G.sm.current.onTouchEnd) {
      var t = e.changedTouches[0];
      G.sm.current.onTouchEnd(t.clientX, t.clientY);
    }
  });
  wx.onTouchMove(function(e) {
    if (G.sm.current && G.sm.current.onTouchMove) {
      var t = e.touches[0];
      G.sm.current.onTouchMove(t.clientX, t.clientY);
    }
  });

  // Share config
  wx.onShareAppMessage(function() {
    return {
      title: '来玩{{GAME_NAME_CN}}吧！',
      imageUrl: ''
    };
  });
  wx.showShareMenu({ withShareTicket: true, menus: ['shareAppMessage', 'shareTimeline'] });

  // Launch screen
  G.sm.switch('HomeScreen');

  // Game loop
  var lastTime = Date.now();
  function loop() {
    var now = Date.now();
    var dt = (now - lastTime) / 1000;
    lastTime = now;
    if (G.sm.current) {
      G.sm.current.update(dt);
      G.sm.current.render(ctx);
    }
    requestAnimationFrame(loop);
  }
  requestAnimationFrame(loop);
}

init();
```

### js/config.js

```javascript
module.exports = {
  W: 375,
  H: 667,
  apiBase: 'https://YOUR_DOMAIN',
  wsBase: 'wss://YOUR_DOMAIN',
  colors: {
    primary: '#6C5CE7',
    primaryDark: '#5A4BD1',
    primaryLight: '#E8E5FF',
    success: '#00B894',
    danger: '#FF6B6B',
    warning: '#FDCB6E',
    bg: '#F8F5FF',
    cardBg: '#FFFFFF',
    text: '#2D3436',
    textSecondary: '#636E72',
    textLight: '#B2BEC3',
    white: '#FFFFFF',
    border: '#DFE6E9',
    overlay: 'rgba(0,0,0,0.5)',
    gold: '#FFD700',
    sakura: '#FD79A8',
    sky: '#74B9FF',
    mint: '#55EFC4',
    coral: '#FF7675',
    lavender: '#A29BFE',
    lemon: '#FFEAA7'
  },
  gradients: {
    primary: ['#6C5CE7', '#5A4BD1'],
    success: ['#55EFC4', '#00B894'],
    danger: ['#FF6B6B', '#EE5A5A'],
    orange: ['#E17055', '#D35E45']
  },
  shadows: {
    sm: { color: 'rgba(108,92,231,0.08)', blur: 4, offsetY: 2 },
    md: { color: 'rgba(108,92,231,0.12)', blur: 8, offsetY: 4 },
    lg: { color: 'rgba(108,92,231,0.18)', blur: 16, offsetY: 8 }
  },
  radius: { sm: 6, md: 10, lg: 16, xl: 24, pill: 50 },
  fonts: {
    title: 'bold 28px sans-serif',
    subtitle: 'bold 20px sans-serif',
    body: '16px sans-serif',
    small: '13px sans-serif',
    card: 'bold 32px sans-serif'
  },
  spacing: { xs: 4, sm: 8, md: 16, lg: 24, xl: 32 },
  timing: { fast: 150, normal: 300, slow: 500 }
};
```

### js/screens/HomeScreen.js

```javascript
function HomeScreen() {
  this.buttons = [];
  this._pressedBtn = null;
  this._time = 0;
}

HomeScreen.prototype.init = function(params) {};

HomeScreen.prototype.update = function(dt) {
  this._time += dt;
};

HomeScreen.prototype.render = function(ctx) {
  var G = GameGlobal.G;
  var draw = G.draw;
  var cfg = G.config;
  var W = G.W;
  var H = G.H;
  var pad = cfg.spacing.md;

  draw.clear(ctx, W, H, cfg.colors.bg);

  // Title
  draw.centerText(ctx, '{{GAME_NAME_CN}}', W / 2, H * 0.3, cfg.fonts.title, cfg.colors.primary);

  // Start button
  var btnW = 200, btnH = 50;
  var btnX = (W - btnW) / 2;
  var btnY = H * 0.55;
  var isPressed = this._pressedBtn === 'start';
  var bounds = draw.gradientButton(ctx, btnX, btnY, btnW, btnH, '开始游戏', {
    gradient: cfg.gradients.primary,
    textColor: cfg.colors.white,
    fontSize: 18,
    radius: cfg.radius.pill,
    pressed: isPressed
  });
  this.buttons = [{ x: bounds.x, y: bounds.y, w: bounds.w, h: bounds.h, action: 'start' }];
};

HomeScreen.prototype.onTouchStart = function(x, y) {
  for (var i = 0; i < this.buttons.length; i++) {
    var b = this.buttons[i];
    if (GameGlobal.G.draw.hit(x, y, b.x, b.y, b.w, b.h)) {
      this._pressedBtn = b.action;
      return;
    }
  }
};

HomeScreen.prototype.onTouchEnd = function(x, y) {
  if (this._pressedBtn === 'start') {
    GameGlobal.G.sm.switch('GameScreen');
  }
  this._pressedBtn = null;
};

HomeScreen.prototype.onTouchMove = function() {};

HomeScreen.prototype.destroy = function() {};

module.exports = HomeScreen;
```

### js/screens/GameScreen.js

```javascript
function GameScreen() {
  this.buttons = [];
  this._pressedBtn = null;
  this._time = 0;
}

GameScreen.prototype.init = function(params) {
  // TODO: initialize game state
};

GameScreen.prototype.update = function(dt) {
  this._time += dt;
};

GameScreen.prototype.render = function(ctx) {
  var G = GameGlobal.G;
  var draw = G.draw;
  var cfg = G.config;
  var W = G.W;
  var H = G.H;

  draw.clear(ctx, W, H, cfg.colors.bg);

  // Back button
  var btnSize = 28;
  var padX = cfg.spacing.md;
  var backY = Math.max(4, G.safeArea.top - 36);
  var backBounds = draw.gradientButton(ctx, padX, backY, btnSize, btnSize, '', {
    gradient: ['rgba(108,92,231,0.06)', 'rgba(108,92,231,0.06)'],
    textColor: cfg.colors.primary,
    radius: 8,
    icon: 'arrow-left',
    iconSize: 14,
    shadow: false
  });

  // Placeholder text
  draw.centerText(ctx, '游戏区域', W / 2, H / 2, cfg.fonts.subtitle, cfg.colors.textSecondary);

  this.buttons = [
    { x: backBounds.x, y: backBounds.y, w: backBounds.w, h: backBounds.h, action: 'back' }
  ];
};

GameScreen.prototype.onTouchStart = function(x, y) {
  for (var i = 0; i < this.buttons.length; i++) {
    var b = this.buttons[i];
    if (GameGlobal.G.draw.hit(x, y, b.x, b.y, b.w, b.h)) {
      this._pressedBtn = b.action;
      return;
    }
  }
};

GameScreen.prototype.onTouchEnd = function(x, y) {
  if (this._pressedBtn === 'back') {
    GameGlobal.G.sm.switch('HomeScreen');
  }
  this._pressedBtn = null;
};

GameScreen.prototype.onTouchMove = function() {};

GameScreen.prototype.destroy = function() {};

module.exports = GameScreen;
```

### js/ui/draw.js

```javascript
var base = require('./draw-base');
var card = require('./draw-card');
var icon = require('./draw-icon');
var style = require('./draw-style');

module.exports = {
  // Base shapes
  roundRect: base.roundRect,
  strokeRoundRect: base.strokeRoundRect,
  gradientRoundRect: base.gradientRoundRect,
  hit: base.hit,
  clear: base.clear,
  overlay: base.overlay,
  divider: base.divider,
  applyShadow: base.applyShadow,
  resetShadow: base.resetShadow,
  // Cards
  shadowCard: card.shadowCard,
  // Icons
  drawIcon: icon.drawIcon,
  // Style components
  gradientButton: style.gradientButton,
  text: style.text,
  centerText: style.centerText,
  gradientText: style.gradientText,
  sectionTitle: style.sectionTitle,
  progressBar: style.progressBar,
  drawStar: style.drawStar,
  drawTrophy: style.drawTrophy,
  drawMedal: style.drawMedal
};
```

### js/ui/draw-base.js

```javascript
function roundRect(ctx, x, y, w, h, r, fill) {
  r = Math.min(r, w / 2, h / 2);
  ctx.beginPath();
  ctx.moveTo(x + r, y);
  ctx.lineTo(x + w - r, y);
  ctx.arcTo(x + w, y, x + w, y + r, r);
  ctx.lineTo(x + w, y + h - r);
  ctx.arcTo(x + w, y + h, x + w - r, y + h, r);
  ctx.lineTo(x + r, y + h);
  ctx.arcTo(x, y + h, x, y + h - r, r);
  ctx.lineTo(x, y + r);
  ctx.arcTo(x, y, x + r, y, r);
  ctx.closePath();
  if (fill) { ctx.fillStyle = fill; ctx.fill(); }
}

function strokeRoundRect(ctx, x, y, w, h, r, stroke, lw) {
  r = Math.min(r, w / 2, h / 2);
  ctx.beginPath();
  ctx.moveTo(x + r, y);
  ctx.lineTo(x + w - r, y);
  ctx.arcTo(x + w, y, x + w, y + r, r);
  ctx.lineTo(x + w, y + h - r);
  ctx.arcTo(x + w, y + h, x + w - r, y + h, r);
  ctx.lineTo(x + r, y + h);
  ctx.arcTo(x, y + h, x, y + h - r, r);
  ctx.lineTo(x, y + r);
  ctx.arcTo(x, y, x + r, y, r);
  ctx.closePath();
  ctx.strokeStyle = stroke;
  ctx.lineWidth = lw || 1;
  ctx.stroke();
}

function gradientRoundRect(ctx, x, y, w, h, r, c1, c2) {
  r = Math.min(r, w / 2, h / 2);
  var grad = ctx.createLinearGradient(x, y, x, y + h);
  grad.addColorStop(0, c1);
  grad.addColorStop(1, c2);
  roundRect(ctx, x, y, w, h, r, null);
  ctx.fillStyle = grad;
  ctx.fill();
}

function hit(px, py, rx, ry, rw, rh) {
  return px >= rx && px <= rx + rw && py >= ry && py <= ry + rh;
}

function clear(ctx, W, H, color) {
  var dpr = GameGlobal.G.dpr || 1;
  ctx.setTransform(1, 0, 0, 1, 0, 0);
  ctx.fillStyle = color || '#F8F5FF';
  ctx.fillRect(0, 0, W * dpr, H * dpr);
  ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
}

function overlay(ctx, W, H, alpha) {
  ctx.fillStyle = 'rgba(0,0,0,' + (alpha || 0.5) + ')';
  ctx.fillRect(0, 0, W, H);
}

function divider(ctx, x, y, w, color) {
  ctx.beginPath();
  ctx.moveTo(x, y);
  ctx.lineTo(x + w, y);
  ctx.strokeStyle = color || '#DFE6E9';
  ctx.lineWidth = 1;
  ctx.stroke();
}

function applyShadow(ctx, opts) {
  ctx.shadowColor = opts.color || 'rgba(0,0,0,0.1)';
  ctx.shadowBlur = opts.blur || 8;
  ctx.shadowOffsetX = 0;
  ctx.shadowOffsetY = opts.offsetY || 4;
}

function resetShadow(ctx) {
  ctx.shadowColor = 'transparent';
  ctx.shadowBlur = 0;
  ctx.shadowOffsetX = 0;
  ctx.shadowOffsetY = 0;
}

module.exports = {
  roundRect: roundRect,
  strokeRoundRect: strokeRoundRect,
  gradientRoundRect: gradientRoundRect,
  hit: hit,
  clear: clear,
  overlay: overlay,
  divider: divider,
  applyShadow: applyShadow,
  resetShadow: resetShadow
};
```

### js/ui/draw-card.js

```javascript
var base = require('./draw-base');

var CARD_PALETTES = [
  ['#74B9FF', '#0984E3'],  // blue
  ['#55EFC4', '#00B894'],  // mint
  ['#A29BFE', '#6C5CE7'],  // purple
  ['#FFEAA7', '#E17055']   // orange
];

function shadowCard(ctx, x, y, w, h, value, opts) {
  opts = opts || {};
  var cfg = GameGlobal.G.config;
  var r = opts.radius || 14;
  var idx = (opts.cardIndex || 0) % CARD_PALETTES.length;
  var palette = CARD_PALETTES[idx];

  // Shadow layers
  if (!opts.disabled) {
    base.applyShadow(ctx, cfg.shadows.md);
  }
  base.roundRect(ctx, x, y, w, h, r, '#FFFFFF');
  base.resetShadow(ctx);

  // Gradient fill
  if (opts.disabled) {
    base.gradientRoundRect(ctx, x + 2, y + 2, w - 4, h - 4, r - 1, '#DFE6E9', '#B2BEC3');
  } else if (opts.selected) {
    base.gradientRoundRect(ctx, x + 2, y + 2, w - 4, h - 4, r - 1, '#FFEAA7', '#FD79A8');
    base.strokeRoundRect(ctx, x + 1, y + 1, w - 2, h - 2, r, '#FFD700', 2.5);
  } else {
    base.gradientRoundRect(ctx, x + 2, y + 2, w - 4, h - 4, r - 1, palette[0], palette[1]);
  }

  // Value text
  var textColor = opts.disabled ? '#B2BEC3' : (opts.textColor || '#FFFFFF');
  var fontSize = opts.fontSize || 32;
  ctx.fillStyle = textColor;
  ctx.font = 'bold ' + fontSize + 'px sans-serif';
  ctx.textAlign = 'center';
  ctx.textBaseline = 'middle';
  ctx.fillText(String(value), x + w / 2, y + h / 2);
}

module.exports = { shadowCard: shadowCard };
```

### js/ui/draw-icon.js

```javascript
function drawIcon(ctx, name, cx, cy, size, color) {
  var s = (size || 24) / 24;
  ctx.save();
  ctx.translate(cx, cy);
  ctx.scale(s, s);
  ctx.strokeStyle = color || '#6C5CE7';
  ctx.fillStyle = color || '#6C5CE7';
  ctx.lineWidth = 2;
  ctx.lineCap = 'round';
  ctx.lineJoin = 'round';

  switch (name) {
    case 'arrow-left':
      ctx.beginPath();
      ctx.moveTo(-8, 0); ctx.lineTo(8, 0);
      ctx.moveTo(-2, -6); ctx.lineTo(-8, 0); ctx.lineTo(-2, 6);
      ctx.stroke();
      break;
    case 'gamepad':
      ctx.beginPath();
      ctx.arc(-6, 2, 4, 0, Math.PI * 2);
      ctx.arc(6, 2, 4, 0, Math.PI * 2);
      ctx.stroke();
      ctx.fillRect(-3, -6, 6, 3);
      break;
    case 'trophy':
      ctx.beginPath();
      ctx.moveTo(-6, -6); ctx.lineTo(6, -6);
      ctx.lineTo(4, 2); ctx.lineTo(-4, 2); ctx.closePath();
      ctx.stroke();
      ctx.beginPath(); ctx.moveTo(-4, 2); ctx.lineTo(-6, 6); ctx.lineTo(6, 6); ctx.lineTo(4, 2); ctx.stroke();
      ctx.beginPath(); ctx.moveTo(0, -6); ctx.lineTo(0, 2); ctx.stroke();
      break;
    case 'timer':
      ctx.beginPath();
      ctx.arc(0, 2, 8, 0, Math.PI * 2); ctx.stroke();
      ctx.beginPath(); ctx.moveTo(0, 2); ctx.lineTo(0, -4); ctx.stroke();
      ctx.beginPath(); ctx.moveTo(0, -6); ctx.lineTo(0, -10); ctx.stroke();
      break;
    default:
      // Unknown icon — draw a dot
      ctx.beginPath(); ctx.arc(0, 0, 3, 0, Math.PI * 2); ctx.fill();
  }
  ctx.restore();
}

module.exports = { drawIcon: drawIcon };
```

### js/ui/draw-style.js

```javascript
var base = require('./draw-base');

function gradientButton(ctx, x, y, w, h, label, opts) {
  opts = opts || {};
  var cfg = GameGlobal.G.config;
  var r = opts.radius || 12;

  if (opts.disabled) {
    base.roundRect(ctx, x, y, w, h, r, '#DFE6E9');
  } else {
    var grad = opts.gradient || cfg.gradients.primary;
    var gradObj = ctx.createLinearGradient(x, y, x, y + h);
    gradObj.addColorStop(0, grad[0]);
    gradObj.addColorStop(1, grad[1]);
    if (opts.shadow !== false) base.applyShadow(ctx, cfg.shadows.sm);
    base.roundRect(ctx, x, y, w, h, r, null);
    ctx.fillStyle = gradObj;
    ctx.fill();
    base.resetShadow(ctx);
  }

  if (opts.icon) {
    var icon = require('./draw-icon');
    icon.drawIcon(ctx, opts.icon, x + 14, y + h / 2, opts.iconSize || 14, opts.textColor || '#FFFFFF');
  }

  if (label) {
    ctx.fillStyle = opts.disabled ? '#B2BEC3' : (opts.textColor || '#FFFFFF');
    ctx.font = (opts.fontSize || 16) + 'px sans-serif';
    ctx.textAlign = opts.icon ? 'left' : 'center';
    ctx.textBaseline = 'middle';
    var textX = opts.icon ? x + 28 : x + w / 2;
    ctx.fillText(label, textX, y + h / 2);
  }

  return { x: x, y: y, w: w, h: h };
}

function text(ctx, str, x, y, opts) {
  opts = opts || {};
  ctx.fillStyle = opts.color || '#2D3436';
  ctx.font = opts.font || '16px sans-serif';
  ctx.textAlign = opts.align || 'left';
  ctx.textBaseline = opts.baseline || 'top';
  if (opts.maxWidth) {
    var measured = ctx.measureText(str).width;
    if (measured > opts.maxWidth) {
      while (str.length > 0 && ctx.measureText(str + '..').width > opts.maxWidth) str = str.slice(0, -1);
      str += '..';
    }
  }
  ctx.fillText(str, x, y);
}

function centerText(ctx, str, cx, cy, font, color) {
  ctx.fillStyle = color || '#2D3436';
  ctx.font = font || '16px sans-serif';
  ctx.textAlign = 'center';
  ctx.textBaseline = 'middle';
  ctx.fillText(str, cx, cy);
}

function gradientText(ctx, str, x, y, c1, c2, font, opts) {
  opts = opts || {};
  ctx.font = font || 'bold 28px sans-serif';
  ctx.textAlign = opts.align || 'center';
  ctx.textBaseline = opts.baseline || 'middle';
  var w = ctx.measureText(str).width;
  var grad = ctx.createLinearGradient(x - w / 2, y, x + w / 2, y);
  grad.addColorStop(0, c1);
  grad.addColorStop(1, c2);
  ctx.fillStyle = grad;
  ctx.fillText(str, x, y);
}

function progressBar(ctx, x, y, w, h, progress, opts) {
  opts = opts || {};
  var r = opts.radius || h / 2;
  base.roundRect(ctx, x, y, w, h, r, opts.bgColor || '#E0E3E8');
  var fillW = Math.max(h, w * Math.max(0, Math.min(1, progress)));
  var grad = ctx.createLinearGradient(x, y, x + fillW, y);
  var colors = (progress < (opts.warningThreshold || 0.15))
    ? (opts.warningGradient || ['#E74C3C', '#C0392B'])
    : (opts.gradient || ['#5B9BD5', '#2E75B6']);
  grad.addColorStop(0, colors[0]);
  grad.addColorStop(1, colors[1]);
  base.roundRect(ctx, x, y, fillW, h, r, null);
  ctx.fillStyle = grad;
  ctx.fill();
}

function drawStar(ctx, cx, cy, outerR, innerR, color) {
  ctx.beginPath();
  for (var i = 0; i < 5; i++) {
    var angle = -Math.PI / 2 + (i * 2 * Math.PI / 5);
    ctx.lineTo(cx + Math.cos(angle) * outerR, cy + Math.sin(angle) * outerR);
    angle += Math.PI / 5;
    ctx.lineTo(cx + Math.cos(angle) * innerR, cy + Math.sin(angle) * innerR);
  }
  ctx.closePath();
  ctx.fillStyle = color || '#FFD700';
  ctx.fill();
}

function drawTrophy(ctx, cx, cy, size, color) {
  require('./draw-icon').drawIcon(ctx, 'trophy', cx, cy, size || 24, color || '#FFD700');
}

function drawMedal(ctx, cx, cy, r, rank) {
  var colors = { 1: '#FFD700', 2: '#C0C0C0', 3: '#CD7F32' };
  ctx.beginPath();
  ctx.arc(cx, cy, r, 0, Math.PI * 2);
  ctx.fillStyle = colors[rank] || '#DFE6E9';
  ctx.fill();
  ctx.fillStyle = '#FFFFFF';
  ctx.font = 'bold ' + (r * 1.1) + 'px sans-serif';
  ctx.textAlign = 'center';
  ctx.textBaseline = 'middle';
  ctx.fillText(String(rank), cx, cy);
}

function sectionTitle(ctx, str, x, y, opts) {
  opts = opts || {};
  text(ctx, str, x, y, { color: opts.color || '#2D3436', font: opts.font || 'bold 18px sans-serif' });
  var w = ctx.measureText(str).width;
  ctx.beginPath();
  ctx.moveTo(x, y + 22);
  ctx.lineTo(x + w, y + 22);
  ctx.strokeStyle = opts.accentColor || '#6C5CE7';
  ctx.lineWidth = 2;
  ctx.stroke();
}

module.exports = {
  gradientButton: gradientButton,
  text: text,
  centerText: centerText,
  gradientText: gradientText,
  sectionTitle: sectionTitle,
  progressBar: progressBar,
  drawStar: drawStar,
  drawTrophy: drawTrophy,
  drawMedal: drawMedal
};
```

### js/ui/animation.js

```javascript
var Easing = {
  linear: function(t) { return t; },
  easeInOut: function(t) { return t < 0.5 ? 4*t*t*t : 1 - Math.pow(-2*t+2, 3) / 2; },
  easeOut: function(t) { return 1 - Math.pow(1 - t, 3); },
  easeIn: function(t) { return t * t * t; }
};

function createTween(opts) {
  var duration = opts.duration || 0.3;
  var easingFn = Easing[opts.easing || 'easeOut'] || Easing.easeOut;
  var elapsed = 0;
  var from = opts.from || {};
  var to = opts.to || {};
  var keys = Object.keys(from);

  return {
    update: function(dt) {
      elapsed += dt;
      var t = Math.min(elapsed / duration, 1);
      var easedT = easingFn(t);
      var values = {};
      for (var i = 0; i < keys.length; i++) {
        var k = keys[i];
        values[k] = from[k] + (to[k] - from[k]) * easedT;
      }
      if (opts.onUpdate) opts.onUpdate(values);
      if (t >= 1 && opts.onComplete) opts.onComplete();
    },
    isDone: function() { return elapsed >= duration; }
  };
}

module.exports = { create: createTween, Easing: Easing };
```

### js/utils/audio.js

```javascript
var _bgm = null;
var _bgmSrc = '';
var _bgmPlaying = false;
var _sfxPool = {};

function bgmPlay(src) {
  if (_bgmSrc === src && _bgmPlaying) return;
  bgmStop();
  _bgm = wx.createInnerAudioContext();
  _bgm.src = src;
  _bgm.loop = true;
  _bgm.volume = 0.5;
  _bgm.play();
  _bgmSrc = src;
  _bgmPlaying = true;
}

function bgmStop() {
  if (_bgm) { _bgm.stop(); _bgm.destroy(); _bgm = null; }
  _bgmPlaying = false;
  _bgmSrc = '';
}

function sfx(name) {
  var audio = wx.createInnerAudioContext();
  audio.src = 'audio/' + name + '.mp3';
  audio.volume = 0.8;
  audio.play();
  audio.onEnded(function() { audio.destroy(); });
}

module.exports = { bgmPlay: bgmPlay, bgmStop: bgmStop, sfx: sfx };
```

### js/utils/timer.js

```javascript
function CountdownTimer(duration, onTick, onEnd) {
  this.remaining = duration;
  this.duration = duration;
  this._onTick = onTick;
  this._onEnd = onEnd;
  this._paused = false;
}

CountdownTimer.prototype.update = function(dt) {
  if (this._paused || this.remaining <= 0) return;
  this.remaining = Math.max(0, this.remaining - dt);
  if (this._onTick) this._onTick(this.remaining);
  if (this.remaining <= 0 && this._onEnd) this._onEnd();
};

CountdownTimer.prototype.pause = function() { this._paused = true; };
CountdownTimer.prototype.resume = function() { this._paused = false; };
CountdownTimer.prototype.progress = function() { return 1 - this.remaining / this.duration; };

module.exports = CountdownTimer;
```

### js/cloud-client.js

```javascript
var config = require('./config');

var API = config.apiBase;

function request(method, path, data) {
  return new Promise(function(resolve, reject) {
    wx.request({
      url: API + path,
      method: method,
      data: data,
      header: { 'content-type': 'application/json' },
      success: function(res) {
        if (res.statusCode >= 200 && res.statusCode < 300) {
          resolve(res.data);
        } else {
          reject(new Error('HTTP ' + res.statusCode));
        }
      },
      fail: function(err) { reject(err); }
    });
  });
}

function login() {
  return new Promise(function(resolve) {
    wx.login({
      success: function(res) {
        if (res.code) {
          request('POST', '/api/login', { code: res.code }).then(resolve).catch(function() {
            resolve({ code: -1 });
          });
        } else {
          resolve({ code: -1 });
        }
      },
      fail: function() { resolve({ code: -1 }); }
    });
  });
}

module.exports = {
  login: login,
  get: function(path) { return request('GET', path); },
  post: function(path, data) { return request('POST', path, data); }
};
```

### .gitignore

```
node_modules/
.project.config.json
*.map
.env
.DS_Store
```

## Post-Scaffold Checklist

After generating all files, remind the user of next steps:

1. **Open in WeChat DevTools** — import the project directory, fill in AppID
2. **Test run** — should see HomeScreen with "开始游戏" button
3. **Create your game logic** — edit `GameScreen.js`, add screens as needed
4. **Add audio files** — put BGM/SFX in `audio/` directory
5. **Configure backend** — edit `js/config.js` apiBase/wsBase if using self-hosted
6. **Register game** — complete filing (备案) at mp.weixin.qq.com before launch

## Key Patterns to Follow

- **Global state via `GameGlobal.G`** — all modules read config/draw from G, no circular requires
- **Screen lifecycle**: `constructor → init → update/render loop → destroy`
- **Button bounds in render, hit test in touch** — never hardcode positions in both
- **`draw.resetShadow(ctx)`** after every shadow to avoid state leaks
- **DPR scaling handled once in init** — all coordinates use logical pixels
- **Safe area**: always start UI below `G.safeArea.top`
