---
name: arxiv-paper-tracker-design
description: AI LLM 论文追踪系统设计文档 - 自动抓取、智能筛选、深度分析、即时推送
created: 2026-03-26
status: approved
---

# arXiv 论文追踪系统设计文档

## 1. 概述

### 1.1 目标

构建一个自动化论文追踪系统，核心能力：
- **定时抓取** arXiv 最新论文及 K8s 社区、硬件生态动态
- **智能筛选** 基于发布者、热度、关键词的多级过滤
- **深度分析** Claude 驱动的技术摘要，保留关键技术细节
- **智能推送** 重大突破即时推送 + 每周汇总

### 1.2 目标用户关注领域

| 领域 | 具体方向 |
|------|---------|
| **LLM 推理优化** | 多模态推理、长上下文、模型架构创新、量化、投机解码、KV cache 优化 |
| **K8s 智能体基础设施** | 新标准、热点项目孵化/毕业/发布、KEP 提案 |
| **软硬协同推理优化** | NVIDIA、灵衢、CXL 等加速技术 |

### 1.3 设计决策

| 维度 | 决策 |
|------|------|
| 运行形态 | 混合方案（Claude Skill 编排分析 + Python 脚本定时触发） |
| 报告频率 | 智能推送（重要即时 + 周汇总） |
| 推送渠道 | Git 仓库存储 + 飞书 Webhook |
| 重要性判断 | 发布者/热度信号 + LLM 智能分析（混合策略） |
| 数据源 | arXiv API + Semantic Scholar API + GitHub + 社区热度（多源融合） |

---

## 2. 系统架构

### 2.1 分层架构

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 1: 数据采集层 (Python 脚本)                               │
│  ├─ arXiv API / Semantic Scholar API                            │
│  ├─ GitHub Releases / CNCF 博客                                 │
│  └─ Hacker News / Reddit 热度抓取                               │
├─────────────────────────────────────────────────────────────────┤
│  Layer 2: 预处理层 (Python)                                      │
│  ├─ 关键词/作者过滤                                             │
│  ├─ 热度评分计算                                               │
│  └─ 初步分类（推理优化 / K8s / 硬件加速）                        │
├─────────────────────────────────────────────────────────────────┤
│  Layer 3: 深度分析层 (Claude Skill)                              │
│  ├─ 高价值论文深度摘要                                          │
│  ├─ 技术创新点提炼                                             │
│  └─ 对关注领域的影响分析                                        │
├─────────────────────────────────────────────────────────────────┤
│  Layer 4: 分发层                                                │
│  ├─ Git 报告存储                                               │
│  └─ 飞书 Webhook 推送                                          │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 数据流

```
[Cron 定时器] → [采集层] → [JSON 缓存] → [预处理层] → [分流判断]
                                                        ↓
                                          ┌─────────────┴─────────────┐
                                          ▼                           ▼
                                   [即时推送路径]              [周汇总路径]
                                          │                           │
                                          ▼                           ▼
                                   [Claude Skill]              [Claude Skill]
                                          │                           │
                                          └───────────┬───────────────┘
                                                      ▼
                                              [分发层]
                                              ├─ Git 存储
                                              └─ 飞书推送
```

---

## 3. 项目结构

```
arxiv-tracker/
├── collectors/                  # Layer 1: 数据采集
│   ├── arxiv_collector.py       #   arXiv API + Semantic Scholar
│   ├── github_collector.py      #   K8s/CNCF GitHub releases & KEP
│   └── community_collector.py   #   Hacker News / Reddit 热度
├── filters/                     # Layer 2: 预处理 & 过滤
│   ├── config.py                #   关注的作者/机构/关键词配置
│   ├── scorer.py                #   热度评分 & 优先级排序
│   └── classifier.py            #   论文分类（推理优化/K8s/硬件加速）
├── reports/                     # Layer 3: 报告生成输出
│   ├── daily/                   #   即时推送内容
│   └── weekly/                  #   周汇总报告
├── data/                        #   采集数据缓存（JSON）
│   ├── arxiv/
│   ├── github/
│   ├── community/
│   └── filtered/                #   预处理后数据
├── config/
│   ├── sources.yaml             #   数据源配置（API keys、查询参数）
│   ├── watchers.yaml            #   关注的作者/机构/项目列表
│   └── feishu.yaml              #   飞书 Webhook 配置
├── skills/
│   └── paper-analyzer.md        #   Claude Code skill 定义
├── run.py                       #   主入口
└── requirements.txt
```

