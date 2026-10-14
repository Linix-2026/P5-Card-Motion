# 09 · 样条曲线弧形手牌（t∈[0,1] 均匀采样 + 切线定向）

| 项 | 内容 |
|---|---|
| 来源 | B站 [BV1CoKPzHEyV](https://www.bilibili.com/video/BV1CoKPzHEyV)（搬运 The Code Otter，原 YouTube hmIS2iBe-iQ） |
| 发布 | 2025-06-20；Unity Splines 包 + DOTween |
| 字幕依据 | B站 AI 字幕（全文，采样公式讲解具体） |
| 归并目标 | **M3 弧面布局的参数化升级版（M3-spline 候选 / M15）**；与 M11 扇牌互补 |

---

### 【设计意图】
手牌曲线不该写死成抛物线公式——把「手牌形状」变成一条**可在编辑器里调的曲线资源**，卡牌沿曲线均匀分布、朝向沿曲线切线，策划不碰代码即可打磨弧度手感。对 p5 的价值：把 M3 的硬编码抛物线升级为「采样函数」抽象，形状可切换（抛物线/贝塞尔/Catmull-Rom/样条），布局与动画代码不动。

### 【效果拆解】
| 层 | 机制（字幕原述要点） | 驱动源 |
|---|---|---|
| 曲线 | 3 节点曲线，中点改 Bezier 拖手柄微调；结点 up 向量须统一朝远离相机方向（否则朝向计算错误） | 静态配置 |
| 参数化 | 曲线上位置 p∈[0,1]，0=起点 1=终点，0.5=中点 | — |
| 居中均布 | `spacing = 1/maxCount`；首张位置 `0.5 - (count-1)*spacing/2`，第 i 张 `p = first + i*spacing` | 数据（卡数） |
| 位置采样 | 曲线 API 由 p 取世界坐标 Vector3 | — |
| 朝向 | 取该点 forward（曲线方向）与 up，**叉积得第三向量**作卡牌向上方向，Quaternion.LookRotation 定旋 | 曲线切线 |
| 到位动画 | DOTween `DOMove(target, dur)` + `DORotateQuaternion(target, dur)` | 事件（抽牌/换序） |

### 【机制原理】（引擎无关）
- **居中均布公式**与本项目 M3 的 `centerOffset` 完全同构：`p_i = 0.5 + (i-(n-1)/2)/maxN`（视频用 maxCount 做 spacing，允许手牌未满时整体仍居中——M3 用的是当前 n 即时均分；两种口径，满手时等价，未满时视频版卡间距恒定、M3 版间距随卡数变化，**这是一个值得显式选择的设计差异**）；
- **沿路径定向**：2D 下等价极简——曲线在 p 处切线角 `rot = atan2(P(p+ε).y-P(p-ε).y, P(p+ε).x-P(p-ε).x)`，卡牌 rotate 该角（再按卡面朝向偏 90°）；不需要叉积三维那一套；
- **路径抽象**：布局函数只依赖 `samplePath(p) → {x,y}`，形状由具体路径函数决定——抛物线、三次贝塞尔、Catmull-Rom 都实现同一接口即可互换。

### 【引擎注记】
Unity Splines 包（SplineContainer/Evaluate forward/up）+ DOTween（仅参考）；2D 用少量数学函数等价，无第三方依赖。

### 【p5 移植映射】
| 原方案组件 | p5.js 2.x 等价物 | 注意点 |
|---|---|---|
| SplineContainer 资源 | `paths = { arc, bezier, catmull }` 采样函数表 | 每个函数签名 `(t)=>{x,y}` |
| Evaluate 位置/forward/up | `samplePath(t)` + 邻近点差分求切线角 | ε≈0.002 |
| Quaternion.LookRotation | 2D `rotate(tangentAngle)` | 卡面法线朝屏，无需 3D |
| DOTween DOMove/Rotate | M1 dampApproach 逐帧逼近采样结果 | 重排/抽牌自动平滑 |

```js
// p5.js 2.x 骨架：路径抽象 + 均布 + 切线定向（≤34 行）
const HandPaths = {
  // 抛物线（等价 M3）：t∈[0,1]，拱高 arcH，两端下沉
  arc(t, w, baseY, arcH){ const u=t*2-1; return { x:(t-0.5)*w, y:baseY - arcH*(1-u*u) }; },
  // 三次贝塞尔：控制点决定形状，编辑器手感的 2D 版
  bezier(t, P0,P1,P2,P3){ const it=1-t;
    const x=it**3*P0.x+3*it**2*t*P1.x+3*it*t**2*P2.x+t**3*P3.x;
    const y=it**3*P0.y+3*it**2*t*P1.y+3*it*t**2*P2.y+t**3*P3.y; return {x,y}; },
};
function tangentAngle(path, t, args){
  const e=0.002, a=path(Math.max(0,t-e),...args), b=path(Math.min(1,t+e),...args);
  return Math.atan2(b.y-a.y, b.x-a.x);
}
function splineLayout(i, n, maxN, path, args, constantSpacing){
  // constantSpacing=true：视频口径（间距按 maxN 固定，未满手也居中）
  const denom = constantSpacing ? maxN : n;
  const spacing = 1 / denom;
  const p = 0.5 - (n - 1) * spacing / 2 + i * spacing;
  const pos = path(p, ...args);
  return { x:pos.x, y:pos.y, rot: tangentAngle(path, p, args) * 0.35 }; // 转角弱化系数 0.35
}
// 动画：把输出作为 vis 目标值喂 dampApproach（与 M13 跟随层、M2 状态字典无缝衔接）
```

**参数表**：
| 参数 | 默认 | 调大/调小 |
|---|---|---|
| 间距口径 | 视频=按 maxN 恒定 / M3=按当前 n | 手感二选一（见机制段） |
| 拱高 arcH | 26–40px @720p | 大→扇形捧持；小→平直 |
| 转角弱化 | 切线角 ×0.35 | 1=完全贴切线（两端卡过斜）；0=不转 |
| 到位时长 | 220–320ms（damp k 0.18 近似） | 大→从容；小→利落 |

**时码锚点**：spacing=1/max 与首张位置公式段（中前段，字幕「零点五减去卡片数量减一乘以卡片间距除以二」）；叉积定向段；DOTween 移动段（结尾演示）。精确秒数 `未验证`。

**模块归并**：**M3-spline（M15 候选）**。不替换 M3，而是为 M3 增加 `pathFn` 注入点：现有抛物线即 `HandPaths.arc`，新增贝塞尔/Catmull-Rom 零改动画层；同时把「恒定间距 vs 即时均分」提为显式配置。与 M11（绕枢轴扇开）构成两种手牌拓扑。
