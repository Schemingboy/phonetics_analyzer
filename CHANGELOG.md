# 版本更迭与 Bug 修复记录

更新时间：2026-06-07

## 当前主线

最新工作版本：`phonetics_v14.23.html`

当前策略：继续用 v14.x 小版本修稳定性和教学显示问题；当核心回归句稳定后，再封 `phonetics_v15.html`。

## v14.23

目标：Batch 4 Prompt 扩充，保护用户自定义 Prompt，并补强 H-dropping 教学覆盖；本批只改 Prompt、Prompt 缓存保护和版本文档，不改前端渲染逻辑。

主要变化：

- 新建 `phonetics_v14.23.html`，基于 `phonetics_v14.22.html` 复制。
- 页面版本号、CORS 提示 URL 和 `PROMPT_VERSION` 升级为 `v14.23-prompt-batch4`。
- 为系统 Prompt 编辑框增加 `pa_prompt_is_custom` 保护标记：
  - 用户手动编辑 `#sp-ta` 时写入 `pa_prompt_is_custom = '1'`。
  - 点击“重置提示词”恢复默认 Prompt 时清除该标记。
  - 默认 Prompt 自动升级时跳过已标记的自定义 Prompt，避免误判后静默覆盖。
- 默认 Prompt 自动升级发生时显示状态提示：`提示词已自动升级至最新版本`。
- `intrusive_r` 在 `DEFAULT_SP` 中降为英式低优先级可选规则；American 不标，不确定时跳过。
- 在 `assimilation` 的 Prompt 说明中追加 H-dropping 子规则：辅音结尾词 + 非重读 `he/him/her/his/have/has/had` 时，整体标为 `assimilation`，`naturalIpa` 显示 H 脱落后的连接读法。
- 真实 API 回归后追加收紧：H-dropping 的 consonant-final 必须按发音判断，不按拼写判断；`I saw him` 作为反例，避免把元音结尾的 `saw him` 误标为 `assimilation`。
- 不改变 phenomenon enum、颜色表、图例、`renderSegment()`、TTS、AI JSON 结构或 `index.html`。

验证：

- 静态检查通过：`phonetics_v14.23.html` 不再残留 `phonetics_v14.22.html`、`v14.22-review-batch3`、`review-batch3`、`v14.21` 或 `review-batch2`。
- 静态检查通过：`PROMPT_VERSION` 为 `v14.23-prompt-batch4`，`DEFAULT_SP` 包含 `H-dropping`、`call him`、`tell her`、`LOW PRIORITY` 和 `Do NOT mark in American accent`。
- 差异检查通过：相对 v14.22，只改页面版本标识、Prompt 文本、Prompt 缓存保护、`resetSP()` 清理标记和 `_statusTimer` 初始化顺序；未改 `renderSegment()`、颜色表、图例、TTS、AI JSON 结构或 `index.html`。
- JavaScript 脚本语法检查通过。
- Prompt 缓存逻辑模拟验证通过：
  - 空缓存会自动写入 v14.23 默认 Prompt，包含 H-dropping 和 intrusive_r 低优先级规则，并显示自动升级提示。
  - 带 `pa_prompt_is_custom = '1'` 的旧版相似自定义 Prompt 不会被覆盖。
  - 用户编辑 `#sp-ta` 会保存 Prompt 并写入 `pa_prompt_is_custom = '1'`。
  - `resetSP()` 会写回 v14.23 默认 Prompt、更新 `pa_prompt_version` 并清除 `pa_prompt_is_custom`。
- 页面级 smoke test 通过：本地服务器打开 `http://127.0.0.1:8767/phonetics_v14.23.html`，页面标题和 header 均显示 v14.23，控制台无 error/warning。
- 真实 API 回归通过（DeepSeek `deepseek-chat`，页面已有 Key 配置，手动刷新到 v14.23 默认 Prompt 后测试）：
  - `call him` 返回并渲染为单个 `assimilation` segment，IPA `/kɔːl ɪm/`，页面显示中文标签 `同化`。
  - `tell her` 返回并渲染为单个 `assimilation` segment，IPA `/tɛl ɜr/`，页面显示中文标签 `同化`。
  - `give him a call` 返回并渲染为 `give him` 的 `assimilation` segment，IPA `/ɡɪv ɪm/`，`a / call` 保持普通词。
  - 对照句 `He called me.` 保持普通词，不触发 H-dropping。
  - 对照句 `I saw him.` 初次回归被误标为 `assimilation`；补充“按发音判断 consonant-final，不按拼写判断”和 `I saw him` 反例后复测通过，不再显示同化标签。

## v14.22 - Review Batch 3 rendering and TTS usability

