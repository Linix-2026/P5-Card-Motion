# 01 · 三步增加游戏按钮反馈（UI Juice 套件）

| 项 | 内容 |
|---|---|
| 来源 | B站 [BV1w4416iEv2](https://www.bilibili.com/video/BV1w4416iEv2) |
| 标题/发布 | 《三步增加你的游戏按钮反馈》· 2026-08-30 |
| 作者 | 独立开发者（AI 字幕，UP 名 MCP 未返回） |
| 引擎 | 视频未强调引擎（像素风 UI，技法引擎无关） |
| 字幕依据 | B站 AI 字幕（全文） |
| 归并目标 | **新模块 M9 · 交互元素 Juice 套件**；与 M1 缓动内核、M4 悬停管线同源 |

---

### 【设计意图】
让「触碰按钮」成为有生命确认的瞬间：悬停＝邀请（变色+Q 弹形变+微倾斜），按下＝下沉（物理压缩），松开＝回弹（能量释放），点击＝扩散光圈（反馈余韵），进场＝错峰（层级秩序），音效随机音高＝避免机械乏味。核心是 **juice 三层递进：色彩提示 → 形变反馈 → 事件余韵**，每层独立可裁剪。

### 【效果拆解】
| 层 | 视觉/听觉职能 | 驱动源 | 参数与成本 |
|---|---|---|---|
| 像素皮肤三层边框 | 外边框/背板/阴影三层结构；hover 时外边框与文字变色 | 事件（enter/exit） | 3 个 rect，零成本 |
| Squash & Stretch | hover：Y 压缩、X 膨胀（Q 弹）；按下：外边框+背板下沉，松开回弹，可叠加缩放制造立体感 | 事件+时间（补间） | 1 组 scale，damp 驱动 |
| 随机倾斜入场 | 进场时随机角度再回正，活泼感 | 事件（onAppear，每元素掷一次） | 复用 M2「掷签一次」纪律 |
| 点击光圈 | 从按钮向外扩散的圆环 | 事件（click，短命粒子） | 单个 ellipse + alpha 衰减，对象池可省 |
| 随机音高音效 | enter/exit/click 三事件音效，pitch 随机扰动 | 事件 | p5.sound `play()` 前设 `rate()` |
| 错峰入场 | 多按钮先后逐个出现 | 事件（场景切换）+ 索引延迟 | 入场延迟 = i × stagger |

### 【机制原理】
- **Squash & Stretch（动画十二原则）**：体积守恒的错觉——Y 压缩时 X 必须等比膨胀（scaleY↓ 则 scaleX↑），不是整体缩放；
- **三态补间**：hover（1→Q 弹目标）/ press（追加下沉偏移）/ release（弹簧回归）——与 08 文档 M4「状态进度唯一驱动源」同构：用 `hoverProgress/pressProgress` 两个进度派生全部形变；
- **错峰 stagger**：第 i 个元素的入场起始时刻 = `i * staggerSec`，制造层级；
- **随机音高**：每次播放前对采样 `rate = 1 ± rand(0.05)`，同一音效不复现为机械节拍。

### 【引擎注记】
视频为像素游戏 UI 实操，未展示引擎补间库；做法等价于通用 UI Tween（参考用，不移植）。

### 【p5 移植映射】
| 原方案组件 | p5.js 2.x 等价物 | 注意点 |
|---|---|---|
| 补间三态 | M1 `dampApproach` + 两个进度字段 | k∈0.1–0.3（硬约束 #15） |
| 随机倾斜 | `rand()` 入场掷签一次 | 禁每帧重掷（M2 教训） |
| 光圈扩散 | ellipse + 进度驱动 radius/alpha | ADD 后必复位 BLEND |
| 随机音高 | p5.sound `Oscillator/Envelope` 或采样 `.rate(1±r)` | 首次交互手势内启动 AudioContext |
| 错峰入场 | `appearT = max(0, t - i*stagger)` 再喂缓动 | — |

```js
// p5.js 2.x 实例模式骨架（≤40 行）：按钮三态 + 光圈 + 错峰
new p5((p) => {
  const btn = { x:200, y:160, w:150, h:52, hov:0, press:0, appear:0, ring:0, tilt:0, seed:0 };
  let sfx;
  p.setup = () => { p.createCanvas(400, 320); p.textFont('sans-serif');
    btn.seed = p.random(-0.12, 0.12);           // 入场倾斜只掷一次
    // sfx = p.loadSound('tick.wav');            // 实际项目加载短采样
  };
  p.draw = () => {
    p.background(16, 16, 26);
    const dt = p.deltaTime || 16.67;
    const over = p.mouseX > btn.x-btn.w/2 && p.mouseX < btn.x+btn.w/2 &&
                 p.mouseY > btn.y-btn.h/2 && p.mouseY < btn.y+btn.h/2;
    btn.hov   = damp(btn.hov,   over ? 1 : 0, 0.25, dt);
    btn.press = damp(btn.press, (over && p.mouseIsPressed) ? 1 : 0, 0.3, dt);
    btn.appear = Math.min(1, btn.appear + dt/320);             // 单元素即错峰 i=0
    const e = 1 - Math.pow(1 - btn.appear, 3);
    btn.tilt = btn.seed * (1 - e);
    btn.ring = over && btn.ring <= 0 ? btn.ring : Math.max(0, btn.ring - dt/420);
    const q = btn.hov, sx = 1 + 0.06*q, sy = 1 - 0.06*q;        // 体积守恒错觉
    p.push(); p.translate(btn.x, btn.y + 8*btn.press + 26*(1-e));
    p.rotate(btn.tilt); p.scale(sx, sy);
    p.noStroke(); p.fill(0, 90); p.rect(-btn.w/2+3, -btn.h/2+3, btn.w, btn.h, 8); // 阴影层
    p.fill(201,168,76, 120+100*q); p.rect(-btn.w/2, -btn.h/2, btn.w, btn.h, 8);  // 背板层
    p.noFill(); p.stroke(240,208,96); p.strokeWeight(2+q);                        // 外边框层
    p.rect(-btn.w/2, -btn.h/2, btn.w, btn.h, 8);
    p.fill(255); p.noStroke(); p.textAlign(p.CENTER,p.CENTER); p.textSize(18); p.text('开始游戏', 0, 1);
    p.pop();
    if (btn.ring > 0){ p.push(); p.blendMode(p.ADD); p.noFill();
      p.stroke(240,208,96, 200*btn.ring); p.strokeWeight(3);
      p.ellipse(btn.x, btn.y, btn.w + (1-btn.ring)*90);       // 点击光圈
      p.pop(); p.blendMode(p.BLEND); }
  };
  p.mousePressed = () => { btn.ring = 1; /* if(sfx) sfx.rate(1+p.random(-.06,.06)), sfx.play(); */ };
  function damp(c,t,k,dt){ const f=1-Math.pow(1-k,dt*60/1e3); return c+(t-c)*f; }
});
```

**参数表**：
| 参数 | 默认 | 调大/调小 |
|---|---|---|
| squash 量 | 0.06 | 大→卡通 Q 弹；小→精致克制 |
| hov damp k | 0.25 | 大→跟手生硬；小→绵 |
| 按下下沉 | 8px | 大→物理实；小→轻触 |
| 光圈寿命 | 420ms / 扩散 90px | 大→余韵长；过大拖泥带水 |
| 音高扰动 | ±6% | 大→活泼；过大会走音 |
| 入场错峰 | 每元素 60–90ms | 大→仪式感；大过 150ms 迟钝 |

**时码锚点**：AI 字幕无逐秒时间轴；关键节拍依字幕顺序＝三层边框 → hover 变色与 squash → 随机倾斜 → 按下下沉/回弹 → 点击光圈 → 音效 → 错峰入场（具体秒数 `未验证`）。

**模块归并**：新模块 **M9**。与 M4 共享进度驱动范式；正式入册时与 M8 卡面绘制解耦（M9 面向任意可交互矩形，不依赖卡牌数据契约）。
