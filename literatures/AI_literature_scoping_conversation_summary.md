# AI 辅助文献快速阅读与 Scoping 工作流

> 对话总结:如何用 Claude 快速扫描领域文献、识别 gap、提取方法论(非系统综述场景)
> 日期:April 2026

---
2
## 背景问题

**Zihan 的需求:**
> "我现在不需要做系统综述,我是想在开始 project 之前大量但快速的用 AI 辅助我阅读领域内的文献,识别 gap,和他们的方法。"

这个场景比系统综述更适合 AI,目标不是穷尽,而是**快速理解领域 landscape、方法论模式、以及 gap**。

---

## Token 消耗的实际规模

大量文献搜索会消耗可观 token,但这类任务的 token 用得"实在",不是填充性工作。

**典型消耗:**

- 单篇论文 fetch + 结构化方法论提取:**10–20K token**
- 一个 session 扫描 20–30 篇论文:**300–500K token**
- 多个子主题平行推进:可以轻松跑满几周额度

**和其他任务比较(从高到低消耗):**

1. Agentic 长对话带工具调用(本对话)
2. 长文档 fetch + 分析
3. Extended thinking 打开的复杂推理
4. 大量代码生成 + 迭代 debug
5. 纯写作(SOP、research statement 反复迭代)
6. 普通问答

---

## 三种 AI 辅助文献阅读工作流

### 模式 1:Scoping Scan(最轻,最快)

**适用场景:** 快速判断一个领域要不要深入

**输入:** 一个主题 + 几篇种子论文

**Claude 输出:**
- 搜出 15–30 篇相关 recent papers
- 每篇 **2–3 句总结**(研究问题、方法、核心发现)
- 按 sub-themes 分组
- 区分方法创新 vs 应用扩展 vs 综述
- 一份可以一眼扫完的 landscape map

---

### 模式 2:方法论提取(中等深度)

**适用场景:** 不读全文就能知道领域用什么方法做什么数据

**输入:** 5–10 篇论文(用户提供或 Claude 搜索)

**每篇结构化提取:**
- 研究问题
- 数据源(哪个 EHR、哪个 cohort、sample size)
- 暴露 / 结局定义
- 统计方法 + 具体模型
- 主要 bias / limitation
- 作者自己声明的 gap

**输出:** Cross-paper comparison table —— 谁用 target trial emulation,谁用 propensity score,谁用 ML,用的什么 cohort 等等。

---

### 模式 3:Gap Hunting(最深,最有价值)

**适用场景:** PhD 申请 SOP 写作 / specific aims 构思 / 项目定位

**输入:** 一个研究问题或方向

**Claude 流程:**
- 识别 5–10 个 sub-questions
- 每个 sub-question 扫描现有证据
- 明确标注"证据强"、"证据矛盾"、"证据缺失"
- 输出一份 gap analysis 文档,可以直接用在 SOP 或 specific aims 里

**Token 消耗最大,但对 PhD 申请 + 项目定位最值钱。**

---

## Claude 的能力边界(必须说清楚)

### ✅ 擅长

- Web search + 大量 abstract 过滤
- `web_fetch` 整页摘录 + 结构化提取
- 跨论文方法比较
- 写"这个领域大家在争什么 / 没解决什么"的整合性判断

### ⚠️ 不擅长 / 有限制

- **付费全文:** 很多 PubMed 只能看 abstract,无法通过 Columbia VPN 拿 NEJM / JAMA 的 PDF
- **非常 niche 或非常老的文献:** 超过 2–3 年的,搜索引擎覆盖差
- **Snippet 深度局限:** 搜索结果的 snippet 信息够做 scoping,但不够做 methodology 深读
  - Claude 应该在这种情况下明确标注:"这篇我只拿到了 abstract,结论不能完全信"

---

## 怎么让输出真正能用

### 1. 开 project 前丢种子论文

给 Claude **2–3 篇用户自己觉得"这个方向就是我想做的"的论文**,从这里辐射出去比 Claude 自己瞎搜精准 10 倍。

### 2. 明确颗粒度

先说清楚想要哪种:
- Scoping map(每篇 3 句话)
- 深度提取(每篇 1 页结构化)
- Gap analysis(主题整合文档)

### 3. 直接上传 PDF

用户手头已有的论文可以直接上传,Claude 读全文,不再受 abstract 限制。

---

## 下一步

选一个具体方向开始,从 **Scoping Scan** 入手(最轻),看输出质量再决定要不要做更深。

可能的候选方向:
- LLM 在健康信息检索 / 健康咨询中的应用(原文献综述题目)
- MIND-AF 相关(AF 患者认知衰退预测 / OMOP-based clinical prediction)
- 或其他正在考虑的 PhD 方向

---

*备注:这份总结本身是本对话的产物,记录的是 Claude 与用户协作使用 AI 辅助文献工作的方法论讨论,不是某个具体领域的文献综述。*
