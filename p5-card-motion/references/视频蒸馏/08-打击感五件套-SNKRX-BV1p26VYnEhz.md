# 08 · 代码级「打击感」五件套（读 SNKRX 源码）

| 项 | 内容 |
|---|---|
| 来源 | B站 [BV1p26VYnEhz](https://www.bilibili.com/video/BV1p26VYnEhz)《如何写代码实现「打击感」？【读游戏源码】〈小丑牌〉和〈SNKRX〉第 1 期》 |
| 发布 | 2024-12-30；SNKRX（Lua，MIT 协议，Steam 好评；被指为 Balatro 灵感来源之一） |
| 字幕依据 | B站 AI 字幕（全文）；读源码讲解，机制描述具体 |
| 归并目标 | **新模块 M14 · 打击感套件**（震屏/弹簧/闪烁/粒子环/慢动作）；与 M5/M13 联动 |

---

### 【设计意图】
打击感不是一个特效，而是一组**在命中瞬间同时发生、各自独立可调**的小系统。低频高强度行为（英雄死亡、关键命中）才配调用全套——克制调用决定了它的分量。每个效果都读得懂源码、抄得走。

### 【效果拆解】（字幕明示的五件）
| # | 效果 | 源码机制（字幕原述） | 驱动源 |
|---|---|---|---|
| 1 | 摄像机抖动 shake | 物体绘制前后 `camera.attach/detach`：attach 保存当前坐标系并按 camera.x/y 建新坐标系，detach 恢复；shake 函数在一段时间内不断改 camera.x/y | 事件+时间 |
| 2 | 弹簧 spring shake/缩放 | 同理改坐标系；`spring.pull` 一段时间内改变 spring.x，用它改坐标系或直接改所绘 object 的 scale | 事件+时间 |
| 3 | 闪烁 flash | 调 `flasher.flash()` 使其 `f` 在一段时间内为 true；绘制时检测 `f` 换色 | 事件+时间 |
| 4 | 粒子 hit circle | 命中处红色粒子（代码较简，视频跳过细节） | 事件（一次爆发） |
| 5 | 慢动作 | 全局 `slow()` 控制 `slow_amount`，把 `slow_amount` 乘进 group 更新的 `dt` → 该组变慢 1/slow_amount 倍 | 事件+全局时间 |

### 【机制原理】
1. **震屏＝变换栈偏移而非真动相机**：所有物体绘制包在一个 translate(shakeX, shakeY) 里——映射 p5 即场景绘制外层一次 `translate`（对应 08 文档项目里的 CinemaCamera 层）；偏移量在持续时间内衰减回 0（弹簧阻尼或指数衰减）；
2. **spring 与 shake 的区别**：shake 是带随机种子的衰减噪声（unstructured），spring 是过冲回弹（目标方向上的一次 pull → 振荡归零），spring.x 还能驱动 scale 造成「被揍压缩」；
3. **flash＝布尔闸门 + 计时**：渲染期 `if(flasher.f) fill(闪色)`，时间到自动 false——与 M12 呼吸灯共享「时间相位驱动呈现」范式；
4. **慢动作＝全局时间缩放乘 dt**（价值最高的一条）：`effDt = dt * timeScale`，所有移动/补间/动画只吃 effDt。**这与 08 文档硬约束「deltaTime 驱动」完美咬合**：正因为动效全部 dt 驱动，一个全局乘数即可实现子弹时间，无需改任何模块；恢复时 timeScale 用 damp 缓回 1（避免硬切）。

### 【引擎注记】
SNKRX Lua：自绘引擎的 camera group/attach 机制、全局 slow 函数（参考用）；p5 等价为变换栈 + dt 乘数。

### 【p5 移植映射】
| 原方案组件 | p5.js 2.x 等价物 | 注意点 |
|---|---|---|
| camera.attach/detach | 场景绘制 `push()+translate(shakeX,shakeY) … pop()` | UI/手牌可不入栈（世界震、UI 稳） |
| shake 衰减 | 触发时存 `{t,dur,amp,seed}`，渲染期算衰减随机偏移 | 结束即清零，禁残余漂移 |
| spring.pull | `springV += impulse; springX += springV*dt; springV -= springX*stiff*dt*…` | 或复用 damp 加过冲近似 |
| flasher.f | `hitFlash=1`，每帧 `hitFlash=damp(…,0)`，绘制 `if>0` 染色 | 可用 ADD 白闪，必复位 |
| hit circle 粒子 | 对象池小粒子（M7 硬顶纪律） | ≤24 粒/次 |
| slow_amount | `timeScale`（damp 回 1）× 进所有 dt | **只动 dt 一处** |

```js
// p5.js 2.x 骨架：打击感套件（≤40 行），挂在场景根部
function makeJuice(p){
  return {
    timeScale: 1, shake:0, shakeT:0, flash:0,
    shake(amp=10, dur=260){ this.shake=amp; this.shakeT=dur; },
    hit(){ this.flash=1; },
    slow(scale=0.35){ this.timeScale=scale; },          // 慢动作入口
    update(dtMs){
      this.timeScale = damp(this.timeScale, 1, 0.06, dtMs); // 缓回正常速
      this.flash = damp(this.flash, 0, 0.18, dtMs);
      if(this.shakeT>0){ this.shakeT-=dtMs; if(this.shakeT<=0) this.shake=0; }
    },
    offset(){ return this.shakeT>0
      ? { x:p.random(-this.shake,this.shake)*(this.shakeT/260),
          y:p.random(-this.shake,this.shake)*(this.shakeT/260) } : {x:0,y:0}; },
    beginWorld(){ const o=this.offset(); p.push(); p.translate(o.x,o.y);
      if(this.flash>0.02){ /* 命中物体自身染色更准；此处给全局一闪的可选项 */ } },
    endWorld(){ p.pop(); },
    effDt(){ return (p.deltaTime||16.67) * this.timeScale; }, // 所有动效改吃它
  };
}
function damp(c,t,k,dt){ const f=1-Math.pow(1-k,dt*60/1e3); return c+(t-c)*f; }
// 命中瞬间：juice.shake(12); juice.hit(); 粒子环爆发； 重大死亡：juice.slow(0.35)
```

**参数表**：
| 参数 | 普通命中 | 重大事件 | 调大/调小 |
|---|---|---|---|
| shake 幅度/时长 | 6–10px / 180–260ms | 16px / 420ms | 大→重；频繁触发必须小 |
| flash 衰减 k | 0.18 | 0.1（更慢退） | — |
| timeScale | 1（不触发） | 0.3–0.4，恢复 k 0.06 | 过小+过长＝打断节奏 |
| 粒子 | ≤24/次 | ≤48/次 | 对象池硬顶 |

**时码锚点**：shake 段（camera.attach/detach）→ spring 段 → flash 段（flasher.f）→ hit circle 段（略讲）→ 慢动作段（slow_amount×dt，结尾）。精确秒数 `未验证`。

**模块归并**：新模块 **M14**。两条架构级要求：① 项目已全量 dt 驱动（08 硬约束），慢动作只加一个乘数；② 效果建议由 07 的效果事件总线触发（如 damage.perform 内 `ctx.fx.hit()`），逻辑层不引用渲染模块。
