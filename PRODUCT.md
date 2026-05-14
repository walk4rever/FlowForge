# FlowForge — 产品定位与路线图

> 内部文件，持续更新
> 最后更新：2026-05-14

---

## 1. 一句话定位

**FlowForge 是 AI workflow 的铸造厂：把自然语言写的 skill，转化成可版本控制、可验证质量、可稳定执行的 flow。**

---

## 2. 为什么现在

### 现在的问题

AI 应用开发有两条路，都有已知上限：

**路径 A：直接写代码（TypeScript/Python）**
- 硬编码业务逻辑，每次改 prompt 要走 PR → review → 部署
- LLM 调用散落在各处，没有统一观测
- 没有版本管理语义："上次改了什么导致质量下降？"

**路径 B：写 skill.md 交给 agent 解释执行**
- 非确定性：同一个 skill，LLM 每次执行路径不同
- 不可观测：不知道当前在哪一步
- 不可复用：别人拿到你的 skill，结果可能完全不同
- 无法商业化：skill.md 是"建议"，不是"软件"

### 核心洞察

> **skill.md 是食谱，无法卖；flow.yaml 是软件，可以卖。**

skill→flow 的转化，把 AI workflow 从"建议"变成"工程产物"——可以 diff、测试、版本控制、依赖、商业化。

---

## 3. 产品架构

### 三层结构

```
┌─────────────────────────────────────────────────────────┐
│                      skill.md                           │
│           自然语言描述的 workflow（任何人都能写）          │
└────────────────────────┬────────────────────────────────┘
                         │
              [FlowForge Harness — Iterate 模式]
              · 解析 skill → 生成 flow.yaml 草稿
              · 执行 eval suite，得到质量分数
              · 自动修改低质量 step，重新评估
              · 达到阈值后，提交人工 review
                         │
              [Human Review + Confirm]
              · 查看 flow.yaml 变化 diff
              · 查看 eval 分数曲线
              · 确认 → git commit + tag
                         │
┌────────────────────────▼────────────────────────────────┐
│                flow.yaml（IR，核心产物）                  │
│   steps · output_schema · retry_policy · eval_suite     │
│   git versioned · frozen at tag · harness-agnostic      │
└────────────────────────┬────────────────────────────────┘
                         │
              [FlowForge Harness — Execute 模式]
              · 纯解释器，读 flow.yaml 按定义执行
              · AI 只出现在 type:llm 的 step 里
              · 每步记录 input/output/duration
                         │
              ┌──────────┴──────────┐
         LLM Gateway            Memory + Tools
         (model routing)        (GBrain / MCP)
```

### flow.yaml（IR）的关键设计原则

- **Harness-agnostic**：任何实现了 FlowForge Harness Spec 的执行器都可以运行
- **Output schema 强制校验**：step N 的输出必须符合 schema，才能流入 step N+1
- **Git 即 freeze**：`git commit + tag` 就是部署，没有额外的 freeze 基础设施
- **Eval 内建**：每个 flow 附带 eval suite，质量分数是 flow 的一等公民属性

```yaml
# 示例：buffett-research.flow.yaml
id: buffett-research
version: 1.2.0
publisher: rafael@air7.fun      # 未来接入 Aurum 身份
eval_suite: evals/buffett-research.yaml

steps:
  - id: understand
    type: llm
    model: haiku
    prompt: prompts/understand.md
    output_schema:
      intent: enum[search, document, graph, combined]
      keyword: string
      yearFrom: integer|null
      yearTo: integer|null

  - id: retrieve
    type: tool
    tool: buffett_search
    depends_on: [understand]
    input:
      q: "{{understand.keyword}}"
      yearFrom: "{{understand.yearFrom}}"
      yearTo: "{{understand.yearTo}}"
    output_schema:
      chunks: array
      found: integer

  - id: generate
    type: llm
    model: sonnet
    depends_on: [retrieve]
    stream: true
    prompt: prompts/generate.md
    retry:
      max_attempts: 2
      on: schema_validation_error
```

---

## 4. 用户与客户

### 两类用户

**C 端：独立 AI 开发者**
- 写了好用的 RAG skill / 研究 skill / 客服 skill
- 今天只能开源分享，无法保证质量，无法收费
- 有了 FlowForge：flow 可以被严肃消费，可以商业化

**B 端：企业 AI 平台团队**
- 公司里有十几个 AI workflow 散落在各处
- 没有版本管理，没有审计日志，出了问题不知道哪步错
- 有了 FlowForge：统一管理，版本可溯，合规可证

