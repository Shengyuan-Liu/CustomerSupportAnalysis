# 协作规范

本文面向 Team Two Seven 的三位成员，说明从个人实验到代码审查、最终展示的完整协作方式。研究范围与里程碑见 [项目章程](docs/project_charter.md)，具体任务见 [Week 1 工作计划](docs/week1_sprint_plan.md)。

## 1. 基本约定

- 共享 Python 代码统一放在根目录的 `tools/`，按职责分模块；本项目采用这一套目录结构。
- `main` 分支保存已审查、可复现的阶段成果，通过 PR 合入。每个任务从最新 `main` 新建分支。
- 三个人各自维护实验 notebook；最终 `notebooks/main.ipynb` 由 Lucas 负责集成，其他成员提交需要纳入的调用、图表和解释。
- 成熟功能及时合入，每周把已完成部分接入最终 notebook。最后一周用于整理叙事、图表、完整重跑和排练。
- `main` 分支和 `main.ipynb` 是两回事：`main` 分支可以同时保存最终 notebook 和有复现价值的个人 notebook。

## 2. 目录结构与文件职责

```text
.
├── README.md
├── CONTRIBUTING.md
├── pyproject.toml               # 环境与依赖配置占位，后续补齐
├── .github/
│   └── pull_request_template.md
├── data/
│   ├── README.md                # 下载说明，进入 Git
│   ├── sample.csv               # 本地下载，不进入 Git
│   └── twcs/
│       └── twcs.csv             # 本地下载，不进入 Git
├── tools/                      # 稳定的 Python 函数与类
│   ├── __init__.py
│   ├── config.py
│   ├── data.py
│   ├── features.py
│   ├── analysis.py
│   ├── plotting.py
│   ├── search.py
│   ├── estimate.py
│   ├── evaluation.py
│   └── utils.py
├── notebooks/
│   ├── main.ipynb              # 最终分析与展示
│   ├── ws1_analysis.ipynb      # Weijun：数据、RQ1、等待时间
│   ├── ws2_analysis.ipynb      # Shujun：文本特征、RQ2
│   └── ws3_search.ipynb        # Lucas：检索、评估与演示
├── tests/
│   ├── test_data.py
│   ├── test_features.py
│   ├── test_analysis.py
│   ├── test_search.py
│   ├── test_estimate.py
│   └── test_evaluation.py
├── outputs/
│   └── .gitkeep                # 仅保留目录；运行结果在本地生成
└── docs/
    ├── project_charter.md
    └── week1_sprint_plan.md
```

当前工程骨架只创建文件：`.py`、测试文件及 `pyproject.toml` 均为空；四份 notebook 均无 cell、无输出。环境配置、初始化代码、功能实现和测试断言由后续任务补齐。占位文件存在不代表相关任务已完成。

| 文件 | 后续承载的内容 |
|---|---|
| `tools/config.py` | 统一路径、随机种子、参数与运行配置 |
| `tools/data.py` | 原始数据读取、字段校验、对话重建与案例库 |
| `tools/features.py` | 文本清洗、语言筛选、情绪与问题标签 |
| `tools/analysis.py` | 时段统计、事件窗口分析与不确定性计算 |
| `tools/plotting.py` | 图表绘制、样式与导出 |
| `tools/search.py` | TF-IDF、句向量检索与查询接口 |
| `tools/estimate.py` | 等待时间查找表与估计 |
| `tools/evaluation.py` | 检索与等待时间估计的评估 |
| `tools/utils.py` | 少量跨模块的通用辅助函数；业务逻辑放入对应模块 |
| `tests/test_*.py` | 对应模块的实际测试，优先使用小型合成数据 |

稳定逻辑即使暂时只有一个人使用，也可以进入 `tools/`。notebook 保留实验过程、函数调用、结果展示和解释，避免三个人各复制一份清洗或绘图逻辑。

## 3. 分工与接口

| 成员 | 主要负责 | 主要实验文件 |
|---|---|---|
| Weijun Fan（WS1） | 数据加载与案例库、RQ1、等待时间估计 | `notebooks/ws1_analysis.ipynb` |
| Shujun Chen（WS2） | 文本特征与标签、RQ2、标注协调与误差分析 | `notebooks/ws2_analysis.ipynb` |
| Lucas Liu（WS3 / PM） | 检索、效率实验、演示及最终 notebook 集成 | `notebooks/ws3_search.ipynb`、`notebooks/main.ipynb` |

模块允许共同贡献；修改其他成员正在使用的接口前，先在 Issue 或 PR 中说明。三个人都要能够解释自己交付的代码与结论。

