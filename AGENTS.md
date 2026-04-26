# AGENTS.md — 面试助手 Skills 系统

本文件是所有 agent、协作者和自动化工具在该仓库中工作的规范文档。所有对本仓库的修改必须遵守此文件中的规则。

---

## 1. 项目目标

这是一个**本地优先的面试助手 skills 系统**，核心由两个 agent 组成：

| Agent | 职责 |
|---|---|
| `interviewer_agent` | 基于题库、面经、简历薄弱点，模拟面试官提问与追问 |
| `candidate_agent` | 以用户本人身份回答问题，同时输出回答建议、包装建议、风险高亮和补充信息 |

---

## 2. 项目结构

```
Skills--/
├── AGENTS.md                  # 本文件，项目规范（禁止删除）
├── README.md                  # 项目简介
│
├── agents/
│   ├── interviewer_agent/
│   │   ├── __init__.py
│   │   ├── agent.py           # 面试官 agent 主逻辑
│   │   ├── prompts.py         # 提问 / 追问 prompt 模板
│   │   └── tests/
│   │       └── test_interviewer.py
│   └── candidate_agent/
│       ├── __init__.py
│       ├── agent.py           # 候选人 agent 主逻辑
│       ├── prompts.py         # 回答 / 建议 prompt 模板
│       └── tests/
│           └── test_candidate.py
│
├── skills/
│   ├── README.md              # skill 实现规范说明
│   └── <skill_name>/
│       ├── skill.py           # skill 主体
│       ├── schema.json        # 输入 / 输出 schema
│       ├── rules.md           # 规则说明
│       ├── prompts.py         # prompt 模板
│       └── tests/
│           ├── test_skill.py
│           └── fixtures/      # 测试用例 JSON 文件
│
├── data/                      # 本地数据目录（不上传云端）
│   ├── resume/
│   │   └── resume.json        # 用户简历数据
│   ├── question_bank/
│   │   └── questions.json     # 题库
│   ├── interview_records/
│   │   └── <timestamp>.json   # 每次面试记录
│   └── weak_points/
│       └── weak_points.json   # 简历薄弱点标注
│
├── config/
│   └── settings.json          # 本地配置（模型、路径等）
│
├── utils/
│   ├── data_loader.py         # 本地 JSON 读写工具
│   ├── highlighter.py         # 建议高亮输出工具
│   └── validator.py           # 输入 / 输出校验工具
│
└── tests/
    └── integration/           # 集成测试（跨 agent / skill）
```

---

## 3. 编码规范

### 语言与版本
- Python 3.10+
- 所有文件使用 UTF-8 编码

