# AGENTS.md — 面试助手 Skills 系统

本文件是本仓库所有 AI agent、人类开发者和协作工具的协作规范。所有对本仓库的贡献都必须遵循本文档中的规则。

---

## 目录

1. [项目目标](#项目目标)
2. [项目结构](#项目结构)
3. [编码规范](#编码规范)
4. [数据文件位置](#数据文件位置)
5. [Skills 实现规范](#skills-实现规范)
6. [测试命令](#测试命令)
7. [不允许修改的原则](#不允许修改的原则)

---

## 项目目标

这是一个**本地优先的"面试助手 Skills 系统"**，核心由两个 agent 组成：

| Agent | 职责 |
|---|---|
| `interviewer_agent` | 基于题库、面经、简历薄弱点，模拟面试官进行提问和追问 |
| `candidate_agent` | 以用户本人身份回答问题，同时输出回答建议、包装建议、风险高亮和需要补充的信息 |

---

## 项目结构

```
Skills--/
├── AGENTS.md                  # 本文件：协作规范
├── README.md                  # 项目简介
│
├── agents/
│   ├── interviewer_agent.py   # 面试官 agent：提问 & 追问
│   └── candidate_agent.py     # 候选人 agent：答题 & 建议
│
├── skills/
│   └── <skill_name>/
│       ├── skill.json         # skill 元数据（输入、输出、规则）
│       ├── prompt.txt         # 该 skill 使用的 prompt 模板
│       └── tests/
│           └── cases.json     # 测试样例（输入/期望输出对）
│
├── data/
│   ├── question_bank.json     # 题库
│   ├── interview_experiences.json  # 面经
│   ├── resume_weakpoints.json      # 简历薄弱点
│   └── sessions/              # 每次面试会话的本地记录
│       └── <session_id>.json
│
├── tests/
│   ├── test_interviewer_agent.py
│   ├── test_candidate_agent.py
│   └── test_skills.py
│
└── requirements.txt
```

---

## 编码规范

### 语言 & 版本

- **Python 3.11+**，所有代码使用类型注解（`typing` 模块或 PEP 604 `X | Y` union 语法，均在 3.10+ 可用）。
- 禁止使用已废弃的 API；保持与 Python 3.11 的完全兼容。

### 代码风格

- 遵循 [PEP 8](https://peps.python.org/pep-0008/)。
- Python 代码缩进使用 **4 个空格**（PEP 8 标准）；JSON 文件缩进使用 **2 个空格**。
- 使用 `black` 进行格式化（行宽 88）。
- 使用 `ruff` 进行 lint 检查。
- 字符串使用双引号 `"`。
- 每个公开函数/类必须有 docstring（Google 风格）。

### 模块职责

- 每个 agent 文件只负责自身 agent 的逻辑，不跨越职责边界。
- skill 逻辑必须完全封装在对应的 `skills/<skill_name>/` 目录中。
- 数据读写只能通过 `data/` 目录中的 JSON 文件；禁止硬编码数据。

### 提交规范

- commit message 使用中文或英文，格式：`<类型>: <描述>`
  - 类型：`feat`、`fix`、`docs`、`test`、`refactor`、`chore`
  - 示例：`feat: 添加 interviewer_agent 追问逻辑`

---

## 数据文件位置

所有数据**仅保存在本地**，不做任何云端同步。

| 文件 | 说明 |
|---|---|
| `data/question_bank.json` | 题库，包含题目、分类、难度、参考答案框架 |
| `data/interview_experiences.json` | 面经，包含真实面试题和考察点 |
| `data/resume_weakpoints.json` | 简历薄弱点，由用户手动维护 |
| `data/sessions/<session_id>.json` | 每次面试会话的完整记录（问题、回答、建议） |

### JSON 格式约定

- 使用 UTF-8 编码，缩进 2 个空格。
- 所有时间戳使用 ISO 8601 格式（`YYYY-MM-DDTHH:MM:SSZ`）。
- 枚举值使用 `snake_case`。

---

## Skills 实现规范

每个 skill 必须满足以下全部要求，缺一不可。

### 目录结构

```
skills/<skill_name>/
├── skill.json      # skill 元数据
├── prompt.txt      # prompt 模板
└── tests/
    └── cases.json  # 测试样例
```

### `skill.json` 必填字段

```jsonc
{
  "name": "skill_name",           // skill 唯一标识符（snake_case）
  "description": "...",           // skill 的功能描述
  "input": {                      // 输入字段定义
    "<field>": "<type> | <description>"
  },
  "output": {                     // 输出字段定义
    "<field>": "<type> | <description>"
  },
  "rules": [                      // 业务规则列表（字符串数组）
    "规则1",
    "规则2"
  ]
}
```

### `prompt.txt` 规范

- 使用 `{{变量名}}` 作为占位符，与 `skill.json` 的 input 字段对应。
- prompt 必须明确区分**事实回答区**和**建议高亮区**（见核心原则）。
- prompt 长度不超过 2000 字符（不含占位符展开后的内容）。

### `tests/cases.json` 规范

```jsonc
[
  {
    "id": "case_001",
    "description": "测试场景描述",
    "input": { ... },           // 与 skill.json input 对应
    "expected_output": { ... }, // 期望输出（用于断言验证）
    "tags": ["edge_case"]       // 可选标签
  }
]
```

每个 skill 至少包含 **3 个测试样例**，覆盖：
1. 正常输入
2. 边界/薄弱输入
3. 用户经历不足时的降级处理

---

## 测试命令

```bash
# 安装依赖
pip install -r requirements.txt

# 运行所有测试
pytest tests/ -v

# 运行单个 agent 测试
pytest tests/test_interviewer_agent.py -v
pytest tests/test_candidate_agent.py -v

# 运行所有 skill 测试
pytest tests/test_skills.py -v

# 代码格式化
black .

# Lint 检查
ruff check .

# 格式化 + lint（一键）
black . && ruff check .
```

---

## 不允许修改的原则

以下原则是本系统的核心约束，**任何时候、任何人、任何 agent 都不得违反**：

### 1. 本地优先，禁止云端同步
- 所有数据只能存储在 `data/` 目录下的本地 JSON 文件中。
- 禁止接入任何云存储、数据库服务或第三方同步 API。

### 2. 禁止虚构信息
- 禁止虚构不存在的经历、公司、岗位、项目和硬数据（薪资、时间、绩效等）。
- `candidate_agent` 的回答必须严格基于用户提供的真实经历。

### 3. 建议与事实严格分离
- 所有"美化、转化、承认不足、补充建议"**必须**作为高亮建议单独输出。
- 禁止将建议内容混入事实答案区域。
- 输出结构必须包含明确的 `fact_answer`（事实回答）和 `suggestions`（建议）两个区域。

### 4. Skills 必须完整实现
- 每个 skill 必须同时具备：输入定义、输出定义、规则、prompt 模板、测试样例。
- 不允许存在缺少上述任一要素的"不完整 skill"。

### 5. 禁止修改本原则列表
- 本节（"不允许修改的原则"）的内容只能增加，不能删除或弱化。
- 如需新增约束，须在 PR 描述中明确说明理由。