- 案例库写入 `outputs/cases.parquet`，特征表写入 `outputs/features.parquet`，通过 `case_id` 关联。
- 字段、类型、时间单位与 UTC 约定以 [Week 1 数据接口](docs/week1_sprint_plan.md#5-数据接口约定) 为准。
- 公共函数需说明输入、输出、单位、缺失值处理和必要的前置条件。修改函数参数、返回结构或表字段时，在 PR 中列出受影响的下游调用。
- 原始数据路径统一为 `data/sample.csv` 和 `data/twcs/twcs.csv`；中间结果、图表、索引和缓存统一写入 `outputs/`。
- 路径以仓库根目录为基准。notebook 的初始化单元需统一设置工作目录，再导入 `tools`；不要依赖个人电脑的绝对路径，也不要假设 kernel 自动位于仓库根目录。初始化单元尚待实现。
- 随机种子统一为 `1007`，需要调整时记录原因和影响。

## 4. 每个任务的操作流程

### 4.1 从最新 main 开始

先确认当前改动已经提交或妥善保存。在仓库根目录运行：

```bash
git status
git switch main
git pull --ff-only origin main
git switch -c feat/ws1-case-builder
```

分支名按任务命名，例如 `feat/ws1-case-builder`、`feat/ws2-text-features`、`feat/ws3-tfidf`。修复用 `fix/<简述>`，文档用 `docs/<简述>`。一个分支对应一个明确任务；任务结束后再从最新 `main` 开新分支。

### 4.2 实验并整理成果

1. 在自己的 notebook 中探索，保留有价值的方法对比、图表与观察。
2. 将确认要复用或正式交付的逻辑整理到 `tools/` 对应模块，notebook 改为调用模块。
3. 为新增的核心逻辑补充实际断言或可复现验证；模块测试尽量使用小型合成数据。
4. 检查本次修改是否影响下游模块、数据接口和最终 notebook，并记录验证方式。

### 4.3 提交并推送任务分支

检查差异，只暂存本任务的文件。下面以 WS1 的任务为例：

```bash
git diff --stat
git add tools/data.py tests/test_data.py notebooks/ws1_analysis.ipynb
git diff --cached --stat
git commit -m "feat: build customer support cases"
git push -u origin feat/ws1-case-builder
```

在 GitHub 创建从任务分支到 `main` 的 PR，关联对应 Issue，并填写仓库的 PR 模板。尚未完成但需要讨论接口时，可以先开 Draft PR。

### 4.4 审查、合并与同步

- PR 至少由一名其他成员审查。交叉审查默认 WS1 → WS2、WS2 → WS3、WS3 与基础设施 → WS1，箭头表示“前者的改动由后者审查”。
- Lucas 负责协调合并和最终 notebook 集成；Lucas 自己的 PR 同样需要其他成员审查。
- 有实现代码的 PR 要说明验证命令与结果；纯文档、空骨架改动说明文件检查即可。没有实际测试时如实写明，不把空测试文件算作测试通过。
- 若 PR 期间 `main` 有更新，在任务分支内运行 `git fetch origin`，再运行 `git merge origin/main`，解决冲突后重跑受影响的验证并推送。
- 审查通过、必要验证完成后合入 `main`；已完成的远端任务分支可以删除。其他成员开始新任务前更新本地 `main`。

## 5. notebook 协作与最终集成

### 5.1 个人 notebook

- 默认只编辑自己负责的 notebook。需要修改别人的文件时，先在任务中协调。
- 有复现价值的个人 notebook 可以随 PR 合入 `main`；临时试错、重复版本和无关输出在提交前整理。
- 提交前检查 notebook diff，清理无用的大表格、重复日志、调试输出和异常堆栈。保留支持结论所需的适量图表与结果。
- 正式结果应来自可复现的执行顺序，避免依赖已经删除的 cell 或手工修改的隐藏变量。
- notebook 冲突由相关成员协调选取所需 cell，整理后重新运行受影响部分；不要直接把 JSON 冲突标记留在文件里。

### 5.2 最终 main.ipynb

- `notebooks/main.ipynb` 是最终分析与展示入口，按数据说明、RQ1、RQ2、检索与评估、结论等章节逐步完善。
- 各成员在任务 PR 中说明建议加入哪一节，提供可运行的模块调用、关键结果和解释；Lucas 负责接入与统一叙事。
- 大段数据处理、统计计算和检索实现保留在 `tools/`，最终 notebook 调用已有模块。
- 每周至少集成一次已完成的功能，并检查受影响章节是否可运行。任务未完成的章节可以暂留空白或标注进度。
- 最后一周从干净 kernel 执行完整流程，检查图表、文字结论和运行顺序，保留交付所需输出，再进行展示排练。

## 6. 数据与 PR 提交范围

- 每位成员按 [data/README.md](data/README.md) 下载数据；原始 CSV 和下载压缩包留在本地。
- `outputs/` 下的生成内容由代码重建，Git 仅追踪 `.gitkeep`。`.parquet`、`.npy`、`.npz` 文件也已忽略。
- 提交代码、必要 notebook、小型测试与相关文档。依赖变化在实际配置后同步到 `pyproject.toml` 和 README。
- PR 描述写清：解决的问题、修改范围、如何验证及结果、接口变化、最终 notebook 如何接入。未完成事项明确列为后续任务。
- 合并前确认：审查已通过；适用的验证已完成；相关文档已同步；原始数据和大文件没有被暂存。
