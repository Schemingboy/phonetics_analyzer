# 英语语音分析器 v14.19 — 产品审查报告（Round 1+2 综合版）

> 本报告综合了两轮审查的结论：Round 1 覆盖 UI/UX 与代码问题，Round 2 覆盖音变体系与符号系统设计。所有来自 Round 2 的条目标注 `[R2]`，以便区分来源和优先级。

---

## Part A — 产品层判断

这个工具在语言学习市场填补了真实空白：句子级连读可视化没有直接竞品。核心假设"看到变化 → 理解听觉 → 模仿跟读"在认知科学上成立（视听联结比纯听输入的编码效率更高）。但当前实现只完成了"看到变化"这一步——学习闭环没有形成，"模仿跟读"完全缺席。

音变体系（9 种现象）整体合理，但存在两个明显盲区：H 击穿（call him → callim）在高频日常对话中极为常见，却无处理；美式闪音（flapping）是美式英语最具特征的音变，参考资料竟然完全漏掉，幸好当前系统已涵盖。

符号系统有一个确认性 bug（`elision` 颜色在代码里写死为 `#ccc`，与 `reduction` 的 `#ccc` 完全一致）和一个设计缺陷（`lost_plosive` 的 CSS 斜杠在小字号下几乎不可见，语义也与省略混淆）。这两个问题是视觉体系里影响最大的短板。

---

## Part B — 问题清单

### 🔴 致命

---

**B-1：新用户打开页面，无引导就无法使用**

- **现象**：`analyze()` 在 Line 1709 检测到 key 为空时调用 `setStatus('请在 API 设置中填入 Key', true)`，但"API 设置"卡片默认折叠（`class="card-body collapsed"` + `class="toggle-arr closed"`，Line 357/355）。错误提示出现在 status 行，用户不知道要展开哪张卡片、去哪里填 Key。
- **Why**：首次使用的老师或学习者会在"输入句子 → 点分析 → 报错"这一步卡死，不知道下一步是什么。
- **Fix**：在 `analyze()` 的 `!cfg.key` 分支里，额外调用展开 API 设置卡的函数，并 `scrollIntoView` 到 `#api-key` 输入框。同时在 `#api-key` 空且页面加载时，在输入行下方显示一行淡色引导文字："先在 API 设置中填入 Key，再开始分析"。

---

**B-2：`stabilizeAnalysisData` 静默拦截 12 个测试句，用户无法观察真实 AI 输出**

- **现象**：Line 1587 起，函数对 12 个精确匹配的句子（"i love it"、"pick it up"、"get it"、"water"、"better"、"not that common" 等）直接返回硬编码 JS 对象，完全绕过 AI 返回值。
- **Why**：两个隐患：① 用户切换了 AI 服务商或模型，想验证其输出质量，但对这 12 个句子看到的永远是硬编码版本，无法调试。② 每次修改系统 Prompt 规则，必须手动同步更新 `stabilizeAnalysisData`，否则"参考"和"实际运行"悄然分裂。
- **Fix**：把硬编码输出移入 `TEST_FIXTURES` 常量对象，默认不启用。在 API 设置区加一个"测试模式"复选框（checkbox），勾选时才激活 fixture。正常用户路径完全不受影响。

---

**B-3：分析完成后 TTS 配置被折叠，但 Play 按钮仍然可见**

- **现象**：`collapseCards()`（Line 511）在分析成功的 Line 1720 被调用，折叠 `ex-body` 和 `api-body`。TTS 引擎选择器 `#tts-engine`（Line 395）和各引擎的 Key 输入框（Lines 408–480）都在 `api-body` 内，而播放按钮在 `result-wrap` 内（Line 341），两者属于不同卡片。
- **Why**：一个从未配置过 TTS 的新用户：看到播放按钮 → 点击 → 浏览器 TTS 播放（默认）→ 想换成 OpenAI TTS → 找不到入口。功能上 DOM 元素仍然存在（collapsed 只是 `max-height:0`），`togglePlay()` 能读到 `#tts-engine` 的值，所以功能不坏；但发现和修改 TTS 配置的路径对新用户不透明。
- **Fix**：把 TTS 引擎选择器（`#tts-engine` 所在的 `<div>`，Lines 393–406）从 `api-body` 迁移到 `result-wrap` 的 audio-section（Line 339）内，与播放按钮并排。引擎切换后的详细配置（Key 输入框等）仍可保留在 `api-body`，只是主选择器要始终可见。

