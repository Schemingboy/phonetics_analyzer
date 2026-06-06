# 英语语音语调可视化工具

更新时间：2026-06-04

## 项目定位

这是一个纯单文件 HTML 英语语音分析工具。用户输入英文句子后，AI 返回结构化 JSON，前端渲染连读、弱化、同化、省略、失去爆破、语调曲线、情绪色条、CUES 辅助提示，并支持 TTS 朗读。

核心价值不是“查字典音标”，而是帮助英语学习者看见自然语流里哪里发生了音变、听起来怎么连、朗读时如何对应视觉标注。

## 当前状态

- 当前最新工作版本：`phonetics_v14.19.html`
- 当前稳定化方向：继续打磨 v14.x，稳定后封 `phonetics_v15.html`
- 版本递增规则：新增功能或规则体系变化才开新小版本；当前版本的显示优化、文案调整、样式微调和 bug 修复直接改当前文件，并记录到 `CHANGELOG.md`。
- 当前入口文档：
  - `PROJECT.md`：项目说明
  - `CHANGELOG.md`：版本更迭与 bug 修复
  - `HANDOFF.md`：历史交接文档
  - `AGENTS.md`：项目协作规则
- GitHub 仓库 `Schemingboy/phonetics_analyzer` 已完成专业化（2026-06-04）：
  - 添加 README.md（项目介绍、功能、用法）、LICENSE（MIT）、.gitignore
  - 修复仓库描述（原为乱码）和 topics（`ai english phonetics language-learning`）
  - GitHub 操作统一用 `gh` CLI 认证，禁止明文 token

## 技术栈

- 单文件 HTML，无框架、无构建工具、无 npm、无 CDN。
- AI 分析：Anthropic / OpenAI / DeepSeek / Groq / 自定义 OpenAI 兼容接口。
- TTS：浏览器 SpeechSynthesis / OpenAI TTS / ElevenLabs / Typecast。
- 渲染：原生 DOM + SVG。
- 设置存储：`localStorage`，key 前缀为 `pa_`。

## 核心数据结构

AI 返回的核心结构：

```json
{
  "senseGroups": [
    {
      "pitchContour": [0.35, 0.7, 0.25],
      "segments": [
        {
          "words": [
            {"text": "love", "stressed": true, "writtenIpa": "/lʌv/"},
            {"text": "it", "stressed": false, "writtenIpa": "/ɪt/"}
          ],
          "phenomenon": "liaison",
          "priority": 1,
          "naturalIpa": "/lʌ vɪt/",
          "label": "连读",
          "elision": null
        }
      ]
    }
  ],
  "tone": {
    "energy": 0.4,
    "polarity": 0.5,
    "description": "中文语气说明"
  },
  "translation": "中文翻译"
}
```

## 现象枚举

| phenomenon | 优先级 | 用途 |
|---|---:|---|
| `liaison` | P1 | 辅音 + 元音连读 |
| `liaison_r` | P1 | 英式 r 连读 |
| `reduction` | P0 | 功能词弱化 |
| `assimilation` | P3 | 同化 |
| `elision` | P3 | 省略 |
| `lost_plosive` | P2 | 失去爆破 |
| `flapping` | P2 | 美式闪音 |
| `contraction` | P1 | 高频口语缩约 |
| `intrusive_r` | P2 | 英式闯入 r |

## 视觉规则

### Segment 粒度

- 普通现象 segment 默认只覆盖直接参与该现象的 1-2 个词。
- 普通词必须保留 `phenomenon:null`。
- 不允许把整句或整段短语包成一个 phenomenon segment。

### Liaison Chain

连读链服务“听感块”，不是机械拆分每一个边界。

- 默认 2-3 个词。
- 可吸收链尾弱读功能词：`a / an / it / us / of / at / on`。
- 弱读功能词也可以作为连读小块的开头；例如 `of + vowel` 应优先显示为 `of‿it -> /ə vɪt/`，不要保留成强读 `/ʌv ɪt/`。
- 不吸收 `the`，例如 `the oven` 不显示成普通连读链。
- `an + silent-h word` 优先形成小链，例如 `an‿hour -> /ə naʊər/`；当前 silent-h 词表是可扩展识别表，不是规则全集。
- 不跨标点、强停顿、强语义边界、强重读内容词边界。
- 不混入 `lost_plosive / assimilation / elision / flapping / contraction / intrusive_r`。