时间：2026-06-07

目标：完成审查报告 Batch 3 的渲染逻辑和 TTS 可用性修复，让 `lost_plosive` 的视觉语义更清楚，并让朗读控制与当前标注区对齐。

主要变化：
- 新建并同步 `phonetics_v14.22.html` 到主文件夹，基于已完成的 `phonetics_v14.21.html`。
- 页面版本号、CORS 提示 URL 和 `PROMPT_VERSION` 升级为 `v14.22-review-batch3`。
- `lost_plosive` 从 CSS 伪元素斜杠改为 `˺` 上标：删除 `.lost-plosive-char::after`，并在 `renderSegment()` 中给词尾爆破音字母追加青绿色 `˺`。
- FLOW 图例同步显示 `d˺ hold`，与正文渲染一致。
- TTS 引擎选择器从折叠的 API 设置卡移到结果区 `.audio-section`，与 Play 按钮和倍速控制放在同一视觉区域。
- browser TTS 播放时基于 `SpeechSynthesisUtterance.boundary` 的 `charIndex` 映射当前 sense group，并给 `.sg-wrap-item` 加 `.playing` 浅蓝背景。
- OpenAI / ElevenLabs / Typecast TTS 暂不做 word-level 时间轴映射；MVP 为播放开始高亮第一个 sense group，播放结束、停止或错误后清除。
- 真实 API 回归后追加修复：多词 `lost_plosive` segment 只给边界前的词加 `˺`；单词 segment 仍正常标自身，避免 `hot dog` 被显示成 `hot˺ dog˺`。
- 不改变 AI JSON 结构、Prompt 规则正文、FLOW 主现象判断、TTS 文本或 `index.html`。

验证：
- 静态检查通过：`phonetics_v14.22.html` 中不再存在 `.lost-plosive-char::after`、`v14.21`、`phonetics_v14.21` 或 `review-batch2` 残留。
- 静态检查通过：`id="tts-engine"` 和 `id="tts-engine-hint"` 均只有一个，且已位于结果区 `.audio-section`。
- JavaScript 脚本语法检查通过。
- 页面级 DOM smoke test 通过：本地服务器打开 `http://127.0.0.1:8766/phonetics_v14.22.html`，页面标题为 v14.22，FLOW 图例显示 `d˺`，TTS 引擎选择器在 `.audio-section` 内且不在 `api-body` 内，控制台无错误或警告。
- 真实 API 回归通过（DeepSeek `deepseek-chat`）：`hot dog` 渲染为 `hot˺ dog`；`good morning` 渲染为 `good˺ morning`；`I wanted to go, but I changed my mind.` 返回 2 个 sense group；`I need all of it out of the oven in about an hour.` 返回 3 个 sense group，连读小块正常，语调曲线 path 非空。

## v14.21 - Review Batch 2 UI cleanup

时间：2026-06-07

目标：完成审查报告 Batch 2 的 3 个独立 UI / JS 清理任务，并将版本命名为 `v14.21`。Batch 1 对应 `v14.20`，Batch 2 对应 `v14.21`。

主要变化：
- 新建并同步 `phonetics_v14.21.html` 到主文件夹，保留 GitHub 入口 `index.html` 为 `v14.15`，不更新线上入口。
- 无 API Key 点击分析时，自动展开 API 设置卡，状态提示改为 `请在 API 设置中填入 Key ↓`，并滚动到 Key 输入框。
- 将 FLOW / CUES 图例移动到意群标注区上方，紧跟结果区视图切换按钮之后，方便边看图例边对照标注。
- 清理 `populateELSelect` 内残留废代码：删除未使用的推荐 optgroup、无效 `addOpt` 调用和死函数，保留 ElevenLabs 推荐 / 其他音色分组逻辑。

验证：
- JavaScript 语法检查通过。
- `git diff --check` 通过。
- 页面级验证通过：无 Key 时 API 设置自动展开并定位到 Key 输入框；FLOW / CUES 图例位于 `#sg-row` 上方；ElevenLabs 音色下拉列表正常显示。
- 主文件夹 `index.html` 确认仍为 `v14.15`。

## v14.19

目标：新增 `schwa cue` 和 `focus cue` 两个 CUES 辅助提示，帮助自主学习者理解自然弱读和当前句子的自然焦点。

主要变化：

- 新建 `phonetics_v14.19.html`，保留 v14.18 作为稳定回退基线。
- 页面版本号、CORS 提示 URL 和 `PROMPT_VERSION` 升级为 `v14.19-cues-layer`。
- 在 `CUES` 图例中新增 `ə weak sound` 和 `focus`。
- 新增前端派生 CUES 层，不扩展 AI JSON 主结构：
  - `schwa cue` 只提示高价值弱读点，包括功能词弱读和少数常见易读重的非重读词内 /ə/。
  - `focus cue` 基于 `word.stressed`、`pitchContour` 高点、内容词和词序，为每个 sense group 最多提示一个自然焦点。
