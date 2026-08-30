# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 仓库概览

`F:/claude/taxi` 是一个**单文件静态网页** —— 深圳市网约车驾驶员从业资格考试模拟系统（`shenzhen_wyc_exam.html`）。整个项目只有一个文件，无构建工具、无外部依赖、无后端、无测试套件。

**目标用户**：准备参加深圳网约车从业资格考试的驾驶员。
**内容范围**：全国公共科目 + 深圳区域科目；题型含单选 / 多选 / 判断。

## 运行方式

直接双击或在浏览器中打开 `shenzhen_wyc_exam.html`。也可以本地起一个简易静态服务（任选其一）：

```bash
# 任选一种
python -m http.server 8080
# 或
npx serve .
```

无 `npm install`、无 `pip install`、无构建步骤 —— 整个应用就是这一个 HTML 文件。

## 文件结构（单文件内部布局）

`shenzhen_wyc_exam.html` 自上而下分四段：

| 行号范围 | 内容 | 说明 |
|---|---|---|
| 1–553 | `<style>` 内联 CSS | 使用 CSS 变量（`--primary` 等），响应式网格布局，`max-width: 900px` 时侧栏上移 |
| 554–646 | `<body>` 四个屏幕 DOM | `setup-screen` / `exam-screen` / `result-screen` / `review-screen` + 侧栏答题卡（`palette-grid`）|
| 731–5970 | `const QUESTIONS = [...]` | 题库数组，**498 道题**（原版 135 + 新增 363）。**这是整个项目最重要的"数据"部分** |
| 5987–6527 | 应用脚本 | 状态机、模式切换、练习、考试、回顾 |

### 题库数据结构

```js
{
  id: 86,                                          // 1..498
  type: 'single' | 'multiple' | 'judge',
  category: '全国公共科目' | '深圳区域科目',
  topic: '政策法律法规' | '服务规范' | '安全运营' | '应急处置'
        | '职业道德' | '车辆与设备' | '危险化学品' | '深圳地方',
  set: '官方基础题库' | '全国公共科目专项题库' | '深圳2026网约车考试题库' | ...,
  text: '题干...',
  options: [{ key: 'A' | 'B' | 'C' | 'D', text: '...' }],
  answer: ['A'] | ['A','C'] | ...                  // 多选用数组
  explanation: '解析...'
}
```

`set` 字段标识题所属的"试卷套题"。原版 135 题没有该字段，代码中通过 `q.set || '官方基础题库'` 兜底。共 **9 个套题**：

| 套题名 | 题目数 | 来源 |
|---|---|---|
| 📘 官方基础题库 | 135 | 原版（兜底字段） |
| 🌐 全国公共科目专项题库 | 46 | m116936 |
| 🌐 全国网约车题库大全 | 50 | m160124 |
| 🌐 网约车考试题库下载 | 49 | m123155 |
| 🌐 从业资格考试题库答案 | 47 | m154085 |
| 🚖 深圳网约车模拟考试题库 | 41 | m1298 |
| 🚖 深圳区域考试题 | 44 | m147454 |
| 🚖 深圳网约车科目一题库 | 49 | m174310 |
| 🚖 深圳2026网约车考试题库 | 37 | m167970 |

题目按主题分组（早期题有 `// ============ 章节名 ============` 注释分段，新增 363 题按 `set` 字段聚类，无章节注释）。

## 应用状态与模式

应用的核心是一个全局对象 `STATE`（约 2256 行）和四个互斥模式：

| 模式 | 触发 | 关键行为 |
|---|---|---|
| `setup` | 初始 / `restart()` | 显示筛选表单：科目 / 题型 / 分类 / 时长（30/50/60/90 分钟，仅考试生效）|
| `practice` | 「开始练习」 | 用户点选选项后**不立即判定**；须点击题目下方的「✓ 确认本题」按钮才显示对错与解析；无计时；无评分汇总 |
| `exam` | 「开始模拟考试」 | 倒计时（`startTimer`）；最后 60s 变黄、最后 30s 变红；时间到自动提交 |
| `review` | 「题目回顾」 | 答题后可进入，显示每题的对错、正确答案、解析；不显示侧栏统计 |

`switchMode(mode)` 是模式切换的唯一切换点，会清理 `STATE.timerInterval` 并切换 `hidden` 类。

### 评分规则（`getScoreForQuestion`）

| 题型 | 全对 | 少选 | 多选/错选 | 未答 |
|---|---|---|---|---|
| 单选 / 判断 | 1 分 | — | 0 分 | 0 分 |
| 多选 | 1 分 | **0.5 分** | 0 分 | 0 分 |

通过线：百分制 **≥ 80%**（`submitExam` 中 `const pass = percentage >= 80;`）。

## 侧栏答题卡（palette）

`renderPalette()` 为每题生成一个色块：
- 当前题 = 实心蓝
- 已作答 = 浅蓝边框
- 已标记（`STATE.flagged`）= 黄边框
- 未作答 = 白色

色块点击可跳转到任意题。

## 开发与维护要点

### 添加 / 修改题目

直接编辑 `QUESTIONS` 数组（行 731–5970）：
- **新题**：复制最近的同 `type` 题目作为模板，确保 `id` 不重复，`answer` 与 `options` 的 `key` 对应
- **新题必须带 `set` 字段**：标识它属于哪一套试卷。若新增全新套题，还需在 `#set-filter` 区块（行 620–630）加上对应 checkbox
- **改题**：注意 `answer` 数组的顺序不影响比对（`arraysEqual` 会先排序）
- **多选题 `key`**：固定使用 `'A' / 'B' / 'C' / 'D'`；多于 4 个选项时需修改 `getFilteredQuestions` 之外无其它约束
- **新增 topic**：必须在 `<body>` 的 `topic-filter`（行 605–614）也加上对应 checkbox，否则用户无法勾选
- **新增套题**：除在 `#set-filter` 增加 checkbox 外，建议同时在 CLAUDE.md 的"题库数据结构"小节更新套题统计表