---

### 🟡 显著

---

**B-4：点击示例句只填充输入框，不触发分析**

- **现象**：Line 963，`btn.onclick=()=>{ document.getElementById('s-input').value=item.text; }`，只填值，不调用 `analyze()`。
- **Why**：示例句的核心用途是"看这个句子的分析效果"，不是"复制句子"。每次点示例都需要额外点击"▶ 分析"。在移动端尤其差。
- **Fix**：改为 `btn.onclick=()=>{ document.getElementById('s-input').value=item.text; analyze(); }`。不需要担心用户未读就触发——如果分析太快，用户可以立刻看到结果，比多一次点击更流畅。

---

**B-5：`elision` 的 CSS 变量存在但未使用，`.elided-char` 和 `.word-el.reduced` 颜色完全相同**

- **现象**：Line 99：`.elided-char{color:#ccc;text-decoration:line-through;text-decoration-color:#ccc;}`。Line 98：`.word-el.reduced{color:#ccc;}`。两者都是硬编码 `#ccc`，视觉上完全相同。而 Line 11 已经定义了 `--c-elision:#B84040`（深红），但从未被 `.elided-char` 引用。
- **Why**：`elision`（某个字母被划掉）和 `reduction`（整个词变灰）是两种完全不同的现象，颜色相同意味着学习者无法靠视觉分辨。同时，`--c-elision` 已有正确值却没有被用上，这是渲染代码和 CSS 变量层脱节的问题。
- **Fix**：改 Line 99 为：`.elided-char{color:var(--c-elision);text-decoration:line-through;text-decoration-color:var(--c-elision);}` 同步更新 Line 118 图例：`.leg-s{text-decoration-color:var(--c-elision);color:var(--c-elision);}`. 改动共 2 行，CSS 变量已存在，无其他代价。

---

**B-6：SVG 曲线定位依赖双 RAF + 100ms setTimeout，是布局不稳定的未解决问题**

- **现象**：渲染完成后调用双层 `requestAnimationFrame(()=>requestAnimationFrame(()=>{ drawAllLocalCurves(); setTimeout(()=>drawAllLocalCurves(), 100); }))` 来触发曲线绘制（约在 Line 1776–1790 区域）；resize 事件处理同样使用双 RAF（约 Line 2407–2419）。
- **Why**：双 RAF 是"赌两帧足够让 flex 布局稳定"，100ms fallback 是"赌 100ms 足够"。这是 SVG `position:absolute` 依赖 `.sg-wrap-item` 宽度的布局循环依赖——曲线需要知道容器宽度，但容器宽度只在内容渲染后才稳定。两者互相等待，只能靠时间猜。
- **Fix**：用 `ResizeObserver` 观察 `#sg-row`，在 `callback` 触发（即 layout 实际稳定后）调用 `drawAllLocalCurves()`。这消除了 setTimeout 和帧数猜测，任何导致重排的操作（resize、窗口缩放、字号变化）都能正确触发重绘。

---

**B-7：`.card-body` 的 `max-height:1800px` 是动画 hack，有两个副作用**

- **现象**：Line 214（约）：`.card-body{max-height:1800px;transition:max-height .3s ease;}`，用 max-height 从 1800px 到 0 做展开/折叠动画。
- **Why**：① 若用户在 `#sp-ta` 粘贴超过 1800px 高度的自定义 Prompt，展开动画会在中途截断。② CSS transition 对完整的 1800px 范围做动画插值，实际内容只有 300px 时也在"动画一个 1800px 的距离"，低端设备上是无谓的 GPU 负担。
- **Fix**：展开时用 JS 读取 `element.scrollHeight` 并设为临时 `max-height`；动画结束后移除 `max-height` 约束（设为 `none`）；折叠时先重置为 `scrollHeight`，再下一帧设为 0。标准的"精确高度动画"模式，无魔法数字。