- 新增独立 `.seg-cue-row`，避免 CUES 与 liaison 下方 IPA、lost plosive 短斜杠、elision 删除线和 pause cue 重叠。
- 追加修复：模型若把不合法的 liaison 链整段返回，例如 `he stole it[liaison]`，前端会重新校验链内边界并拆成 `he | stole‿it`，不再把元音结尾 + 辅音开头的 `he‿stole` 误显示为普通连读。
- `coverage` 测试组新增：
  - `I thought it would be easy, but it turned out to be harder than I expected.`
  - `I didn't say he stole it.`
- 不改变 FLOW 主现象、segment 边界、TTS 文本、`stressed`、`pitchContour` 或 AI 数据结构。
- `PROJECT.md` 更新当前版本、CUES 层规则和访问 URL。

Batch 1 — 零风险 bug 修复（2026-06-07）：

- 修复 `elision` 删除线颜色：`.elided-char` 和图例 `.leg-s` 改用 `--c-elision`，与弱化灰色区分。
- 示例句按钮点击后直接填入文本并触发 `analyze()`，减少一步手动操作。
- 修复 `wordHasSchwaCue` 恒真问题：`SCHWA_CONTENT_WORDS` 命中后仍必须由 IPA 内容确认 schwa。
- 删除 `STORE.fields` 中没有 DOM 对应的 `prompt-version` 幽灵 key；`pa_prompt_version` 仍由专门逻辑读写。
- 分析加载期间给旧结果区添加 `.stale` 半透明遮罩，成功或失败后移除，避免用户误把旧结果当新结果。

验证：

- JavaScript 脚本语法检查通过。
- 代码级检查通过：v14.19 页面标题、CORS URL、`PROMPT_VERSION` 和新增 CUES 图例均已更新。
- 代码级检查通过：已加入 liaison segment 内部边界校验，`he stole it` 这类模型过度合并会保留合法的 `stole‿it`，拆掉非法的 `he‿stole`。
- Batch 1 代码级检查通过：目标 CSS、示例句点击、schwa cue、STORE 字段和 stale overlay 均已落在 `phonetics_v14.19.html`。
- 页面级轻量 smoke test 通过：Codex 内置浏览器打开 `http://127.0.0.1:8765/phonetics_v14.19.html`，标题、输入框、结果区和 `.stale` 样式规则正常，控制台无错误。
- 待真实 API 回归：确认 CUES 提示轻量显示且不与现有 FLOW 标注重叠。
- 后续 UX 计划：系统提示词编辑区默认隐藏，普通用户界面不提供入口；仅保留开发者打开方式，用于调试 prompt 和缓存版本。

## v14.18

目标：加入 `thought group cue` 最小版，让多意群句子的分块和停顿位置更清楚。

主要变化：

- 新建 `phonetics_v14.18.html`，保留 v14.17 作为稳定回退基线。
- 页面版本号和 CORS 提示 URL 升级为 `v14.18`。
- `PROMPT_VERSION` 内部缓存标记升级为 `v14.18-contrast-clause-boundary`。
- 在 `CUES` 图例中新增 `| pause`。
- 在多个 sense group 之间显示轻量 `|` pause 提示：
  - 普通横排模式下，`|` 作为意群之间的独立停顿提示。
  - wrap 长句模式下，`|` 贴在后续意群左侧，避免换行时单独占位跑偏。
- 追加修正：撤回同一 sense group 内的逗号 inline pause；pause 不应机械重复标点作用，而应只显现自然 spoken chunk / sense group 边界。
- 追加修正：强化 prompt 的 sense group 规则，明确标点只是线索，不是分组规则；无标点但母语者自然分口的位置才是 pause cue 的主要价值。
- 追加修正：强化转折从句分组规则，优先保持 `but + clause` 完整，避免 `easy but it | turned out...` 这类把弱桥接词留在上一意群末尾的切法。
- 新增前端兜底 `fixContrastSenseGroupBoundaries`，覆盖 `but it / but I / but then + turned out / seems / feels / became / realized...` 这类高频转折结构。
- 追加修正：扩展旧默认 prompt 识别，避免浏览器继续沿用 v14.17 之前缓存的系统提示词，导致 v14.18 的新规则不生效。
- 追加修复：DeepSeek 真实 API 回归暴露出 `pitchContour` 有时按 segment 数返回，而不是按 word 数返回；新增前端兜底，自动展开或重采样为每个词一个曲线点，避免 SVG 语调曲线对齐不稳。
- 追加优化：旧稳定兜底不再把多意群测试句强行压回单个 sense group；`Pick it up, and put it down.`、`I need all of it out of the oven in about an hour.`、`I wanted to go, but I changed my mind.` 会保留合理分组，让 `| pause` cue 有实际显示空间。
- 不改变 AI 数据结构、TTS 文本、FLOW 主现象或任何发音规则判断。
- `PROJECT.md` 更新当前版本、访问 URL 和验证重点。

