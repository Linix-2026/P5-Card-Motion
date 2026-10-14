# 02 · 视差 Banner 的自然手感（标准化指针输入 + 多层 lerp）

| 项 | 内容 |
|---|---|
| 来源 | B站 [BV1t69gB5EkZ](https://www.bilibili.com/video/BV1t69gB5EkZ) |
| 标题/发布 | 《B站首页这种视差 Banner，为什么你做出来总是不自然？》· 2026-04-04 |
| 作者 | 前端向教学 UP（MCP 未返回 UP 名） |
| 引擎 | Web 前端（pointermove / requestAnimationFrame / lerp，技法与 p5 同构） |
| 字幕依据 | ⚠️ **无字幕**（SUBTITLE_UNAVAILABLE），依据视频简介明示的技术点蒸馏；参数级细节标 `未验证` |
| 归并目标 | **新模块 M10 · 指针视差层**；直接增强 M6（3D 倾斜的输入归一化思路同源） |

---

### 【设计意图】
视差的「不自然」根源不是幅度，而是**输入空间错了**：直接用视口坐标会让 Banner 在页面不同位置、不同容器尺寸下表现不一致，且指针瞬移到元素上时图层瞬间跳位。目标是让多层元素像被一根看不见的弹簧牵引，对指针做出「以容器中心为基准、远近分层、平滑跟随」的响应。

### 【效果拆解】
| 层 | 视觉职能 | 驱动源 | 参数与成本 |
|---|---|---|---|
| 输入换算 | pointermove 视口坐标 → 容器内部坐标 → 以中心为基准的 [-1,1] 标准化值 | 鼠标（每事件） | 两次减法+除法，零成本 |
| 多层位移 | 每层配一个 depth 系数，标准化输入 × depth = 该层位移 | 鼠标 | 数组配置，每层 1 次 translate |
| lerp 平滑 | 当前位移逐帧 lerp 向目标位移，消除瞬移 | 时间（rAF 循环） | 即 08 文档 dampApproach 的定帧前身 |

### 【机制原理】
- **中心基准归一化**（简介明示）：`nx = (clientX - rect.left - rect.width/2) / (rect.width/2)` → [-1,1]。分母是**容器半宽**而非全屏宽——同一指针角度在任何布局位置产生相同响应，这是「位置一致性」的关键；
- **多层 depth 配置**：`layer.dx = nx * layer.depthPx`，远景层 depth 小（微动）、近景层 depth 大；可选 Z 轴旋转 `rz = -ny * depth`；
- **rAF + lerp**：位移目标在事件回调里更新，实际位移在 rAF 里 `cur += (target-cur)*k` 逼近——事件频率（可变、可能突发）与渲染频率解耦。数学等价于 08 文档 **M1 dampApproach 定 k 版**（帧率严谨场景应直接换 dampApproach）；
- **出界回中**：指针离开容器时目标归零，图层优雅回正（与展厅 pointerInside 处理同源纪律）。

### 【引擎注记】
原视频为 DOM/Canvas 前端实现（pointermove + requestAnimationFrame，简介明示）；不涉及引擎专属 API，无移植损耗。

### 【p5 移植映射】
| 原方案组件 | p5.js 2.x 等价物 | 注意点 |
|---|---|---|
| getBoundingClientRect 归一化 | 以容器（canvas 或卡片包围盒）中心/半宽除算 | p5 内嵌画布时 rect 即画布尺寸 |
| rAF 循环 | `p.draw`（自带 rAF） | 无需手写 requestAnimationFrame |
| lerp 跟随 | **直接用 M1 dampApproach**（帧率无关） | 比定系数 lerp 更严谨 |
| 多层容器 | `layers=[{depthPx,rotK,draw()}]` 配置数组 | depth 为「布局量」，随画布缩放换算 |

```js
// p5.js 2.x 实例模式骨架：容器中心化指针视差（≤30 行）
new p5((p) => {
  // depth 越大越「近」；近景跟得多，远景跟得少
  const layers = [
    { depth: 6,  col: [60,40,90]  },   // 远景
    { depth: 16, col: [90,60,130] },   // 中景
    { depth: 30, col: [201,168,76] },  // 近景
  ];
  layers.forEach(l => { l.x = 0; l.y = 0; });
  let tx = 0, ty = 0, inside = false;
  p.setup = () => p.createCanvas(400, 240);
  p.draw = () => {
    const dt = p.deltaTime || 16.67;
    inside = p.mouseX >= 0 && p.mouseX <= p.width && p.mouseY >= 0 && p.mouseY <= p.height;
    if (inside) { // 以容器中心为基准、半宽归一化到 [-1,1]
      tx = (p.mouseX - p.width/2)  / (p.width/2);
      ty = (p.mouseY - p.height/2) / (p.height/2);
    } else { tx = 0; ty = 0; }
    p.background(12, 12, 20);
    for (const l of layers) {
      l.x = damp(l.x, (inside ? tx : 0) * l.depth, 0.12, dt); // 远景 k 可更小更沉
      l.y = damp(l.y, (inside ? ty : 0) * l.depth, 0.12, dt);
      p.push(); p.translate(p.width/2 + l.x, p.height/2 + l.y);
      p.noStroke(); p.fill(l.col[0], l.col[1], l.col[2], 180);
      p.ellipse(0, 0, 90, 90); p.pop();
    }
  };
  function damp(c,t,k,dt){ const f=1-Math.pow(1-k,dt*60/1e3); return c+(t-c)*f; }
});
```

**参数表**：
| 参数 | 参考默认 | 调大/调小 |
|---|---|---|
| 各层 depth | 6/16/30 px（@400 宽） | 层间差越大空间越深；过大穿帮 |
| 跟随 damp k | 0.12 | 大→跟手轻盈；小→沉重迟滞（远景宜更小） |
| 归一化基准 | 容器半宽/半高 | 误用全屏宽＝跨位置不一致（本视频核心坑） |

> 以上数值为本转译按 400px 画布给的参考实现，**非视频实测值（未验证）**；视频原始参数与图层数量未公开。

**时码锚点**：无字幕无法锚定；简介列出的讲解顺序＝视口坐标问题 → 容器中心换算 → 标准化输入/多层/rAF/lerp。

**模块归并**：新模块 **M10**。其输入归一化函数可被 M6 `computeCardTilt` 复用（目前 M6 用卡面半宽归一化，已是同原理的局部版本）；正式入册时抽取为公共 `normalizePointer(clientX, clientY, box)`。