---

## 4. 数据采集层（Layer 1）

### 4.1 采集源与调度

| 数据源 | 采集频率 | API | 采集内容 |
|--------|---------|-----|---------|
| arXiv cs.CL / cs.AI / cs.LG / cs.AR / cs.DC | 每天 1 次 | arXiv API + Semantic Scholar | 论文元数据、摘要、作者、引用数 |
| K8s GitHub (kubernetes/kubernetes, SIG 仓库) | 每天 1 次 | GitHub REST API | Releases、KEP 状态变更、CNCF 项目孵化/毕业 |
| NVIDIA 博客 / 灵衢 / CXL 联盟 | 每天 1 次 | RSS / Web 抓取 | 技术发布、性能数据、新硬件/规范 |
| Hacker News / Reddit r/MachineLearning | 每天 2 次 | 公开 API | arXiv 论文讨论热度、社区关注度 |

### 4.2 arXiv 采集配置

```yaml
# config/watchers.yaml
arxiv_queries:
  categories: ["cs.CL", "cs.AI", "cs.LG", "cs.AR", "cs.DC"]
  keywords:
    tier1_critical:
      - "speculative decoding"
      - "flash attention"
      - "ring attention"
      - "continuous batching"
    tier2_high:
      - "inference optimization"
      - "long context"
      - "multimodal"
      - "mixture of experts"
      - "quantization"
      - "KV cache"
    tier3_normal:
      - "language model"
      - "transformer"
      - "attention mechanism"
```

### 4.3 Semantic Scholar 增强数据

对 arXiv 采集到的论文，批量查询 Semantic Scholar 获取：
- **引用数**（判断影响力）
- **作者历史发表**（判断是否为关键发布者）
- **相关论文推荐**（发现关联工作）

### 4.4 统一数据格式

```python
{
    "source": "arxiv" | "github" | "nvidia_blog" | "hackernews",
    "id": "2503.12345",
    "title": "...",
    "authors": [...],
    "date": "2026-03-26",
    "url": "https://...",
    "summary": "...",
    "tags": ["inference", "quantization"],
    "heat_score": 0,
    "priority": "normal"  # normal | high | critical
}
```

---

## 5. 预处理与过滤层（Layer 2）

### 5.1 三级过滤流水线

```
原始数据 → [关键词过滤] → [发布者匹配] → [热度评分] → [分类]
           约 100-200篇/天   约 50-80篇     约 20-30篇
```

### 5.2 发布者配置

```yaml
# config/watchers.yaml
watchers:
  authors:
    - name: "Tri Dao"              # FlashAttention 作者
    - name: "Junxian He"           # vLLM 核心
    - org: "DeepSpeed Team"        # Microsoft
    - org: "Google DeepMind"
    - org: "Anthropic"
    - org: "vLLM Team"
    - org: "Hugging Face"
```

### 5.3 热度评分公式

```
heat_score = w1 * 引用增速 + w2 * HN/Reddit 讨论数
           + w3 * 作者影响力 + w4 * 关键词层级

# 默认权重
w1 = 0.3   # 引用增速（24h内新增引用）
w2 = 0.3   # 社区讨论热度
w3 = 0.2   # 作者/机构影响力
w4 = 0.2   # 关键词层级（tier1=100, tier2=70, tier3=40）
```

**即时推送触发条件**: `heat_score > 80 且 priority in [critical, high]`

### 5.4 分类标签

| 标签 | 覆盖内容 |
|------|---------|
| `inference-opt` | 推理优化（量化、投机解码、KV cache、continuous batching） |
| `architecture` | 模型架构创新（MoE、新注意力机制） |
| `multimodal` | 多模态推理 |
| `long-context` | 长上下文技术 |
| `k8s-infra` | K8s 智能体基础设施 |
| `hw-accel` | 软硬协同（NVIDIA、灵衢、CXL） |

---