验证：

- JavaScript 脚本语法检查通过。
- 本地服务器可访问 `http://127.0.0.1:8765/phonetics_v14.18.html`。
- 侧边栏页面冒烟检查通过：页面标题显示 v14.18，`CUES` 图例包含 `pause`。
- 侧边栏 DOM 检查通过：系统提示词 textarea 已刷新到 v14.18 默认 prompt，包含 `flapping` 和新的 `contraction` heard-layer 规则。
- DeepSeek 测试 API 可正常返回 JSON；回归句暴露的主要问题是 `pitchContour` 长度漂移，已由前端兜底修复。
- 修复后 JavaScript 语法检查通过，本地页面可加载，控制台无本项目运行错误。
- 代码级结构检查通过：三个旧稳定兜底句分别保留 2 / 3 / 2 个 sense group，避免 pause cue 被单组兜底吞掉。
- 用户页面级验证反馈：基础回归基本没问题，v14.18 可作为稳定候选进入封版前检查。
- 已更新 `PROJECT.md`：下一阶段从“补规则”切换为“封版前检查”，暂不自动更新 `index.html`，也暂不自动封 `phonetics_v15.html`。

## v14.17

目标：修正 v14.16 新增 `flapping / contraction` 后，正文标注下方直接显示中文术语“闪音 / 缩约”的问题。

追加优化：收窄默认 `contraction` 的触发边界，让默认主标注和当前文本/TTS 更一致。

判断：

- `flapping / contraction` 作为内部 phenomenon 是需要的：
  - 用来告诉前端这是哪类语流现象。
  - 用来控制颜色、优先级、IPA 显示和是否并入 liaison chain。
  - 用来让 prompt 约束模型输出，避免把 `get it` 随机标成普通连读。
- 但正文下方不需要显示“闪音 / 缩约”这两个中文术语：
  - 这会让界面回到“语音学报告感”。
  - 学习者真正需要看到的是词面变化、自然 IPA 和图例预览。
  - 与此前去掉 `弱化 / 失去爆破 / 省略` 中文下标的方向一致。

主要变化：

- 新建 `phonetics_v14.17.html`，保留 v14.16 作为回退基线。
- 页面版本号、`PROMPT_VERSION`、CORS 提示 URL 升级为 `v14.17`。
- `renderSegment` 中正文 label 渲染排除 `flapping / contraction`。
- `flapping / contraction` 仍保留在 prompt、数据结构、颜色表、图例和稳定兜底中。
- `PROJECT.md` 更新当前版本与验证 URL。
- `contraction` 默认层改为只标输入文本里已经写成缩约或口语词形的块，例如 `I'm / gonna / wanna / lemme`。
- 不再把标准写法 `I am / going to / want to / let me` 默认改写成 `I'm / gonna / wanna / lemme`。
- 测试句从 `I am going to go. / I want to go. / Let me see it.` 调整为 `I'm going to go. / I wanna go. / Lemme see it.`。
- `PROMPT_VERSION` 内部缓存标记改为 `v14.17-heard-layer`，确保浏览器刷新后使用新的默认提示词；页面版本仍为 `v14.17`。

验证：

- JavaScript 脚本语法检查通过。
- 代码级检查通过：正文 label 渲染已排除 `flapping / contraction`，因此 `I am going to go.` 不会再在词下显示“缩约”，`get it / water / better` 不会再在词下显示“闪音”。
- 本地服务器可访问 `http://127.0.0.1:8765/phonetics_v14.17.html`。

## v14.16

目标：在 v14.15 稳定基线上加入 `flapping` 和 `contraction`，优先覆盖学习者最常听见、最值得视觉化的美音闪音和高频口语缩约。

主要变化：

- 新建 `phonetics_v14.16.html`，保留 v14.15 作为稳定回退基线。
- 页面版本号、`PROMPT_VERSION`、CORS 提示 URL 升级为 `v14.16`。
- 新增 `flapping`：
  - American-only，用于 /t,d -> ɾ/，例如 `get it / water / better`。
  - 作为独立 FLOW 现象，不并入普通 liaison chain。
  - 图例新增 `t→ɾ flap`。