### 风格
- 遵循 [PEP 8](https://pep8.org/)
- 使用 `black` 格式化代码（行宽 100）
- 使用 `isort` 管理 import 顺序
- 使用 `mypy` 进行类型检查，所有公开函数必须标注类型

### 命名
- 模块、文件：`snake_case`
- 类：`PascalCase`
- 常量：`UPPER_SNAKE_CASE`
- 变量、函数：`snake_case`

### 依赖管理
- 使用 `requirements.txt` 管理依赖
- 禁止引入云端同步、数据库（SQLite 除外，作为可选本地存储）、第三方账号鉴权等依赖

### Prompt 管理
- 所有 prompt 必须集中在对应模块的 `prompts.py` 中，禁止在业务逻辑文件中硬编码 prompt 字符串
- prompt 模板使用 Python f-string 或 `str.format()` 进行变量插值

---

## 4. 数据文件位置

所有数据文件保存在 `data/` 目录下的本地 JSON 文件中。

| 数据类型 | 文件路径 | 说明 |
|---|---|---|
| 用户简历 | `data/resume/resume.json` | 用户手动维护，agent 只读 |
| 题库 | `data/question_bank/questions.json` | 可手动追加，agent 可读写 |
| 面试记录 | `data/interview_records/<timestamp>.json` | 每次面试自动生成 |
| 薄弱点标注 | `data/weak_points/weak_points.json` | 由 `interviewer_agent` 分析后更新 |
| 本地配置 | `config/settings.json` | 模型名称、数据路径等本地设置 |

**数据文件原则：**
- 所有数据文件不得上传至云端或任何第三方服务
- `data/` 和 `config/` 目录已加入 `.gitignore`
- agent 不得修改 `data/resume/resume.json` 中的原始简历内容

---

## 5. Skills 实现规范

每个 skill 必须包含以下五个要素，缺一不可：

### 5.1 输入（Input）
- 在 `schema.json` 的 `"input"` 字段中定义，使用 JSON Schema 格式
- 必须明确字段名称、类型、是否必填、取值范围或示例

### 5.2 输出（Output）
- 在 `schema.json` 的 `"output"` 字段中定义，使用 JSON Schema 格式
- **建议类输出**（包装建议、风险高亮、补充建议）必须与**事实类输出**（回答内容）分离，不得混合

### 5.3 规则（Rules）
- 在 `rules.md` 中以条目形式列出，例如：
  - 不允许输出未在简历中出现的公司、岗位、项目名称
  - 优化建议必须在 `suggestions` 字段输出，不得混入 `answer` 字段
  - 数字、时间等硬数据必须与简历原文一致

### 5.4 Prompt（Prompt Template）
- 在 `prompts.py` 中定义，以函数形式封装，接受结构化参数，返回字符串
- prompt 必须包含角色设定、任务说明、约束条件、输出格式要求

### 5.5 测试样例（Test Cases）
- 在 `tests/fixtures/` 目录下以 JSON 文件形式提供，至少包含：
  - 1 个正常输入的预期输出
  - 1 个边界输入（如空字段、缺失字段）
  - 1 个包含虚构信息的非法输入（预期 agent 拒绝或高亮警告）

### Skill schema.json 示例结构

```json
{
  "name": "skill_name",
  "version": "0.1.0",
  "description": "该 skill 的功能描述",
  "input": {
    "type": "object",
    "required": ["question", "resume_context"],
    "properties": {
      "question": { "type": "string", "description": "面试问题" },
      "resume_context": { "type": "object", "description": "简历相关上下文" }
    }
  },
  "output": {
    "type": "object",
    "properties": {
      "answer": { "type": "string", "description": "基于真实经历的回答" },
      "suggestions": {
        "type": "array",
        "items": { "type": "string" },
        "description": "包装建议、风险高亮、补充建议（高亮输出，不混入 answer）"
      }
    }
  }
}
```

---

## 6. 测试命令

```bash
# 安装依赖
pip install -r requirements.txt

# 运行所有测试
pytest tests/ -v

# 运行单个 agent 的测试
pytest agents/interviewer_agent/tests/ -v
pytest agents/candidate_agent/tests/ -v

# 运行单个 skill 的测试
pytest skills/<skill_name>/tests/ -v

# 运行集成测试
pytest tests/integration/ -v

# 代码格式检查
black --check --line-length 100 .
isort --check-only .

# 类型检查
mypy agents/ skills/ utils/
```

---

## 7. 不允许修改的原则

以下原则为本项目的核心约束，**任何 agent、协作者或自动化工具均不得违反**：

1. **禁止云端同步**：不得将任何用户数据、简历、面试记录上传至任何云服务或第三方 API（仅允许调用 LLM 推理接口，且不得传入完整简历原文）。

2. **禁止虚构硬数据**：agent 输出中不得包含简历中不存在的公司名称、岗位名称、项目名称、时间节点、数字指标等硬数据。

3. **建议与事实必须分离**：所有"表达优化、包装建议、风险高亮、补充建议"必须作为独立的 `suggestions` 字段高亮输出，严禁混入 `answer`（事实回答）字段。

4. **允许的优化范围**：可基于真实经历进行表达优化、结构化包装（如 STAR 法则）、能力迁移解释，但必须在 `suggestions` 字段中标注这是优化建议。

5. **数据本地优先**：默认数据存储方案为本地 JSON 文件，不得将本地文件存储替换为远程数据库或云存储。

6. **skill 必须完整**：每个 skill 上线前必须具备输入、输出、规则、prompt、测试样例五要素，缺少任何一项的 skill 不得合并至主分支。

7. **禁止删除本文件**：`AGENTS.md` 是本项目的核心规范文件，任何 PR 不得删除或清空本文件。

8. **禁止修改原始简历数据**：`data/resume/resume.json` 由用户手动维护，agent 只能读取，不得写入或修改。

---

*最后更新：2026-04-26*