## 6. 深度分析层（Layer 3）

### 6.1 Claude Skill 定义

**文件**: `skills/paper-analyzer.md`

**触发方式**：
- 即时推送：`/paper-analyzer --mode=instant --items=<item_ids>`
- 周汇总：`/paper-analyzer --mode=weekly --date-range=2026-03-19..2026-03-26`

### 6.2 Skill 核心职责

1. 读取 `data/filtered/` 目录下预处理后的 JSON 数据
2. 对高优先级论文生成**深度技术摘要**（非泛泛总结）
3. 分析对用户关注领域的**影响和机会点**
4. 输出结构化报告

### 6.3 摘要模板

```markdown
## {{title}}

**来源**: {{source}} | **日期**: {{date}} | **热度**: {{heat_score}}

### 核心技术点
- [具体技术点1，保留原文关键术语]
- [具体技术点2，保留原文关键术语]

### 创新点
[用 2-3 句话说明与现有方法的本质区别]

### 对我关注领域的影响
- **推理优化**: [具体影响]
- **K8s 基础设施**: [具体影响]（如适用）
- **软硬协同**: [具体影响]（如适用）

### 值得跟进的点
- [ ] [可落地的技术点或实验方向]
- [ ] [需要进一步阅读的关联论文]

---
arXiv: {{arxiv_id}} | [原文链接]({{url}})
```

### 6.4 即时推送 vs 周汇总

| 维度 | 即时推送 | 周汇总 |
|------|---------|--------|
| 触发条件 | heat_score > 80 且 tier1/tier2 | 每周一自动生成 |
| 内容范围 | 仅高优先级条目（预计 2-5 篇/天） | 全部通过过滤的条目（20-30 篇/周） |
| 报告深度 | 完整技术摘要 | 精简摘要 + 分类索引 |
| 飞书推送 | 即时 Webhook | 周一早上推送 |

---

## 7. 分发层（Layer 4）

### 7.1 Git 存储

- 即时推送报告：`reports/daily/2026-03-26-instant.md`
- 周汇总报告：`reports/weekly/2026-W13.md`
- 自动 `git add && git commit && git push`

### 7.2 飞书 Webhook 推送

使用飞书开放平台的**消息卡片**格式：

```json
{
  "msg_type": "interactive",
  "card": {
    "header": {
      "title": { "tag": "plain_text", "content": "🚀 论文即时推送" },
      "template": "blue"
    },
    "elements": [
      {
        "tag": "div",
        "text": {
          "tag": "lark_md",
          "content": "**{{title}}**\n{{summary_short}}"
        }
      },
      {
        "tag": "action",
        "actions": [
          {
            "tag": "button",
            "text": { "tag": "plain_text", "content": "查看详情" },
            "url": "{{report_url}}"
          }
        ]
      }
    ]
  }
}
```

---

## 8. 调度与编排

### 8.1 调度流程

```
┌────────────────────────────────────────────────────────────────────┐
│                        每日 08:00 触发                              │
│                      (系统 cron / Claude Cron)                      │
└────────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌────────────────────────────────────────────────────────────────────┐
│  Step 1: 数据采集 (collectors)                                      │
│  ├─ arxiv_collector.py      → data/arxiv/2026-03-26.json          │
│  ├─ github_collector.py     → data/github/2026-03-26.json         │
│  └─ community_collector.py  → data/community/2026-03-26.json      │
└────────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌────────────────────────────────────────────────────────────────────┐
│  Step 2: 预处理过滤 (filters)                                       │
│  ├─ 加载 watchers.yaml 配置                                        │
│  ├─ 关键词过滤 + 发布者匹配                                        │
│  ├─ 计算热度评分                                                   │
│  └─ 输出 → data/filtered/2026-03-26.json                          │
└────────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌────────────────────────────────────────────────────────────────────┐
│  Step 3: 分流判断                                                  │
│  ├─ 有 critical/high 且 heat_score > 80?                          │
│  │   └─ YES → 触发即时推送流程                                     │
│  └─ 否则 → 数据累积等待周汇总                                      │
└────────────────────────────────────────────────────────────────────┘
                                 │
                    ┌────────────┴────────────┐
                    ▼                         ▼
        ┌──────────────────┐       ┌──────────────────┐
        │  即时推送路径     │       │   周汇总路径      │
        │  (立即执行)       │       │  (每周一 09:00)   │
        └──────────────────┘       └──────────────────┘
                    │                         │
                    ▼                         ▼
┌────────────────────────────────────────────────────────────────────┐
│  Step 4: Claude Skill 分析 (paper-analyzer)                        │
│  /paper-analyzer --mode=instant --date=2026-03-26                 │
│                  或                                                │
│  /paper-analyzer --mode=weekly --week=2026-W13                    │
└────────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌────────────────────────────────────────────────────────────────────┐
│  Step 5: 分发 (distribute)                                         │
│  ├─ 生成 Markdown 报告 → reports/                                 │
│  ├─ git commit && git push                                        │
│  └─ 飞书 Webhook 推送                                             │
└────────────────────────────────────────────────────────────────────┘
```