- 新增 `contraction`：
  - 用于高频固定口语块，例如 `I am -> I'm`、`going to -> gonna`、`let me -> lemme`。
  - 不把所有弱读泛化成缩约。
  - 图例新增 `I am→I'm contract`。
- 现象覆盖测试组新增：
  - `get it`
  - `water`
  - `better`
  - `I am going to go.`
  - `I want to go.`
  - `Let me see it.`
- 前端稳定兜底新增：
  - American `get it -> /ɡeɾ ɪt/`
  - American `water -> /ˈwɔːɾər/`
  - American `better -> /ˈbeɾər/`
  - `I am going to go.` 拆成 `I am` 缩约、`going to` 缩约、`go.` 普通词。
  - `I want to go.` 拆成 `I` 普通词、`want to` 缩约、`go.` 普通词。
  - `Let me see it.` 拆成 `Let me` 缩约、`see‿it` 连读。
- `PROJECT.md` 更新当前版本、现象枚举、规则互斥和下一步验证路线。

验证：

- JavaScript 脚本语法检查通过。
- 代码级标记检查通过：`flapping / contraction` 已进入 prompt、颜色表、图例、测试句和稳定兜底。
- 尚未做真实 API 回归；下一步需要在页面里用 v14.16 跑新增测试句和旧稳定句。

## v14.15

目标：review/debug v14.14 最新版，修正真实 DeepSeek 测试中暴露的显示稳定性问题。

主要变化：

- 新建 `phonetics_v14.15.html`，保留 v14.14 作为回退基线。
- 页面版本号、`PROMPT_VERSION`、CORS 提示 URL 升级为 `v14.15`。
- 修复 `liaison_r` 在前端合并阶段被降级成普通 `liaison` 的问题：
  - 新增 `liaisonPhenomenonForWords` / `liaisonLabelForPhenomenon` / `makeDetectedLiaisonChain`。
  - British 下 `r/re + vowel` 自动候选保留为 `liaison_r`，标签保留 `r连读`。
  - 模型已经返回 `liaison_r` 时，后续链合并不再强行改成普通 `liaison`。
- 增加 `normalizeIpaText`，统一模型返回的 IPA 显示：
  - 模型返回 `fɑr əˈweɪ` 时，前端规范化为 `/fɑr əˈweɪ/`。
  - `naturalIpa` 与 `writtenIpa` 都走同一兜底，避免斜杠显示不一致。

验证：

- JavaScript 脚本语法检查通过。
- 本地服务器 `http://127.0.0.1:8765/phonetics_v14.15.html` 可打开，页面标题显示 v14.15。
- DeepSeek API 真实测试通过：
  - `Pick it up, and put it down.` 稳定显示为 `Pick‿it‿up, | and | put‿it | down.`，红色语调曲线 path 非空。
  - `I need all of it out of the oven in about an hour.` 稳定显示为 `I | need | all‿of‿it | out‿of | the | oven | in‿about | an‿hour`，`the oven` 未被普通 liaison chain 吸收。
  - `far away` API 返回后页面显示 IPA 斜杠正常：`/fɑr əˈweɪ/`。
  - `the idea of it` API 返回后页面显示 IPA 斜杠正常：`/aɪˈdiːə ʌv ɪt/`。
- 页面控制台无本项目运行时报错。

## v14.14

目标：把图例进一步改成英文符号化预览，并为 `Pick it up, and put it down.` 加稳定兜底。

主要变化：

- 新建 `phonetics_v14.14.html`，保留 v14.13 作为回退基线。
- 页面版本号和 `PROMPT_VERSION` 升级为 `v14.14`。
- 图例组名从中文改为英文短标题：
  - `FLOW`
  - `CUES`
- 图例标签改为英文符号化表达：
  - `A‿B link`
  - `far‿away r-link`
  - `to weak`
  - `d→j assimilate`
  - `x sound drop`
  - `d hold`
  - `idea‿r‿of insert-r`
  - `pitch`
  - `word stress`
- Prompt 增加 `Pick it up, and put it down.` canonical 示例。
- 前端 `stabilizeAnalysisData` 增加该句固定兜底：
  - `Pick‿it‿up,`
  - `and` 弱化为 `/ən/`
  - `put‿it`
  - `down.`