### 谁付钱

| 客户 | 付费动机 | 模式 |
|---|---|---|
| 企业（B 端） | AI workflow 治理、审计、合规 | SaaS 订阅 |
| Flow 消费者 | 用别人发布的高质量 flow，省去自研 | 按使用量付费 |
| Flow 发布者 | 通过 marketplace 将 skill 变现 | 平台抽佣（未来） |

---

## 5. 商业模式

### Phase 1：开源工具（验证期）
- FlowForge core 开源：IR schema + harness spec + 基础 converter
- 不收费，积累 flow 案例和 eval 基准
- 目标：用 buffett-tribe 验证核心假设

### Phase 2：FlowForge Enterprise（第一个商业形态）
- 企业自托管或 SaaS
- 内部 flow registry：管理公司所有 AI workflow 的版本和质量
- 收费功能：audit log、RBAC、eval dashboard、CI/CD 集成
- 类比定位：**AI workflow 的 Artifactory**

### Phase 3：FlowForge Marketplace（双边市场）
- Publisher 发布 flow，Consumer 付费使用
- 两种消费模式：
  - **下载模式**：获取 flow.yaml，在自己的 harness 运行
  - **SaaS 执行模式**：委托 publisher 的 harness 运行（flow 不离开作者），天然防盗用
- 平台抽取佣金
- 接入 Aurum 身份层：密码学验证 publisher 身份

---

## 6. 与 Aurum 的关系

```
Aurum 解决：WHO — 这个 agent/developer 是谁？可信吗？
FlowForge 解决：WHAT — 这个 flow 做什么？质量如何？

两者互补，不重叠：
  · FlowForge 的 publisher 字段 → 未来填 Aurum 地址
  · Aurum 的任务委托层 → 承载 FlowForge 的 SaaS 执行模式
  · 合体 → 可信 publisher 发布 flow，消费者通过 Aurum 委托执行

Marketplace 是建在两者之上的第三个产品。
```

**短期**：FlowForge 独立运行，publisher 用 GitHub/email 身份。
**中期**：Aurum P0（任务协作）完成后，FlowForge 接入 Aurum 身份层。
**长期**：Marketplace 成为 Aurum + FlowForge 生态的商业变现层。

---

## 7. Roadmap

### Phase 1 — 验证核心假设（现在）

目标：证明"skill 可以被解析成结构化 flow 并稳定执行"

```
Week 1：
  · 设计 flow.yaml schema（含 output_schema、eval_suite 字段）
  · 把 buffett-tribe RAG pipeline 写成 flow.yaml
  · 写 harness executor（~200 行，纯解释器，无 AI 决策）

Week 2：
  · 接入 buffett-tribe 现有 30 题 eval suite
  · 手动迭代 flow.yaml → human review → git commit
  · 对比 TypeScript 实现：质量、稳定性、可观测性

Week 3（可选）：
  · 写 harness iterate 模式：eval 失败 → 自动修改 step → 重跑
  · 验证 pi-autoresearch 风格的自动优化是否收敛

验证成功标准：
  ✓ flow 输出质量 ≥ 当前 TypeScript 实现
  ✓ 每次执行的步骤路径完整可回溯
  ✓ 改一个 step 的 prompt，不需要改任何代码
```

### Phase 2 — Enterprise Flow Registry

- 打包成可部署的服务（Docker/cloud）
- Flow registry UI：版本历史、eval dashboard、diff viewer
- CI/CD 集成：PR 里自动跑 eval suite
- RBAC：谁可以发布、谁可以修改
- Audit log：每次 execute 记录 flow version + step outputs
- 目标客户：有多个 AI workflow 在生产环境的团队

### Phase 3 — Marketplace

- 公开 flow registry
- Publisher 注册、发布、定价
- Consumer 搜索、安装、使用
- SaaS 执行模式（接入 Aurum 后）
- 平台抽佣机制

---

## 8. 未解决的关键问题

1. **Harness Spec 的边界**：哪些 step type 必须支持？条件分支、并行、循环的标准表达是什么？

2. **Output schema 校验失败的处理**：是 retry 同一个 step，还是 fail fast，还是走 fallback 分支？

3. **Eval suite 的设计标准**：谁来写 eval？eval 本身的质量怎么保证？

4. **Freeze 粒度**：整个 flow 是一个版本，还是每个 step 可以独立版本化？

---

*FlowForge 和 Aurum 与任何投资人或第三方机构无关联。内部构思文件，不对外公开。*
