# 项目协作规则

## 基本约定

- 默认使用中文说明；代码、变量名、命令保持英文。
- 本项目当前是纯单文件 HTML 工具，无 npm、无构建流程、无外部依赖。
- 约 2422 行。纯 HTML+CSS+JS，零框架
- 当前开发主线从最新 `phonetics_v*.html` 继续；稳定后再封 `v15`。
- 版本递增规则：新增功能、规则体系变化、数据结构变化时，才新建下一个小版本。
- 如果是在当前版本之后做该版本的优化、显示修正、文案调整、样式微调或 bug 修复，且没有添加新功能，直接在当前版本文件上修改，不再新建下一个版本。
- 旧版本文件保留为回退基线；只有开新功能版本时才复制新文件，不为了纯优化复制新文件。


## 文档习惯

以后涉及本项目或新项目，都必须至少维护两类 Markdown 文档：

- `PROJECT.md`：项目说明、核心价值、运行方式、关键规则、当前状态。
- `CHANGELOG.md`：版本更迭、bug 修复、验证结果、遗留问题。

如果是新项目，先建规则和文档，再开始实现；如果是已有项目，先读当前文档再动代码。

## 关键红线

- 不要把所有语音现象拆到最碎；本工具优先服务“学习者听得懂、看得见”的自然语流块。
- 不要让 liaison chain 无限延长；默认 2-3 个词，长句优先清晰小块。
- `the + vowel` 暂不作为普通连读链显示，例如 `the oven` 保持视觉普通，读法由 IPA 体现。
- `an + silent-h word` 优先形成小链，例如 `an‿hour -> /ə naʊər/`。
- 不要把 `lost_plosive / assimilation / elision / intrusive_r` 混进 liaison chain。
- `sg-pitch-spacer` 高度必须与 SVG 高度保持一致，当前为 48px。
- SVG 语调曲线必须使用 `position:absolute`，不要改回参与 flex 布局。
- SVG class 必须用 `setAttribute('class', ...)`，不要用普通 `className`。

## 关键位置
- CSS 变量：文件顶部 <style> 标签，约 Line 10-25
- `analyze()`：搜索 "async function analyze"
- `renderSegment`：搜索 "function renderSegment"
- `DEFAULT_SP`：Line 606 往下
- `PROMPT_VERSION`：Line 602
- `stabilizeAnalysisData`：Line 1587
- `wordHasSchwaCue`：Line 1158

## 改动原则
- 精确改动，不重构无关代码
- 改动前核对行号内容与描述是否一致
- 保留所有现有 id、class 属性


## 本地运行

在项目根目录启动本地服务器：

```powershell
python -m http.server 8765 --bind 127.0.0.1
```

访问最新版本，例如：

```text
http://127.0.0.1:8765/phonetics_v14.19.html
```

不要直接双击 HTML 用 `file://` 打开，否则外部 API 请求容易被 CORS 拦截。

## GitHub 仓库

- 仓库地址：`https://github.com/Schemingboy/phonetics_analyzer`
- 已推送文件：`index.html`（v14.15）、`phonetics_v14.15.html`、`README.md`、`LICENSE`、`.gitignore`
- 以下文件**仅本地保留，不推 GitHub**：`PROJECT.md`、`CHANGELOG.md`、`AGENTS.md`、`HANDOFF.md`、`review-brief.md`、`review-round2-brief.md`、`old version/`、旧版 `phonetics_v*.html` 文件
- GitHub 操作统一用 `gh` CLI（已安装 `C:\Program Files\GitHub CLI\gh.exe`），已通过 `gh auth login` 认证
- 禁止在 git remote URL 或任何地方粘贴明文 GitHub token