- `naturalIpaForLiaisonChain` 和已知 liaison chunk 增加 `put it -> /pʊ tɪt/`。
- 修正 `reduction` 渲染：不再在词下显示中文 `弱化` label，只保留浅灰词面和上方 natural IPA。
- 补齐 FLOW 图例覆盖：`liaison_r` 显式标为 `r-link`，`intrusive_r` 显式标为 `insert-r`。
- 将 `elision` 图例从具体例子收敛为通用 `x sound drop`，避免误解为只支持 `t/h` 两类省音。
- 排查“例子误收窄为规则”的残留问题：
  - `silent-h word` 改为可扩展词表表达，并扩展 `honour / honorable / herb / hours` 等常见变体。
  - `isKnownLiaisonChunk` 重命名为 `isPinnedLiaisonFallbackChunk`，明确它只服务过长连读链的兜底拆分，不代表 liaison 规则全集。
  - r-link 自动候选从 `far / for / there / where` 等例子列表改为 `r/re` 结尾规则，避免漏掉 `more apples / near it / here again` 这类同规则场景。
- 补充 `reduction + liaison` 通用规则：弱读功能词后接元音时可作为连读小块开头，例如 `of‿it -> /ə vɪt/`，避免显示为强读 `/ʌv ɪt/`。
- 修正 `file://` / CORS 提示：旧文案里的 `python3 -m http.server 8080` 和 `phonetics_analyzer.html` 改为当前项目命令 `python -m http.server 8765 --bind 127.0.0.1` 与 `phonetics_v14.14.html`。
- 在 `PROJECT.md` 增加规则互斥与优先级说明，明确 `liaison_r / intrusive_r`、`lost_plosive / elision`、`reduction / liaison chain` 的边界。
- 写入后续计划：下一步优先加 `flapping`，再考虑 `contraction`，`glottal stop` 暂缓。
- 写入 `CUES` 层后续计划：`schwa cue`、`focus stress cue`、`thought group cue`；这些只作为读法提示，不进入 FLOW 主图例。

验证：

- JavaScript 脚本语法检查通过。
- 固定句兜底验证通过：输出 4 个 segment，顺序为 `liaison | reduction | liaison | plain`。
- `Pick it up, and put it down.` 的 pitchContour 为 7 点，与 7 个词对齐。
- 页面代码包含 `FLOW` / `CUES` 英文图例标题和 `v14.14` 版本号。
- 代码级检查通过：`reduction` 与 `lost_plosive` 一样被排除在中文下标渲染之外。
- 代码级检查通过：`elision` 也被排除在中文下标渲染之外，避免页面显示中文 `省略`。
- 规则覆盖检查通过：当前 7 个 phenomenon 都有 FLOW 图例对应项。
- 代码级验证通过：`idea | of(reduction) | it` 会收束为 `idea | of‿it`，natural IPA 为 `/ə vɪt/`。
- 代码级检查通过：旧端口 `8080` 和旧文件名 `phonetics_analyzer.html` 已不再出现在 v14.14 页面提示中。

## v14.13

目标：把图例从“颜色说明表”升级成“正文标记预览”，减少术语堆叠感。

主要变化：

- 新建 `phonetics_v14.13.html`，保留 v14.12 作为回退基线。
- 页面版本号和 `PROMPT_VERSION` 升级为 `v14.13`。
- 图例分为两组：
  - `语流现象`
  - `辅助标记`
- 图例标识改成接近正文的预览：
  - `A‿B 连读`
  - 浅灰 `to 弱化`
  - `d→j 同化`
  - 删除线 `t 省略`
  - 短斜杠 `d 失爆`
  - 红线 `语调`
  - 蓝色加粗 `word 重读`
- 移除单独的“省略字母”图例，避免和“省略”重复。

验证：

- JavaScript 语法检查通过。
- 本地浏览器可打开 `phonetics_v14.13.html`，页面标题显示 v14.13。
- 图例文本包含 `语流现象` / `辅助标记` 两组。
- 图例包含 `d→j 同化`、浅灰 `to 弱化`、短斜杠 `d 失爆`、蓝色 `word 重读`。
- 图例不再包含重复的“省略字母”。

## v14.12

目标：把 `lost_plosive` 从词下中文标签改成词尾字母短斜杠，减少“报告感”，让学习者直接看到哪个爆破音没爆出来。

主要变化：

- 新建 `phonetics_v14.12.html`，保留 v14.11 作为回退基线。
- 页面版本号和 `PROMPT_VERSION` 升级为 `v14.12`。
- 新增 `.lost-plosive-char` 词面标记样式：
  - 只在词尾 `p / b / t / d / k / g` 上显示短斜杠。
  - 不使用删除线，避免和 `elision` 混淆。
- `renderSegment` 对 `lost_plosive` 特殊渲染：
  - 保留上方 natural IPA。
  - 保留 P2 虚线下划线。
  - 不再显示下方中文“失去爆破”。
- 图例从“绿色点 + 失去爆破”改成“带短斜杠的 d + 失去爆破”。

验证：