### 8.2 触发机制

**方式 1：Claude Code Cron（开发调试）**
```
CronCreate(cron="0 8 * * *", prompt="/run-arxiv-tracker --mode=daily")
CronCreate(cron="0 9 * * 1", prompt="/run-arxiv-tracker --mode=weekly")
```

**方式 2：系统 crontab（生产环境）**
```bash
# crontab -e
0 8 * * * cd /path/to/arxiv-tracker && python run.py --mode=daily
0 9 * * 1 cd /path/to/arxiv-tracker && python run.py --mode=weekly
```

### 8.3 主入口伪代码

```python
# run.py
def main(mode: str):
    # Step 1: 采集
    collect_all_sources()

    # Step 2: 过滤
    items = filter_and_score()

    # Step 3: 分流
    if mode == "daily":
        hot_items = [i for i in items
                     if i["heat_score"] > 80
                     and i["priority"] in ["critical", "high"]]
        if hot_items:
            invoke_skill(mode="instant", items=hot_items)
            distribute_report("instant")

    elif mode == "weekly":
        invoke_skill(mode="weekly", items=items)
        distribute_report("weekly")


if __name__ == "__main__":
    import argparse
    parser = argparse.ArgumentParser()
    parser.add_argument("--mode", choices=["daily", "weekly"])
    args = parser.parse_args()
    main(args.mode)
```

---

## 9. 配置文件

### 9.1 sources.yaml

```yaml
# config/sources.yaml
arxiv:
  base_url: "http://export.arxiv.org/api/query"
  page_size: 100
  rate_limit: 3  # requests per second

semantic_scholar:
  base_url: "https://api.semanticscholar.org/graph/v1"
  api_key: "${SEMANTIC_SCHOLAR_API_KEY}"

github:
  token: "${GITHUB_TOKEN}"
  repos:
    - "kubernetes/kubernetes"
    - "kubernetes/enhancements"
    - "cncf/toc"

community:
  hackernews:
    base_url: "https://hacker-news.firebaseio.com/v0"
  reddit:
    subreddits: ["MachineLearning", "LocalLLaMA"]
```

### 9.2 feishu.yaml

```yaml
# config/feishu.yaml
webhook_url: "${FEISHU_WEBHOOK_URL}"
message_templates:
  instant:
    title: "🚀 论文即时推送"
    color: "blue"
  weekly:
    title: "📊 本周论文汇总"
    color: "green"
```

---

## 10. 实施计划

### Phase 1：最小可用版本（1-2 天）

- [ ] 项目骨架搭建
- [ ] arXiv 采集器实现
- [ ] 基础关键词过滤
- [ ] Claude Skill 基础版本
- [ ] Git 报告存储

### Phase 2：完善采集与过滤（2-3 天）

- [ ] Semantic Scholar 集成
- [ ] GitHub 采集器
- [ ] 热度评分系统
- [ ] 飞书 Webhook 推送

### Phase 3：智能化增强（2-3 天）

- [ ] 社区热度采集（HN/Reddit）
- [ ] 智能分流判断
- [ ] 周汇总报告
- [ ] 调度系统完善

### Phase 4：优化与迭代（持续）

- [ ] 性能优化（增量采集）
- [ ] 配置热更新
- [ ] 错误恢复机制
- [ ] 监控与告警