---

**B-8：图例（Legend）在分析结果下方，参考信息倒置**

- **现象**：FLOW/CUES 图例渲染在 `#sg-row` 下方（约 Line 312–333），Translation 上方。
- **Why**：学习者看着 `A‿B`、灰色字、删除线时，需要向下滚动查图例，然后滚回来对照——认知负担不合理。图例是解码器，应该在编码对象的旁边或上方。
- **Fix**：把 Legend 移到 `result-header`（视图切换/口音选择那一排）的下方，让图例和分析区在同一视线内。或者给每个 legend item 做 tooltip，悬停时显示完整解释。

---

**B-9：TTS 播放和视觉标注之间没有任何联动**

- **现象**：`togglePlay()` 调用 `speechSynthesis.speak(u)` 或 HTTP TTS，没有 `boundary` 事件监听，没有当前播放单词/意群的高亮。
- **Why**：工具的核心教学假设是"视觉 + 听觉同步"，但实现上两者完全解耦。用户在听的时候要自己找到对应的标注位置，正是需要被工具替代的认知任务。
- **Fix**：对 browser SpeechSynthesis：`SpeechSynthesisUtterance` 的 `boundary` 事件提供 char offset，可以映射到当前意群。MVP 方案：播放时给当前意群的 `.sg-wrap-item` 加 `.playing` class，CSS 加浅蓝背景。对 OpenAI TTS / ElevenLabs：返回 audio blob，通过 WebAudio `AudioContext` 的 `currentTime` 做时间轴映射。

---

**B-10：`looksLikeLegacyDefaultPrompt` 可能静默覆盖用户的自定义 Prompt**

- **现象**：Line 894，函数用 5 个条件检测旧版 Prompt，4 个条件基于同一个前缀字符串（"You generate structured teaching annotations for an English connected-speech visualizer."）。
- **Why**：如果用户以此前缀开头写了自定义 Prompt 但删掉了"flapping"相关规则（比如专注英式英语的用户），条件 `!prompt.includes('flapping')`（约 Line 909）成立，整个自定义 Prompt 在版本更新时被静默替换，无任何提示。
- **Fix**：自定义 Prompt 保存时，在 localStorage 里额外写入一个 `pa_prompt_is_custom:true` 标记；`looksLikeLegacyDefaultPrompt` 运行前先检查此标记，为 true 则跳过覆盖。同时在覆盖发生时调用 `setStatus` 显示："已将提示词升级至 v14.xx 版本。如需恢复自定义版本，请手动粘贴。"

---

**B-11：`populateELSelect` 中存在废代码，有一次多余的 DOM 操作**

- **现象**：Line 2195–2196 附近：创建 `g1` optgroup 但从未 `appendChild`；`addOpt` 把 option 直接添加到 `sel`，紧接着 `sel.innerHTML=''` 立刻清空。后续重新创建 `grp` 并正确添加。最终结果正确，但有两次无效的 DOM 写入。
- **Why**：这是一次不完整重构的残留。不影响功能，但 `addOpt` 函数只在废弃路径中被调用，是死代码。
- **Fix**：删除 `g1` 声明、第一个 `recs.forEach(addOpt)` 循环、`sel.innerHTML=''` 这三行，以及 `addOpt` 函数定义。

---

**B-12：`wordHasSchwaCue` 对 content words 的判断恒为 true**

- **现象**：Line 1163：`if(SCHWA_CONTENT_WORDS.has(key)) return textHasSchwa(source) || ['about','support','today','again','around','away'].includes(key);`。进入这个 if 分支的前提是 `SCHWA_CONTENT_WORDS.has(key)` 为 true，而 `SCHWA_CONTENT_WORDS` 就是那 6 个词。所以 `includes(key)` 恒为 true，`textHasSchwa(source)` 永远被短路。
- **Why**：这 6 个词会无条件显示 schwa cue，不管 IPA 里是否真的有 /ə/。本意是"只有当 IPA 中确实有 schwa 时才显示"，但短路逻辑写反了。
- **Fix**：Line 1163 改为：`if(SCHWA_CONTENT_WORDS.has(key)) return textHasSchwa(source);`，删掉后半段 `||includes(key)` 即可。