- JavaScript 语法检查通过。
- 本地浏览器可打开 `phonetics_v14.12.html`，页面标题显示 v14.12。
- 代码级检查通过：`.lost-plosive-char` 样式存在，`lost_plosive` 下方中文 label 被抑制，图例已切换为短斜杠示例。
- 词尾爆破定位验证通过：`not -> t`、`that -> t`、`changed -> d`，`my` 不误标。

## v14.11

目标：修正短句多现象测试句 `I wanted to go, but I changed my mind.` 的标注质量。

问题：

- DeepSeek 测试时只给出 `wanted to` 和 `changed my` 两个现象。
- `wanted to` 给成 `/ˈwɑːntɪd tuː/`，太接近逐词仔细读，没有体现 `to -> /tə/` 弱化。
- `changed my` 给成 `/tʃeɪndʒd maɪ/`，没有体现结尾 /d/ 在 /m/ 前弱释放或失去爆破。

主要变化：

- 新建 `phonetics_v14.11.html`，保留 v14.10 作为回退基线。
- 页面版本号和 `PROMPT_VERSION` 升级为 `v14.11`。
- Prompt 增加该句 canonical 示例：
  - `wanted to -> /ˈwɑːnɪdə/`，标为弱化。
  - `go,` 保持普通词，不跨逗号连到 `but`。
  - `but I -> /bə taɪ/`，标为弱化。
  - `changed my -> /tʃeɪndʒd̚ maɪ/`，标为失去爆破。
- 前端 `stabilizeAnalysisData` 增加该句稳定兜底，确保模型输出不稳时仍能显示正确教学小块。

验证：

- JavaScript 语法检查通过。
- 前端稳定兜底验证通过：`I | wanted to | go, | but I | changed my | mind.`。
- 现象序列为 `plain | reduction | plain | reduction | lost_plosive | plain`。
- pitchContour 词数保持 9:9。
- 本地浏览器可打开 `phonetics_v14.11.html`，页面标题显示 v14.11，`现象覆盖` 组仍完整显示 8 条。

## v14.10

目标：把短句阶段的回归测试入口做进页面，方便后续稳定验证。

主要变化：

- 新建 `phonetics_v14.10.html`，保留 v14.9 作为回退基线。
- 页面版本号和 `PROMPT_VERSION` 升级为 `v14.10`。
- 示例卡标题从“示例句子（美剧经典台词）”改为“示例与测试句”。
- 新增两个测试组：
  - `核心语流块`：覆盖 `love‿it / pick‿it‿up / were‿on‿a / lost_plosive 粒度 / 长句自然小块`。
  - `现象覆盖`：覆盖 reduction、assimilation、elision、liaison_r、intrusive_r、多意群和标点边界。
- 测试组会完整显示全部句子；美剧示例仍保持随机显示 5 条。

测试路线：

- 当前先完成短句阶段。
- 后续逐步进入长句阶段。
- 最后再进入文章阶段，验证多句、多意群和整体教学可读性。

验证：

- JavaScript 语法检查通过。
- 本地浏览器可打开 `phonetics_v14.10.html`，页面标题显示 v14.10。
- `核心语流块` 测试组完整显示 5 条。
- `现象覆盖` 测试组完整显示 8 条。

## v14.9

目标：继续推进“看得懂的自然语流块”，把模型可能返回的过长 liaison segment 收束成稳定小块。

主要变化：

- 新建 `phonetics_v14.9.html`，保留 v14.8 作为回退基线。
- `PROMPT_VERSION` 升级为 `v14.9`，避免继续复用旧默认提示缓存。
- 新增前端可读拆链兜底：
  - 超过 3 个词的 liaison segment 会按已知自然块重切。
  - 已知块包括 `all‿of‿it / out‿of / in‿about / an‿hour / were‿on‿a / pick‿it‿up / love‿it`。
  - `the` 仍保持普通词，不吸入普通 liaison chain。
- 新增长句稳定兜底：
  - `I need all of it out of the oven in about an hour.`
  - 固定显示为 `I | need | all‿of‿it | out‿of | the oven(普通相邻词) | in‿about | an‿hour`。
- 新增固定 IPA：`all of it -> /ɔː lə vɪt/`。

验证：

- JavaScript 语法检查通过。
- 本地浏览器可打开 `phonetics_v14.9.html`，页面标题显示 v14.9。
- 前端稳定兜底验证通过：长句收束为 `I | need | all‿of‿it | out‿of | the | oven | in‿about | an‿hour`，pitchContour 词数保持 13:13。
- 仍需用真实 API 配置实测模型返回后的完整页面渲染和红色语调曲线。

## v14.8

目标：修复 v14.7 长句连读链边界过度合并。