### 添加新套题的批处理流程

如需从外部来源（如驾驶员之家、人人文库等）批量导入新题库：

1. 用 `curl` 或 Python `urllib` 抓取页面，注意 GB18030 解码
2. 用正则提取 `<p><b>N、题干</b></p>` / `<p>A、选项</p>` / `<b>正确答案：X</b>` / `<p class="fxda">解析</p>` 四块
3. 按 `category` / `topic` / `type` 规则分类（参考现有 `set: 'xxx'` 取值）
4. 拼成 `id: NN, type, category, topic, set, text, options, answer, explanation` 对象列表，写入 `QUESTIONS` 数组
5. 在 `#set-filter` 增加新套题 checkbox
6. 更新首页的 `intro.innerHTML` 统计——已写为动态从 `QUESTIONS.map(q => q.set || '官方基础题库')` 去重，无需手动改数字

### 修改样式

CSS 集中在 `<style>` 顶部，颜色变量在 `:root`（行 9–20）。修改主题色只改变量即可。

### 模式 / 流程改动

主要操作点：
- `switchMode` 行 2392 — 屏幕可见性 + 计时器清理
- `startPractice` 行 2420 / `startExam` 行 5613 — 初始化题目、答案、标记集合
- `submitExam` 行 5663 — 计算总分 / 判定通过
- `renderQuestion` 行 2444 — 单题渲染；practice 模式下"已选未确认"显示「确认本题」按钮，已确认显示对错反馈
- `onAnswerChange` 行 2547 — 仅更新 `STATE.answers[q.id]`，**不再自动标记 `STATE.checked`**
- `confirmCurrentQuestion` 行 2572 — 显式置位 `STATE.checked[q.id] = true` 并刷新视图
- `renderReview` 行 5717 — 回顾界面渲染
- `getQuestionSet` 行 5994 — 兜底映射 `q.set || '官方基础题库'`，让原版 135 题也能被套题过滤器识别
- `getFilteredQuestions` 行 5999 — 现在过滤四类条件：category、type、topic、**set**

### 练习模式的"显式确认"约定

旧版练习模式：选项一点就立即判定对错，多选题尤其吃亏（选 A 立刻看到 A 是否正确，等于把答案漏了出来）。

当前实现（已修复）：
1. 用户勾选 / 取消选项 → `STATE.answers[q.id]` 更新，选项保持蓝色"已选"高亮
2. 题目下方出现「✓ 确认本题」按钮（多选会额外提示"请勾选全部后再确认"）
3. 用户点击确认 → `STATE.checked[q.id] = true`，选项变为绿（对）/红（错），下方出现正确选项与解析
4. 已确认后用户仍可改选项，改完点「🔄 重新核对」再次判定

如需恢复"选完即判"的老行为：把 `onAnswerChange` 中那段"不再自动核对"的注释恢复为原来的 `STATE.checked[q.id] = true` 即可。

### 移动端适配

三层断点（CSS 行号附近 `index.html`）：

| 断点 | 行为 |
|---|---|
| ≤ 900px（平板） | 答题卡改为右侧抽屉（`transform: translateX(100%)` 隐藏），头部加 `📌 答题卡` 切换按钮；答题卡网格 5 列 → 8 列 |
| ≤ 640px（手机） | 头部 h1 缩到 15px；选项按钮最小 56px、触摸目标 ≥ 44px；底部 `上一题/标记/下一题` 改为 sticky 操作栏并加 `env(safe-area-inset-bottom)`；字号 15–16px；结果页统计 5 列 → 2 列 |
| ≤ 380px（小屏） | h1 缩到 14px；答题卡 5 列；导航按钮缩小 padding |

JS 配套：
- `toggleSidebar()` / `openSidebar()` / `closeSidebar()`（行 6493–6531）— 抽屉显隐与背景遮罩
- `renderPalette` 中点击答题卡项后自动收起（仅 `window.innerWidth <= 900`）
- `switchMode` 切模式时调用 `closeSidebar()`

### 已知约束 / 可改进点

- **无持久化**：刷新页面即丢失所有答题状态。如需保存进度，需在 `startPractice` / `startExam` / `onAnswerChange` 处接入 `localStorage`
- **题库 id 须唯一**：目前 id 1–498；新增题目需要避开（可用 499+，或重组已有空缺）
- **时长选项硬编码**：`#time-filter` 行 635–641，若改 50 分钟默认值需同步调整
- **未做无障碍优化**：依赖鼠标点击；色盲用户区分"已答/当前/已标记"可能困难
- **无 i18n**：UI 文案全部为简体中文（含 `lang="zh-CN"`），改动会影响所有用户
- **题量约 498 道**：原版 135 题 + 第三方题库（驾驶员之家、人人文库等）抓取的 363 题。第三方题目的解析为爬取原文，未做人工校对，可能存在质量参差
- **题库来源标注缺失**：页脚目前只列出官方三类参考文件；如需在 UI 标注每套题来源，需要为每条题目添加 `source` 字段并在结果页展示

## 浏览器兼容性

使用 `document.querySelectorAll`、模板字符串、`Set`、`Array.from`、箭头函数 —— 现代浏览器即可运行（Chrome / Edge / Firefox / Safari 近 5 年版本均可）。