---

**B-13 [R2]：`lost_plosive` 的 CSS 斜杠不可见且语义错误，应改为 ˺ 字符**

- **现象**：Line 101：`.lost-plosive-char::after{content:"";width:.42em;height:1px;background:var(--c-lost-plosive);transform:rotate(-58deg);}` 用一个 0.42em × 1px 的旋转线条模拟失爆标记。
- **Why**：两个问题叠加：① 可见性：`--fs` 降到 13px 时，这条线只有约 5px 宽 × 1px 高，屏幕上几乎不可见。② 语义：斜线在视觉上和 `elision` 的删除线太像——学习者容易把"音被卡住了"看成"音被划掉了"，恰好是需要严格区分的两件事。参考资料（Round 2）使用 ˺（Unicode U+02FA，MODIFIER LETTER UNASPIRATED），在爆破音字母右上角显示，语义清晰（"封闭"/"卡住"），纯 Unicode 渲染，不依赖 CSS 伪元素。
- **Fix**：删除 `.lost-plosive-char::after` 的 CSS 规则（Line 101–102）；在 `renderSegment` 函数的 `lost_plosive` 字符渲染路径中，紧跟爆破音字母后插入 `<sup style="font-size:.58em;color:var(--c-lost-plosive);vertical-align:.35em;line-height:0;">˺</sup>`；同步更新图例 `.leg-lp` 的 HTML 内容。**技术代价：需要改渲染逻辑**（renderSegment 函数，约 10-15 行）+ 删除 CSS 伪元素规则 + 图例更新。

---

**B-14 [R2]：H 击穿（H-dropping）高频场景未覆盖**

- **现象**：当前系统 Prompt 没有对 he/him/her/his/have/has 在辅音结尾词后的 H 脱落做专项规则。"call him → /kɔːlɪm/"、"tell her → /telɜː/"这类场景可能落入 `reduction`（him 弱化为 /ɪm/），但不会同时呈现随之产生的 liaison 视觉。
- **Why**：H 击穿是极高频的听力误解场景——"I told him → I toldim"，学习者听不出"him"在哪里。参考资料将其列为独立教学概念（同化 3.1 下的子类），覆盖频率支持在产品中明确处理。
- **Fix**：在 `DEFAULT_SP`（Line 602 后的字符串）的 PHENOMENA ENUM 部分，在 `assimilation` 的说明下追加规则（约 5-8 行）：当辅音结尾词后紧跟非重读的 he/him/her/his/have/has 时，H 脱落，将该词组整体标为 `assimilation`，`naturalIpa` 显示 H-dropped 形式。同步更新 `PROMPT_VERSION`（Line 602）触发自动升级。**技术代价：只改 Prompt 和版本号，不改前端渲染**。

---

### 🟢 建议

---

**B-15：`<html lang="zh">` 与英文内容分析不匹配**

- **现象**：Line 2，整个页面声明为中文。`startSpeech()` 里手动设置了 `u.lang='en-US'/'en-GB'`（约 Line 2104），TTS 功能不受影响；但分析结果中的英文单词节点没有 `lang="en"` 属性，屏幕阅读器会用中文语音尝试朗读英文标注。
- **Fix**：在 `renderSegment` 的 `.word-el` 创建时加 `el.setAttribute('lang','en')`，或在 `#sg-row` 上统一加 `lang="en"`。

---

**B-16：点击"▶ 分析"时旧分析结果在新结果加载前仍可见**

- **现象**：`analyze()` 开始时不清空 `#sg-row`，旧的标注内容在新分析完成前一直显示，状态行显示"正在分析…"但视觉上没有过期提示。
- **Fix**：`analyze()` 开始时给 `#result-wrap` 或 `#sg-row` 加 `.stale` class，CSS 设 `opacity:.35;pointer-events:none`。分析成功后移除。

---

**B-17：颜色是 4 种现象的唯一区分方式（色盲问题）**