主要变化：

- 新增 `the + vowel` 规则：不把 `the oven` 当普通 liaison chain 显示。
- 新增 `an + silent-h word` 规则：优先显示 `an‿hour -> /ə naʊər/`。
- 从链尾可吸收功能词中移除 `the`。
- 新增 silent-h 识别：`hour / honest / honor / heir`。
- 新增固定 IPA：
  - `out of -> /aʊ dəv/`
  - `of it -> /ə vɪt/`
  - `in about -> /ɪ nəbaʊt/`
  - `an hour -> /ə naʊər/`
- 新增已有链纠偏：
  - 模型返回 `out‿of‿the` 时，前端拆成 `out‿of` + `the`。
  - 模型返回 `in‿about‿an | hour` 时，前端把 `an` 移到 `hour` 前，形成 `an‿hour`。

待验证：

- `I need all of it out of the oven in about an hour.`
- 预期不再显示 `out‿of‿the`。
- 预期能稳定显示或保留 `an‿hour`。

## v14.7

目标：把 `were‿on‿a` 从单句判断升级成通用 liaison chain 规则。

主要变化：

- 新增 `LIAISON CHAIN RULE`。
- `liaison` 允许 2-3 词自然听感块。
- 功能词可在链尾弱读并入，例如 `were‿on‿a`。
- 不跨标点、强停顿、强语义边界、重读内容词边界。
- 前端新增通用合并逻辑：
  - `mergeLiaisonChains`
  - `canAbsorbIntoLiaisonChain`
  - `canStartLiaisonChain`
  - `naturalIpaForLiaisonChain`

已知问题：

- 对长句会过度吸收 `the`，例如 `out‿of‿the`。
- 对 `an hour` 可能拆成 `in‿about‿an | hour`。

这些问题已在 v14.8 处理。

## v14.6

目标：解决同一句重复分析时标注不稳定的问题。

主要变化：

- API 请求加入 `temperature:0`。
- `PROMPT_VERSION` 升级为 `v14.6`。
- Prompt 增加 canonical examples：
  - `I love it`
  - `pick it up`
  - `not that common`
- 新增前端稳定兜底 `stabilizeAnalysisData(data, sentence, accent)`。
- 固定高频教学例：
  - `I love it -> I | love‿it`
  - `pick it up -> pick‿it‿up`
  - `not that common` 不被 liaison 合并，保持 lost_plosive 粒度。

## v14.5

目标：优化 IPA 字体和连读 IPA 的视觉位置。

主要变化：

- 新增 IPA 字体栈：
  - `Charis SIL`
  - `Gentium Plus`
  - `Segoe UI`
  - `Arial Unicode MS`
  - `Noto Sans`
- `.seg-nat-ipa`、`.word-wipa`、`.seg-below-ipa` 不再使用 `Courier New`。
- 连读 IPA 作为整个 segment 的下方标签居中显示。
- `.seg-label-row` 高度从 17px 增加到 22px。
- 给连读 segment 增加 `liaison-segment` class。

## v14.4

目标：系统提示词从“语音学报告”转向“教学可视化标注”。

主要变化：

- 重写 `DEFAULT_SP` 为稳健短版 prompt。
- 强化 JSON-only 输出。
- 强化 Segment Scope。
- 强化 pitchContour 为 SVG 曲线服务。
- 强化 tone.description 面向英语老师解释。
- 增加 `PROMPT_VERSION` 和 `pa_prompt_version` 缓存机制。

## v14.3 修复后状态

目标：修复长期存在的语调曲线不可见问题。

关键 bug：

- 代码用 `svg.className='sg-local-svg'` 给 SVG 元素设置 class。
- 对 SVG 元素来说，这不会可靠写入普通 `class` 属性。
- 结果：
  - CSS `.sg-local-svg` 不生效。
  - `drawAllLocalCurves()` 查不到 SVG。
  - path 的 `d` 为空。
  - 页面看不到红色语调曲线。

修复：

```javascript
svg.setAttribute('class','sg-local-svg');
```

验证：

- 分析后 `.sg-local-svg` 能被查询到。
- path `d` 有实际贝塞尔路径。
- 红色语调曲线可显示。

## 仍需关注

- v14.9 长句边界和曲线显示需要实测。
- `the + vowel` 暂不显示为普通连读链，但是否需要轻提示 `the -> /ði/` 可作为后续 UX 讨论。
- 未来封 v15 前，应统一更新 `index.html` 指向稳定版本。
- `phonetics_v14.21.html` 已进入主文件夹本地提交，`phonetics_v14.22.html` 已同步到主文件夹；后续版本文件是否纳入版本控制，仍需按任务范围单独确认。
