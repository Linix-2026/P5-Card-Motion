# 07 · 模块化卡牌效果系统（组合优于硬编码）

| 项 | 内容 |
|---|---|
| 来源 | B站 [BV1CZKPzDE6H](https://www.bilibili.com/video/BV1CZKPzDE6H)（搬运 The Code Otter，原 YouTube -mmIeDaua_M） |
| 发布 | 2025-06-20；Unity（ScriptableObject / SerializeReference） |
| 字幕依据 | B站 AI 字幕（全文，有少量识别重复，已去噪理解） |
| 归并目标 | ⚠️ **非动效模块**——归入 **M0 数据契约扩展（卡牌效果契约）**；为 skill 的「数据驱动」主张提供跨引擎佐证 |

---

### 【设计意图】
消灭「每张卡一个硬编码效果类」。一张「造成 5 点伤害并抽 1 张牌」的卡不应有专属代码——它是**效果组件的组合**（Damage(5) + Draw(1)）。同套架构可外推到技能/buff/物品效果。这是卡牌游戏可扩展性的根基决策，决定了后续动效系统「为什么能按数据播放」。

### 【效果拆解】（架构层，非视觉层）
| 组件 | 职能 | 驱动源 |
|---|---|---|
| CardData | 卡牌数据资产，持有 `effects[]` 有序列表 | 数据 |
| Effect（抽象基类） | 仅一个 `perform(ctx)` 抽象方法；ctx 含施法者/目标/游戏状态 | 事件（出牌时顺序执行） |
| 具体效果子类 | DrawCard、DealDamage（带 amount 字段）等 | — |
| 两种装配法 | ① Effect 资产实例（ScriptableObject）：适合效果种类少、取值范围小；② SerializeReference 纯 C# 类内联：适合每卡数值各异、复杂组合 | — |

### 【机制原理】（引擎无关的两条）
1. **效果＝实现统一接口的多态对象数组，出牌＝遍历调用 perform(ctx)**。顺序即语义（先伤害后抽牌 vs 反过来）；
2. **两种装配的取舍标准**（视频明示）：
   - 共享取值（很多卡都用「3 点伤害」）→ 效果做成共享资产，改一处全卡生效；
   - 取值高度离散（每张卡数值都不同）→ 内联序列化组件，卡内直接编辑，避免资产爆炸。
   映射到 JS：前者＝**共享效果原型/注册表引用**，后者＝**卡数据里内联 `{type:'damage', amount:5}` 纯数据 + 注册表查执行器**（后者天然可 JSON 序列化、可存档）。

### 【引擎注记】
Unity ScriptableObject / `[SerializeReference]` + Serialize Reference Editor 插件（仅参考）；JS 无 SO 概念，用「注册表 + 纯数据」同构。

### 【p5 移植映射】
| 原方案组件 | p5.js/JS 等价物 | 注意点 |
|---|---|---|
| ScriptableObject Effect | 效果注册表 `EFFECT_REGISTRY[type].perform(ctx, params)` | 逻辑与渲染分离 |
| perform(ctx) | `card.effects.forEach(e => REG[e.type](ctx, e))` | ctx 聚合 cs/gs，对齐 M0 注入点 |
| SerializeReference 内联 | 卡 JSON 内联 effects 数组（推荐） | 可直接进存档/星匣 SDK storage |
| 拖放触发 | 宿主出牌事件（M13 拖拽结束后调用） | 动效与逻辑解耦：先算结果，再播表现 |

```js
// p5 项目通用 JS 骨架：数据驱动效果管线（≤24 行）
const EFFECT_REGISTRY = {
  damage: (ctx, p) => { ctx.target.hp -= p.amount; ctx.fx.hit(ctx.target); }, // fx=表现层钩子
  draw:   (ctx, p) => { for (let k=0;k<p.count;k++) ctx.deck.drawTo(ctx.hand); },
  block:  (ctx, p) => { ctx.self.block += p.amount; },
};
function playCard(card, ctx){
  for (const e of card.effects || []) {
    const fn = EFFECT_REGISTRY[e.type];
    if (fn) fn(ctx, e); else console.warn('未注册效果:', e.type); // 坏数据不炸
  }
}
// 卡数据示例（内联装配，可序列化）：
const sigilCard = { id:'sigil', name:'深渊咒印', effects:[
  { type:'damage', amount:5 }, { type:'draw', count:1 } ] };
```

**参数表（架构参数）**：
| 决策点 | 选项 | 取舍（视频明示） |
|---|---|---|
| 效果装配 | 共享原型 / 卡内内联 | 取值共享→共享；离散→内联 |
| 表现触发 | perform 内直接调 / 事件总线 | 推荐后者：逻辑算完发事件，动效系统订阅（与 M0 注入点、M14 打击感联动） |

**时码锚点**：方法一（SO 资产）前段；方法二（SerializeReference）中段；两种方法取舍准则在结尾总结（精确秒数 `未验证`）。

**模块归并**：**M0 扩展候选**——向卡牌数据契约追加：`effects: Array<{type:string, [key:string]:any}>` + 宿主注入点 `effectRegistry`。对动效 skill 的意义：呼吸灯/打击感/镭射等表现层全部通过**效果事件**订阅播放，渲染模块永不直接写游戏状态。