推荐示例：

```text
I | love‿it
We | were‿on‿a | break.
I | need | all‿of‿it | out‿of | the oven(普通相邻词) | in‿about | an‿hour
```

### 规则互斥与优先级

- 一个 segment 默认只承载一种主要现象；不要在同一 segment 里混合多个 phenomenon。
- `elision / assimilation / lost_plosive / flapping / contraction / intrusive_r` 不并入普通 liaison chain。
- `liaison_r` 是已有/拼写 r 在元音前读出，例如 `far‿away`；图例里的 `far` 只是预览样例，实际规则应覆盖拼写上以 `r/re` 结尾且后接元音的词。`intrusive_r` 是拼写里没有 r 但英音插入，例如 `idea‿r‿of`，两者不要混标。
- `reduction` 可以被 liaison chain 吸收；如果功能词已经在自然连读块里，不再单独拆一个弱化块。
- `reduction` 与 liaison 不互斥：如果弱读功能词后接元音，优先形成弱读连读小块，例如 `of‿it`。这是一条通用规则，不是 `the idea of it` 的单句兜底。
- `the + vowel` 暂不作为普通 liaison chain；`the oven` 保持视觉普通，读法由 IPA 体现。
- 节奏上连续不等于普通 liaison；普通 liaison 必须满足合法音边界，例如 `stole‿it` 可以保留，但 `he‿stole` 不应显示为普通连读。
- `lost_plosive` 表示爆破音闭住不爆开；`elision` 表示声音被省掉，二者视觉标识必须区分。
- `flapping` 优先服务美音里高频、可听见、值得教学标记的 /t,d -> ɾ/，例如 `get it / water / better`；它不混进普通 liaison chain。
- `contraction` 默认只标输入文本里已经写成缩约或口语词形的块，例如 `I'm / gonna / wanna / lemme`；不要在默认主标注里把标准写法 `I am / going to / want to / let me` 强行改写成更口语的读法。
- 默认主标注优先服务“当前文本和 TTS 大概率真的听得到”的 heard layer；`want to -> wanna` 这类可选口语读法以后应放入 optional cue layer，而不是默认层。
- `elision` 是通用省音规则：任何明确省掉、且值得教学标记的声音都用同一种删除线视觉表示；`next day` 的词尾 `t` 省略、`asked him -> asked 'im` 的功能词 `h` 省略只是例子，不是规则边界。
- 代码里的 pinned liaison fallback chunk 只用于“模型给出过长 liaison chain 时拆回可读小块”，不是 liaison 规则全集；普通 liaison 仍由通用边界规则判断。

### Lost Plosive

- `lost_plosive` 不再在词下显示中文标签“失去爆破”。
- 在发生失爆的词尾爆破字母 `p / b / t / d / k / g` 上加短斜杠。
- 短斜杠表示“闭住但不爆开”，不要和 `elision` 的删除线混用。
- 上方 natural IPA 仍保留，例如 `/tʃeɪndʒd̚ maɪ/`。

### Legend

- 图例服务“正文标记预览”，不是颜色说明表。
- 图例分为 `FLOW` 和 `CUES` 两组。
- 现有 phenomenon 必须全部能在 `FLOW` 图例中找到对应视觉标识。
- 语流现象用接近正文的英文短标记：`A‿B link`、`far‿away r-link`、浅灰 `to weak`、`d→j assimilate`、删除线 `x sound drop`、短斜杠 `d hold`、`idea‿r‿of insert-r`。
- 辅助标记只放 `pitch` 曲线、`word stress`、`pause`、`schwa cue` 和 `focus cue` 提示。
- `schwa cue` / `focus cue` 属于前端派生的 CUES 层，不进入 AI JSON 主结构，不改变 FLOW 主现象、TTS 文本、segment 边界或 `pitchContour`。
- `schwa cue` 第一版只提示高价值弱读点：功能词弱读和少数常见易读重的非重读词内 /ə/，不机械标出所有 /ə/。
- `focus cue` 第一版只服务当前句子的自然读法：每个 sense group 最多提示一个自然焦点，不做课堂式多重重音含义对比。

