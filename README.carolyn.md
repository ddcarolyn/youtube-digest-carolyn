# YouTube Digest — Carolyn 二创版

> 分支 `carolyn` · fork 自 [zarazhangrui/youtube-digest](https://github.com/zarazhangrui/youtube-digest) v1.2.0 · MIT
> `main` 分支是上游原版镜像,不做任何改动;所有二创都在 `carolyn` 分支上。

## 为什么要改

上游的 Overview 只有 **Chapters + Key Quotes**,两样都是按时间轴铺开的。
想知道「这视频到底在讲什么、我看完能拿走什么」,还是得从头扫一遍章节。

所以加了一块**论文式 Executive Summary**,放在 Overview 最上面,四行读完就能决定要不要花这个时间。

## 改了什么

### 1. Executive Summary(新增)

Overview 顶部新增一块,四段固定结构:

| 字段 | 上限 | 回答什么 |
|---|---|---|
| `oneLine` | 1 句 ≤25 词 | **讲了啥** —— 视频的主张,不是主题。"怎么招人"是主题,"招聘该按 executive search 做而不是漏斗"才是主张 |
| `framework` | 2–4 条,每条 ≤12 词 | 作者思考的骨架 —— 心智模型 / 阶段 / 拿来比较的轴。不是流水账复述 |
| `takeaway` | 1–3 条,每条 ≤15 词 | **你能收获啥** —— 看完能做或能决定什么不一样的事 |
| `verdict` | ≤25 词 | 花几分钟 + 跳到哪个时间戳 |

**takeaway 防注水规则**:每条必须带「一个数字 / 一个具名方法 / 一个被推翻的假设」之一。
三样都没有就是废话,**宁可少写一条也不许凑数**。视频真的没有可操作的东西,就老实写一条说没有。

### 2. 反废话 prompt 规则

`prompts/analysis.md` 里加了硬约束:

- **禁开场白**:`This video discusses`、`In this video the speaker`、`The video covers`
- **禁填充连接词**:`It's worth noting that`、`Overall`、`In today's world`、`At the end of the day`
- **禁空强化词**:`very`、`really`、`incredibly`、`a lot of`
- **禁虚夸**:`valuable insights`、`great advice`、`deep dive`、`practical tips`
- **禁复述标题**,禁在作者明确给了数字时还含糊其辞

**发送前自检**(照搬 [ayghri/i-have-adhd](https://github.com/ayghri/i-have-adhd) 的 pre-send check):
删掉不带信息的模糊副词、删掉成语和比喻、删掉预告式开场。

**终测**:只读 `oneLine` 和 `verdict`,能不能判断出(a)这视频在主张什么、(b)要不要花这个时间?任一为否,两段都重写。

**章节摘要口径也改了**:说这段"得出了什么结论",不许说这段"讨论了什么"。

`verdict` 必须给**具体分钟数**和**一个具体时间戳**(来自 i-have-adhd 的规则 6 和规则 3):
不是"值得扫一遍",而是"6 分钟扫完;跳到 12:30 看 sourcing 方法"。

### 3. Overview 导航

Chapters 分段多了就很长,所以加了:

- **顶部 sticky 跳转目录**:`Summary / Chapters / Quotes`,点一下平滑滚过去。
  目标段如果是收起状态会**自动展开**,否则跳过去看到空标题像是没反应。
- **Chapters / Key Quotes 可点折叠**,带箭头指示。折叠状态存 `chrome.storage.local`,下次打开保持上次的样子。
- Executive Summary 不给折叠 —— 它就四行,而且是最该先看的四行。

### 4. analysis 缓存版本化(踩坑修复)

**坑**:改完 Overview 结构、重载扩展,界面纹丝不动。

**根因**:`analysis` 缓存在 `chrome.storage.local` 里。重载扩展换的是代码,缓存原样还在,
新渲染函数拿着**旧结构**去画,`execSummary` 字段不存在,整块被判定为空然后隐藏。
表现就是「插件明明更新了但没效果」。

**修法**:

```js
const ANALYSIS_SCHEMA_VERSION = 2;   // 改 Overview 结构就 +1
```

`saveToCache` 写进去,`loadFromCache` 里版本对不上就把 `analysis` 置 null。

**关键取舍:只丢 analysis,保留字幕和翻译缓存。**
字幕花过 Supadata credit(免费版每月 100 个,1 视频 1 credit),analysis 让 DeepSeek 重算是一分钱量级。
丢贵的留便宜的就搞反了。

> ⚠️ **以后再改 Overview 结构,记得把版本号 +1**,否则同样的坑会再踩一次。

## 文件改动

| 文件 | 改了什么 |
|---|---|
| `prompts/analysis.md` | Executive Summary 输出规格 + 字数上限 + 禁用词清单 + 发送前自检 |
| `background.js` | 白名单里加 `execSummary`,按 `safeString`/`safeList` 重建;加 schema 版本号 |
| `sidepanel.html` | Executive Summary 容器 + 跳转目录 + 可折叠标题 |
| `sidepanel.js` | 渲染 Executive Summary、跳转/折叠逻辑、缓存版本校验 |
| `sidepanel.css` | 对应样式,复用上游已有的 CSS 变量 |

合计 5 个文件、376 行新增。**所有改动块都带 `/* CAROLYN */` 注释**,`grep -rn "CAROLYN"` 可定位。

## 语言:源文本保持英文

Executive Summary 没让模型直接吐中文,而是走和 Chapters、Quotes 同一套翻译层。
好处是顶部那三个按钮(Original / 中文 / 双语)对它一样生效、共享缓存。
而且它的 segments 排在翻译队列**最前面** —— 渐进翻译时,总结比章节先出来。

## 安装

```bash
git clone https://github.com/ddcarolyn/youtube-digest.git
cd youtube-digest && git checkout carolyn
```

然后 `chrome://extensions` → 开启开发者模式 → 加载已解压 → 选这个文件夹。
API key 的申请和填写见上游 [README.zh-CN.md](README.zh-CN.md)。

> API key 存在 `chrome.storage.local`,**不跨 Chrome profile、不跨设备同步**,每台设备/每个 profile 都要重填一次。

## 改完怎么生效

1. `chrome://extensions` → 找到 YouTube Digest → 点**重新加载**
2. 回 YouTube 标签页按 **Cmd+R** 刷新(不刷新的话侧边栏还是旧代码)
3. 打开 Overview

## 跟上游同步

```bash
git checkout main && git pull origin main   # origin = 上游
git checkout carolyn && git rebase main
npm test && npm run check
```

冲突大概率只在 `prompts/analysis.md` 和 `background.js` 的返回语句那一行。

三个 remote 的分工:

```
origin = zarazhangrui/youtube-digest   # 上游,只 pull
mine   = ddcarolyn/youtube-digest      # 自己的,push 这里
```

## 测试

```bash
npm test     # 上游 62 个测试,二创后仍 62/62 pass
npm run check # 发布文件白名单校验
```

上游测试没有断言 analysis 的返回结构,所以加 `execSummary` 字段不会打破它们。

## 致谢

- 上游:[zarazhangrui/youtube-digest](https://github.com/zarazhangrui/youtube-digest)(MIT)
- 反废话规则来源:[ayghri/i-have-adhd](https://github.com/ayghri/i-have-adhd)(MIT)