- **现象**：`assimilation`（紫色）、`contraction`（橄榄绿）、`flapping`（橙红）、`intrusive_r`（棕黄）四种现象仅靠颜色区分，没有形状伙伴。`flapping` 在 IPA 行有 ɾ 符号可作形状参照，但 `assimilation` 和 `contraction` 是纯色编码。
- **Fix**：短期：在图例中为每个现象添加简短字母标签（如 `A` 同化 / `C` 缩约）作为颜色的文字伴侣。长期：[R2 建议] 实现"无障碍模式"开关，开启后在 `reduction` 词上显示 ° 上标，参考资料中的形状伙伴方案。

---

**B-18：`STORE.fields` 里有一个幽灵 key**

- **现象**：Line 854 附近：`'prompt-version': 'pa_prompt_version'`，但 DOM 里没有 `id="prompt-version"` 的元素。`bindAll()` 里的 `document.getElementById` 返回 null 后静默跳过，功能不受影响（`pa_prompt_version` 的读写通过 `localStorage.setItem` 直接调用，Lines 923/988），但这是一个误导性的死引用。
- **Fix**：删除 `STORE.fields` 中的 `'prompt-version'` 条目，一行改动。

---

**B-19 [R2]：`intrusive_r` 应降为可选层，避免占用 AI 输出配额**

- **现象**：当前 `intrusive_r` 是 P2 现象（虚线下划线，与 `lost_plosive`/`flapping` 同级），但"数据少"已经是代码层面的信号——AI 对英式口音下的 V+V 插入 /r/ 场景的输出频率很低且不稳定。
- **Why**：受众窄（仅英式口音学习者）+ AI 输出稀少，占据 Prompt 规则篇幅不合算。降级不会影响核心学习者。
- **Fix**：在 Prompt 里将 `intrusive_r` 的说明移至"Optional / British-accent only"部分，并将其触发条件收紧（仅当输入句明确触发英式口音时才标注）。前端渲染保持不变，只改 Prompt 权重。

---

**B-20 [R2]：元元连读（V+V + W/J 过渡音）可条件性加入**

- **现象**：当前系统没有处理"do it → /duːwɪt/"、"see it → /siːjɪt/"等 V+V 过渡音，这是 `liaison` 未覆盖的场景（`liaison` 只处理辅元 C+V 边界）。参考资料将元元连读列为独立的粘连子类（2.2），使用 ͡w/͡j 符号。
- **建议**：不立即加入。先用 10 个 V+V 测试句验证当前 AI 对该现象的识别稳定性（标准：漏标率 < 30%）。若通过，以 `v_glide` 枚举值加入，使用 ͡w/͡j 符号（与 `liaison` 共享蓝色），归入补充层。**技术代价：需要改 Prompt + renderSegment + 新增枚举**。

---

## Part C — 技术评估

**架构评分：2/5**

2422 行单文件，无模块化，所有函数在全局作用域（`mergeLiaisonChains`、`splitOvermergedLiaisonSegment`、`stabilizeAnalysisData`、`normalizeAnalysisData`、`fixContrastSenseGroupBoundaries`、`smoothPitchBoundaries` 等几十个函数平铺）。CSS 变量命名空间良好（`--c-liaison`、`--fs`、`--sg-gap` 等），是单文件项目里少见的清晰设计。但当前规模是维护临界点，再加 300 行就会失控。

**健壮性评分：3/5**

normalization pipeline（`normalizeAnalysisData → stabilizeAnalysisData → fixContrastSenseGroupBoundaries → smoothPitchBoundaries`）层次分明，是思路清晰的防御设计。但四层后处理会遮蔽 AI 输出质量变化，难以诊断是"AI 退步了"还是"兜底修复了"。`stabilizeAnalysisData` 的 oracle 设计让这个问题更严重。SVG 双 RAF + 100ms 是布局不稳定的症状而非解法。

**Prompt 架构评分（Round 2 新增）：4/5**

CUES 层（ə/•/|）完全前端派生、不进入 AI JSON 的设计是正确决策——既保证了 AI 输出的稳定性，又留出了前端迭代的空间。`PROMPT_VERSION` + `looksLikeLegacyDefaultPrompt` 的自动升级机制思路正确，但实现有 edge case（B-10）。