## 语调曲线

语调曲线用 SVG 绘制在每个 sense group 上方。

关键实现约束：

- `.sg-local-svg` 必须是绝对定位。
- `.sg-pitch-spacer` 占位高度为 48px。
- SVG class 必须用 `setAttribute('class','sg-local-svg')`。
- 曲线坐标由 `.word-el` 的实际位置计算。

## 本地运行

```powershell
python -m http.server 8765 --bind 127.0.0.1
```

访问：

```text
http://127.0.0.1:8765/phonetics_v14.19.html
```

## 测试路线

当前测试路线：先短句，再长句，最后文章。

- 短句阶段：已基本覆盖核心语流块和主要语音现象，用于快速回归。
- 长句阶段：重点验证自然语流块边界、`the` 不吸收、`an‿hour` 保留、多块并存时的可读性。
- 文章阶段：后续验证多句、多意群、语调曲线连续性和整体教学可读性。

## 下一步建议

v14.19 已新增 `schwa cue` 和 `focus cue` 的前端派生 CUES 层。当前重点是验证新增 cue 是否足够轻量、是否不干扰既有 FLOW 标注和语调曲线。

已回归的重点：

- 核心语流块测试：
  - `I love it`
  - `pick it up`
  - `We were on a break.`
  - `not that common`
  - `I need all of it out of the oven in about an hour.`
- 现象覆盖测试：
  - `Give it to me.`
  - `Did you see it?`
  - `next day`
  - `I asked him.`
  - `far away`
  - `the idea of it`
  - `I wanted to go, but I changed my mind.`
  - `Pick it up, and put it down.`
  - `get it`
  - `water`
  - `better`
  - `I'm going to go.`
  - `I wanna go.`
  - `Lemme see it.`

已确认方向：

- 多意群句子之间显示轻量 `|` pause 提示；它属于 `CUES` 层，不进入 FLOW 主现象。
- pause 提示只显现自然 spoken chunk / sense group 边界，不机械重复逗号、分号等标点；标点只是判断线索，不是显示规则。
- pause 提示不改变 AI 数据结构、TTS 文本或任何语音现象判断。
- 转折从句优先保持 `but + clause` 完整；不要把 `but it / but I / but then` 这类弱桥接词留在上一意群末尾。例如优先 `I thought it would be easy | but it turned out...`，不要 `easy but it | turned out...`。
- American 下 `get it / water / better` 可稳定标为 `flapping`。
- `I'm going to go.` 只把文本里已经写出的 `I'm` 标为 contraction，不把 `going to` 默认改成 `gonna`。
- `I wanna go.` 可稳定显示 `wanna -> /ˈwɑːnə/`。
- `Lemme see it.` 可稳定显示 `Lemme -> /ˈlemi/`，并保留 `see‿it` 连读。

后续计划：

- 重点检查 `schwa cue` 是否只出现在高价值弱读点，不形成满屏噪音。
- 重点检查 `focus cue` 是否每个 sense group 最多一个，不变成课堂式多义对比面板。
- 封版前再做一轮浏览器页面级 smoke test：页面加载、内置测试组、语调曲线、`| pause`、CUES 提示、TTS 设置区不报错。
- 后续 UX 调整：系统提示词编辑区默认隐藏，普通用户界面不提供入口；仅保留开发者打开方式，用于调试 prompt 和缓存版本。
- 决定是否把 `index.html` 指向 v14.19；如果要对外使用，优先做这一步。
- 如果 v14.19 继续稳定，再复制封为 `phonetics_v15.html`，并把 v14.19 保留为回退基线。
- `glottal stop` 暂缓；它口音差异更大，且容易和 `lost_plosive` 视觉含义冲突。
- `thought group cue` 后续可考虑加入更明确的停顿时长或一句式提示，但不进入 FLOW 主图例。
