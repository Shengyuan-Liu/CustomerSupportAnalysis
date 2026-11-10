# Repository Guidelines

## Notebook Collaboration Preferences

以下要求适用于当前 `notebooks/explore.ipynb` 的逐步 EDA 工作。这个阶段优先按用户的节奏在 notebook 内探索；下文的通用模块化要求不应触发未经用户要求的重构。

- **一次只写一个 block**：一个 block 指一个 code cell 或 Markdown cell。每次只新增或修改用户指定的一个 cell，完成后停下，不自动继续写后面的内容。用户明确要求搬移内容等涉及多个 cell 的操作时，只修改完成该请求所需的部分。
- **先汇报，再写入**：写之前用简单中文说明这个 cell 要做什么，等用户认可或调整后再写。用户明确要求写入已讨论的方案，或直接指定某个 cell 的内容时，视为本次授权，不重复询问；授权不延伸到后续 cell。
- **询问下一步不等于授权写入**：用户问“接下来做什么”时，只提出下一步建议，等用户确认后再修改 notebook。
- **使用课堂基础语法**：优先使用最朴素的 NumPy、pandas、Matplotlib 写法。需要核实课堂范围时，查看仓库根目录下相对路径 `../../lectures/` 和 `../../labs/`；未经核实，不声称某个方法课上讲过。确有需要时可以补充额外方法，但先简单解释用途。
- **代码简短易读**：使用顺序执行的 notebook 代码，能自然合并的简单步骤就合并。避免不必要的函数封装、类、复杂链式调用和额外依赖；不要为了压缩行数增加理解难度。
- **解释放在 Markdown**：分析目的、字段含义、方法说明和结果解读都写在 Markdown cell，不写成代码注释。Code cell 只保留实际执行的代码。
- **表达简单**：与用户用简单中文交流；notebook 文字沿用现有的简单英文风格，用户另有要求时按其要求。列举字段或发现时使用 `-` 列表，不使用 Markdown 表格。
- **输出要看得懂**：统计结果用带明确列名的 DataFrame，或带说明文字的 `print()` 展示，不单独输出没有含义说明的数字。DataFrame 输出可以保留表格形式；默认只展示需要的数量，不自行增加百分比。
- **保留用户的修改**：每次修改前读取 notebook 最新内容，沿用用户根据 proposal 建立的章节结构，在指定位置添加或修改。保留其他 cell、用户文字、代码和已有输出，不擅自重排整个 notebook。
- **结论依据实际结果**：写结果解读前查看已有输出，必要时只运行相关部分。区分“已写入代码”和“已运行验证”，不把尚未运行的结果写成已确认的发现。

## Project Structure & Module Organization

This project studies customer-support response times and similar-case retrieval. Follow [CONTRIBUTING.md](CONTRIBUTING.md) for collaboration details.

- `tools/`: shared Python logic; currently only an empty `__init__.py`. Add modules by responsibility, such as `data.py`, `features.py`, and `search.py`.
- `notebooks/`: workstream experiments (`ws1_analysis.ipynb`, `ws2_analysis.ipynb`, `ws3_search.ipynb`) and the final presentation, `main.ipynb`, integrated by Lucas.
- `tests/`: module tests; `test_data.py` is currently empty.
- `data/`: local datasets and tracked download instructions. `outputs/`: generated tables, figures, indexes, and caches.
- `docs/`: project charter and sprint plans; consult `week1_sprint_plan.md` for table schemas.

## Build, Test, and Development Commands

Run commands from the repository root. Dependency configuration and a pipeline entry point are not implemented; there is no build command yet.

- `python3 -m venv .venv` and `source .venv/bin/activate`: create and activate a local environment.
- `python3 -m jupyter lab notebooks/`: open notebooks after installing Jupyter and the required analysis dependencies.
- `python3 -m pytest -q`: run tests once pytest and actual tests are added; the empty placeholder provides no validation.
- `git diff --check`: check changes for whitespace errors and conflict markers.

Document dependency installation and execution commands in `README.md` during environment setup.

## Coding Style & Naming Conventions

Use four-space indentation, `snake_case` for modules/functions/variables, and `PascalCase` for classes. Document public functions' inputs, outputs, units, missing-value handling, and preconditions. No formatter or linter is configured.

Move stable logic into `tools/`; keep notebooks focused on experiments, calls, figures, and interpretation. Use repository-relative paths, seed `1007`, UTC timestamps, and minutes for waiting times.

## Testing Guidelines

Use pytest with `tests/test_<module>.py` and `test_<behavior>` functions. Prefer small synthetic datasets covering reply pairing, duplicate IDs, invalid waiting times, and retrieval behavior. No numerical coverage threshold is defined. Rerun affected notebook sections; execute the final notebook from a clean kernel before delivery.

## Commit & Pull Request Guidelines

History contains descriptive unprefixed commits; the documented convention is `<type>: <summary>`, e.g., `feat: build customer support cases`. Use `feat/`, `fix/`, or `docs/` task branches from updated `main`.

Target `main` through a PR with an Issue link and the repository template. Include scope, validation commands/results, interface changes, and final-notebook integration. Obtain another member's review; Lucas coordinates merges. Clean noisy notebook outputs before submission.

## Data & Configuration

Download CSVs according to `data/README.md`. Keep datasets, generated outputs, caches, and credentials out of Git. Join case and feature tables by `case_id`; document schema changes and downstream effects in the PR.
