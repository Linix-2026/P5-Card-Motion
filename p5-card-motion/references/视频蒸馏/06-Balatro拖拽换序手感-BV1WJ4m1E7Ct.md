# 06 · 重现 Balatro 游戏手感（拖拽换序 · 视觉分离 · 3D 旋转 · 虹彩 shader）

| 项 | 内容 |
|---|---|
| 来源 | B站 [BV1WJ4m1E7Ct](https://www.bilibili.com/video/BV1WJ4m1E7Ct)（搬运 Mix and Jam，原 YouTube I1dAZuWurw4） |
| 发布 | 2024-04-30；Unity 实现，配套源码 github.com/mixandjam/bolatro-feel |
| 字幕依据 | B站 AI 字幕（全文）+ 简介官方章节时间戳 |
| 归并目标 | **新模块 M13 · 拖拽换序手感**；同时为 M3/M5/M6/M7 各提供一条交叉验证与增强项 |

---

### 【设计意图】
Balatro 的手感建立在一个反直觉决策上：**逻辑与视觉是两个对象**。逻辑卡瞬移/换序都无所谓，视觉卡永远在「平滑追赶」逻辑卡——于是任何数据层操作（换位、归位、置顶）天然变成动画。其余细节全部建立在这条主干上：倾斜、阴影、3D 旋转、虹彩、相机噪声都是挂在视觉对象上的「抛光层」。

### 【效果拆解】
| 层 | 视觉/交互职能 | 驱动源 | 章节 |
|---|---|---|---|
| 插槽父子模型 | 每张卡永远有父插槽；拖拽结束 localPosition 归零即回位；换位＝**交换两张卡的父对象**并重置被拖卡之外的位置 | 事件+数据 | 00:38 基本逻辑 |
| 逻辑/视觉分离 | 逻辑卡处理交互（OnPointerEnter/Down/Up）；视觉预制件每帧**插值跟随**逻辑卡位置；旋转向运动方向微倾 | 时间（每帧 lerp） | 02:20 视觉分离 |
| DOTween 动画组 | 悬停摇晃（wobble）、选中位置打孔（punch）；拖拽时放大视觉并置顶 sortingOrder | 事件 | 03:47 轮换细节 |
| 偏移阴影 | 卡下挂一个 Y 偏移的阴影图，拖拽时强化「换了个视角」的抬升感 | 状态 | 03:47 |
| 闲置 3D 圆周旋转 | 额外子对象（不干扰父变换）：闲置时 x 轴 sin、y 轴 cos → 圆周路径缓转 | 时间 | 03:47 |
| 悬停视角旋转 | 鼠标距卡心距离 → 旋转值（与我们 M6 完全同构） | 鼠标 | 03:47 |
| 手牌弧度 | 动画曲线（AnimationCurve）决定位置与旋转偏移 | 数据 | 03:47 |
| 虹彩 shader | 旋转节点生成螺旋 + Hue 节点按 UV 偏移改色；卡的旋转值驱动偏移（牌动光带才动）；负片版反色+加强旋转 | 时间+状态 | 04:48 |
| 背景/相机/后处理 | 噪声+漩涡 UV 的迷幻背景 shader；Cinemachine 噪声微移；扫描线 CRT 后处理 | 时间 | 04:48 |

### 【机制原理】
1. **视觉/逻辑分离（最重要）**：`visual.pos = lerp(visual.pos, logic.pos, k)` 每帧执行。它把「动画」从命令式播放（给每个事件写补间）降级为**恒成立的跟随不变量**——数据怎么走，视觉自己追。这是与本项目「悬停进度唯一驱动源」并列的第二条架构级原则；
2. **父交换换序**：顺序变化即「把 A 的 slot 父对象换成 B 的」，位置由布局决定，无显式位移动画——跟随层自动补间；
3. **运动方向倾斜**：`tiltZ = clamp((pos.x - prevX) * k)`，给水平拖拽加纸张甩动感；
4. **闲置圆周**：两个正交相位的 sin/cos 驱动旋转子对象——等价于本项目 M4 的 sin 呼吸漂移的 3D 扩展；注意其**独立子对象**设计＝不污染布局用的父变换（对应我们 M6 在局部变换内施矩阵、布局层只存 cx/cy/rot 的纪律）；
5. **虹彩随旋转驱动**：螺旋 UV 的相位输入是卡的旋转值而非全局时间——光带响应「牌怎么动」，闲置时光影也活着。这是对 M7 驱动源设计的直接增强候选（M7 当前以鼠标/frameCount 为主，可追加 tilt 输入）。

### 【引擎注记】
Unity 实现：RectTransform 插槽父子、DOTween、AnimationCurve、Shader Graph 旋转/Hue 节点、Cinemachine noise（均仅参考）。

### 【p5 移植映射】
| 原方案组件 | p5.js 2.x 等价物 | 注意点 |
|---|---|---|
| 逻辑卡/视觉卡双对象 | `logic{x,y}` + 渲染状态 `vis{x,y}`，M1 damp 跟随 | 与 M2 状态字典合并：vis 即字典字段 |
| 交换父对象 | 交换数组元素后重算 M3/M11 布局 | 零动画代码 |
| DOTween wobble/punch | hoverProgress 上叠加 sin 衰减脉冲 | 脉冲短命、用后归零 |
| sortingOrder 置顶 | M4 两遍绘制（已实现） | 直接复用 |
| 偏移阴影 | M5 多层阴影，拖拽态加 lift | 直接复用 |
| 旋转驱动虹彩 | drawCardLaser 增加 tiltZ 相位输入 | **M7 增强项** |
| 动画曲线手牌 | M3 抛物线 / M9-spline 样条 | 见蒸馏 09 |
| Cinemachine 噪声 | 相机 translate 叠加小振幅 noise | 后处理扫描线＝半透明 scanline 贴图（谨慎，非核心） |

```js
// p5.js 2.x 骨架：逻辑/视觉分离 + 拖拽换序（≤40 行核心）
new p5((p) => {
  const W=120,H=168;
  const hand = [ {name:'A'},{name:'K'},{name:'Q'} ].map((c,i)=>({
    ...c, i, lx:0, ly:0, vx:0, vy:0, tiltZ:0, drag:false, punch:0 }));
  let dragIdx = -1;
  p.setup=()=>p.createCanvas(460,300);
  function layout(){ // 逻辑布局（M3 简化直线版，换 M3/M11 即得弧度/扇形）
    hand.forEach((c,i)=>{ c.lx = 120 + i*130; c.ly = 190; c.i = i; });
  }
  layout();
  p.draw=()=>{
    const dt=p.deltaTime||16.67; p.background(18,16,30);
    // ① 拖拽逻辑：指针更新逻辑坐标；越界即交换数组元素
    if(dragIdx>=0){ const c=hand[dragIdx]; c.lx=p.mouseX; c.ly=p.mouseY-30;
      for(let k=0;k<hand.length;k++){ if(k===dragIdx)continue;
        if((dragIdx<k && c.lx>hand[k].lx)||(dragIdx>k && c.lx<hand[k].lx)){
          [hand[dragIdx],hand[k]]=[hand[k],hand[dragIdx]]; dragIdx=k; layout(); break; } } }
    // ② 视觉恒跟随 + 方向倾斜
    for(const c of hand){
      const px=c.vx;
      c.vx=damp(c.vx,c.lx,c.drag?0.35:0.18,dt);
      c.vy=damp(c.vy,c.ly,c.drag?0.35:0.18,dt);
      c.tiltZ=p.constrain((c.vx-px)*0.012,-0.35,0.35);
      c.punch=damp(c.punch,0,0.12,dt);
      const sc=c.drag?1.1:1;
      p.push(); p.translate(c.vx,c.vy + (c.drag?-18:0) + c.punch);
      p.rotate(c.tiltZ); p.scale(sc);
      p.noStroke(); p.fill(0,90); p.rect(-W/2+4,-H/2+6,W,H,10);
      p.fill(201,168,76); p.rect(-W/2,-H/2,W,H,10);
      p.fill(30); p.textSize(34); p.textAlign(p.CENTER,p.CENTER); p.text(c.name,0,0);
      p.pop(); } // 两遍置顶在完整管线中实现，此处单遍仅演示跟随主干
  };
  p.mousePressed=()=>{ for(let i=hand.length-1;i>=0;i--){
      const c=hand[i]; if(p.abs(p.mouseX-c.vx)<W/2&&p.abs(p.mouseY-c.vy)<H/2){
        dragIdx=i; c.drag=true; c.punch=-14; return; } } };
  p.mouseReleased=()=>{ if(dragIdx>=0){ hand[dragIdx].drag=false; dragIdx=-1; layout(); } };
  function damp(c,t,k,dt){ const f=1-Math.pow(1-k,dt*60/1e3); return c+(t-c)*f; }
});
```

**参数表**：
| 参数 | 默认 | 调大/调小 |
|---|---|---|
| 跟随 k（静止/拖拽） | 0.18 / 0.35 | 大→灵敏；小→甩尾 |
| 方向倾斜系数 | 0.012（限幅 ±0.35rad） | 大→纸张感强；过大晕 |
| 拖拽抬升 | −18px + scale 1.1 | 与 M4 悬停 −50/1.18 区分（拖拽态更深） |
| punch 脉冲 | −14px，k 0.12 归零 | 大→选中「打孔」脆 |
| 闲置圆周 | sin/cos 振幅约 ±3°（字幕无数值，`未验证`） | — |

**时码锚点**（官方章节）：[00:38] 基本逻辑（插槽/父交换）｜[02:20] 视觉分离（插值跟随/方向倾斜）｜[03:47] 轮换细节（wobble/punch/置顶/偏移阴影/sin-cos 闲置旋转/鼠标距离旋转/手牌曲线）｜[04:48] 着色器与抛光（螺旋虹彩/负片/背景噪声/相机噪声/扫描线）｜[06:27] 最终结果。

**模块归并**：新模块 **M13**（拖拽换序：逻辑/视觉双轨 + 父交换 + 方向倾斜）。另登记三条增强：① M7 追加 tiltZ 驱动相位；② M6 闲置 sin/cos 圆周（从单轴呼吸扩为双轴圆周）；③ M5 拖拽态深度阴影。