**最让人担心的 3 个技术决定：**

1. **`stabilizeAnalysisData` 的 oracle 设计**（B-2）：12 个句子的硬编码输出随 Prompt 迭代越来越难同步，同时屏蔽了对 AI 真实表现的观测。这是当前最高优先级的技术债。

2. **双 RAF + 100ms setTimeout**（B-6）：是布局依赖循环的贴片方案，不是解决方案。每次 layout 条件变化（不同字体、不同屏幕、浏览器更新）都可能复现曲线渲染失败。

3. **`collapseCards()` 在分析成功后折叠 TTS 配置**（B-3）：这个交互设计在用户没有预配置 TTS 时会产生严重困惑，且对新用户的惩罚程度最高。

---

## Part D — 优先级路线图（4 周，R1+R2 综合版）

### 第 1 周：消除上手摩擦 + 零代价 bug 修复

**优先级依据**：这些改动成本极低但影响首次使用的成功率。

| 任务 | 来源 | 技术代价 |
|------|------|---------|
| 无 Key 时展开 API 卡片并滚动到输入框 | B-1 | `analyze()` + 3 行 JS |
| 示例句点击触发分析 | B-4 | Line 963，1 行改动 |
| **`elision` 颜色 bug 修复**（`#ccc` → `var(--c-elision)`） | B-5 | Lines 99 + 118，各 1 行 |
| 分析加载时旧结果加 `.stale` 半透明 | B-16 | CSS + 2 行 JS |
| 删除 `STORE.fields` 幽灵 key | B-18 | 1 行 |

---

### 第 2 周：视觉体系修复 + TTS 可用性

| 任务 | 来源 | 技术代价 |
|------|------|---------|
| TTS 引擎选择器迁移到 `result-wrap`，脱离折叠卡片 | B-3 | HTML 结构调整 + CSS |
| **`lost_plosive` 符号改为 ˺ 字符**（替换 CSS 伪元素） | B-13 | renderSegment 约 15 行 + 删除 CSS |
| TTS 播放时当前意群 highlight（browser SpeechSynthesis `boundary` 事件） | B-9 | JS 事件监听 + CSS `.playing` class |
| 图例移到分析区上方 | B-8 | HTML 重排 |
| `wordHasSchwaCue` bug 修复 | B-12 | Line 1163，1 行 |

---

### 第 3 周：架构稳定化 + Prompt 扩充

| 任务 | 来源 | 技术代价 |
|------|------|---------|
| `ResizeObserver` 替换双 RAF + 100ms | B-6 | `drawAllLocalCurves` 调用方重构 |
| `stabilizeAnalysisData` → `TEST_FIXTURES` 模式 | B-2 | 重构 + 新增 UI 开关 |
| **H 击穿 Prompt 扩充 + PROMPT_VERSION 更新** | B-14 | 改 DEFAULT_SP + Line 602 |
| `looksLikeLegacyDefaultPrompt` 加 custom 标记保护 | B-10 | localStorage key + setStatus 通知 |
| `max-height:1800px` 改为 scrollHeight 动画 | B-7 | JS expand/collapse 逻辑重写 |
| `populateELSelect` 废代码清理 | B-11 | Line 2195–2196 删除 + addOpt 删除 |

---

### 第 4 周：可访问性 + 体系扩展

| 任务 | 来源 | 技术代价 |
|------|------|---------|
| `.word-el` 加 `lang="en"` 属性 | B-15 | renderSegment 1 行 |
| 无障碍模式开关（reduction 词加 ° 上标） | B-17 | CSS + JS 开关 |
| 375px 移动端全流程测试 + overflow 修复 | B-1 附带 | 测试 + CSS 调整 |
| `intrusive_r` Prompt 权重降级 | B-19 | 改 DEFAULT_SP |
| **元元连读 AI 稳定性测试**（决定是否加入 `v_glide` 枚举） | B-20 | 先测试，再决定是否实现 |

---

