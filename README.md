# 英语语音语调可视化工具

输入英文句子，AI 自动分析语音现象并可视化渲染——连读、弱化、同化、省略、失去爆破、闪音、缩约、意群停顿、语调曲线、情绪色条，支持 TTS 朗读。

核心价值不是"查字典音标"，而是帮英语学习者**看见自然语流里哪里发生了音变**、听起来怎么连、朗读时如何对应视觉标注。

## 功能

- **语流分析**：自动识别 liaison（连读）、reduction（弱化）、assimilation（同化）、elision（省略）、lost plosive（失去爆破）、flapping（美式闪音）、contraction（文本已写出的口语缩约）、intrusive-r（英式闯入 r）
- **语调曲线**：每个意群上方绘制 SVG 语调曲线
- **意群提示**：用轻量 pause cue 显示自然 spoken chunk / sense group 边界
- **情绪色条**：显示语气能量和极性
- **TTS 朗读**：支持浏览器 SpeechSynthesis / OpenAI TTS / ElevenLabs / Typecast
- **多 AI 后端**：Anthropic / OpenAI / DeepSeek / Groq / 自定义 OpenAI 兼容接口
- **纯单文件**：无框架、无构建工具、无外部依赖

## 使用方法

在项目目录启动本地服务器：

```powershell
python -m http.server 8765 --bind 127.0.0.1
```

浏览器访问：

```
http://127.0.0.1:8765/index.html
```

当前 GitHub 公开入口：

```
http://127.0.0.1:8765/index.html
```

当前公开入口 `index.html` 保留 v14.15。后续版本和本地工作资料不进入 GitHub 公开仓库。

> 不要直接双击 HTML 用 `file://` 打开，否则 API 请求会被 CORS 拦截。

首次使用需要配置 API Key：页面内点击 **设置**，填入对应服务的 Key 即可。

## 技术栈

- 单文件 HTML，原生 DOM + SVG 渲染
- AI 分析：调用各平台 Chat Completions API，返回结构化 JSON
- 设置存储于 `localStorage`

## 项目结构

```
.
├── index.html              ← GitHub Pages / 公开入口（v14.15）
├── README.md
├── LICENSE
└── .gitignore
```

## License

MIT