## Part E — 音变体系 & 符号系统方案（Round 2 结论）

> 本部分是 Round 2 的可操作结论摘要。完整分析见 Round 2 审查报告。

### E-1：优化后三级梯队

| 层级 | 现象 | 枚举值 | 入选理由 |
|------|------|--------|---------|
| **核心层** | 辅元连读 | `liaison` | 最高频，‿ 符号零学习成本 |
| **核心层** | 弱化 | `reduction` | 功能词全面弱化，不掌握听不懂任何句子 |
| **核心层** | 失去爆破 | `lost_plosive` | "明明认识这个词但听不出来"的最高频原因 |
| **核心层** | 美式闪音 | `flapping` | 参考资料漏掉，但美式英语核心音变，必须保留 |
| **补充层** | 辅辅吞音 | `elision` | 跨词 C 删除，高频，删除线直觉强 |
| **补充层** | 腭化同化（含 H 击穿） | `assimilation` | did you/would you 类高频；H 击穿通过 Prompt 扩充覆盖 |
| **补充层** | 缩约 | `contraction` | 帮助学习者建立书面形和口语形的连接 |
| **补充层** | 英式 r 连读 | `liaison_r` | 英式口音学习者核心现象 |
| **补充层** | 元元连读 | （条件性 `v_glide`） | 先做 AI 稳定性测试，通过后加入 |
| **可选层** | 英式闯入 r | `intrusive_r` | 受众窄，AI 输出少，降级处理 |
| **可选层** | 逆向同化（n+k→ŋ） | （扩充 `assimilation` Prompt） | 不独立成枚举，加入 assimilation 覆盖范围 |

### E-2：符号最终方案

| 现象 | 当前符号 | 最终推荐 | 改动 |
|------|---------|---------|------|
| `liaison` | ‿ 蓝色，边界加粗，IPA | 不变 | — |
| `reduction` | 灰色 `#ccc`（实为 bug，应为 `#bbb`） | 灰色词面 `var(--c-reduction)`；无障碍模式加 ° | CSS 修正 |
| `lost_plosive` | CSS 伪元素斜杠（0.42em×1px） | **˺ 字符右上标**，颜色 `var(--c-lost-plosive)` | 改渲染逻辑 |
| `elision` | `#ccc` 删除线（bug：同 reduction） | `var(--c-elision)` 即 `#B84040` 删除线 | 改 CSS 2 行 |
| `flapping` | 橙红色词面 + IPA 显示 ɾ | 不变（ɾ 是形状伙伴） | — |
| `assimilation` | 紫色词面 + IPA 显示变化结果 | 不变；扩充覆盖 H 击穿场景 | 改 Prompt |
| `contraction` | 橄榄绿 + 实线下划线 | 不变 | — |
| `intrusive_r` | 棕黄色，idea‿r‿of | 不变；降为可选层 | 改 Prompt 权重 |
| `liaison_r` | 同 liaison 蓝色 + IPA 区分 | 不变 | — |

### E-3：5 条设计原则（新增现象时的决策依据）

1. **形义一致**：符号的视觉形态必须暗示音变性质。‿ = 滑连；删除线 = 删除；˺ = 封闭/卡住。不用与含义相反的符号（斜杠既像"划掉"又像"失爆"，是语义污染）。
2. **颜色是类别索引，形状是含义载体**：颜色分类，但不能是唯一信息通道。每种现象必须有形状伙伴（符号、装饰线、IPA 变化）。
3. **同类现象共享颜色，用符号细分**：连读类共享蓝色（liaison/liaison_r/v_glide）；弱化类分开（reduction 灰 vs contraction 绿）；同化类共享紫色（assimilation + H 击穿）。
4. **IPA 层是解释，不是装饰**：`naturalIpa` 只在有变化时显示。空 IPA 优于错误 IPA。reduction 的 IPA 字号最小，形成层级感。
5. **AI 稳定性决定是否成为一等公民**：temperature=0 下仍不稳定的现象，不进入核心层。每次加新现象先用 10 个测试句验证，通过再上线。

---

*综合版报告终 | Round 1（代码/UX）+ Round 2（体系/符号）*
