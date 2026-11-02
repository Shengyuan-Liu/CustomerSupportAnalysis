# 数据下载与目录结构

本项目使用 Thought Vector 发布的 [Customer Support on Twitter（Kaggle）](https://www.kaggle.com/datasets/thoughtvector/customer-support-on-twitter) 数据集。原始 CSV 不随 Git 仓库分发，克隆仓库后需要自行下载。

## 下载步骤

1. 打开上面的 Kaggle 数据集页面，按页面提示登录账号。
2. 点击 **Download** 下载数据集压缩包，并解压。
3. 将解压得到的 `sample.csv` 放到本仓库的 `data/` 下，将完整数据 `twcs.csv` 放到 `data/twcs/` 下。如果解压后的目录层级不同，请按下面的结构整理。

所有路径都以仓库根目录为起点。最终应为：

```text
data/
├── README.md          # 本说明，提交到 Git
├── sample.csv         # 小样例，仅保存在本地
└── twcs/
    └── twcs.csv       # 完整原始数据，仅保存在本地
```

| 文件 | 用途 | 当前本地版本大小 |
|---|---|---|
| `data/sample.csv` | 开发调试、检查读取逻辑；不能替代完整数据进行正式分析 | 17,357 bytes，93 条记录（不含表头） |
| `data/twcs/twcs.csv` | 完整数据，用于对话重建、分析和检索 | 516,508,641 bytes，约 493 MiB |

## CSV 字段

两个 CSV 使用 UTF-8 编码，首行为表头，每条记录对应一条推文，字段顺序如下：

| 字段 | 含义 |
|---|---|
| `tweet_id` | 推文唯一 ID，用于关联消息 |
| `author_id` | 作者 ID；品牌为账号名，普通用户为匿名编号 |
| `inbound` | 是否为发给公司的消息，CSV 中为 `True` / `False` |
| `created_at` | 发帖时间，包含时区信息；分析时统一转换为 UTC |
| `text` | 推文正文 |
| `response_tweet_id` | 回复本条推文的 ID；可能有多个，以逗号分隔，也可能为空 |
| `in_response_to_tweet_id` | 本条推文所回复的推文 ID；没有时为空 |

正文可能包含逗号和换行，`response_tweet_id` 也可能包含逗号，请使用 `csv` 或 `pandas.read_csv` 等 CSV 解析器读取。小样例不保证包含全部被引用的推文。

## 检查文件位置与表头

在仓库根目录运行下面的命令，仅需 Python 标准库：

```bash
python3 - <<'PY'
import csv
from pathlib import Path

expected = [
    "tweet_id", "author_id", "inbound", "created_at", "text",
    "response_tweet_id", "in_response_to_tweet_id",
]
for path in (Path("data/sample.csv"), Path("data/twcs/twcs.csv")):
    with path.open(encoding="utf-8", newline="") as file:
        header = next(csv.reader(file))
    if header != expected:
        raise ValueError(f"{path}: 表头不匹配：{header}")
    print(f"OK: {path} ({path.stat().st_size:,} bytes)")
PY
```

`.gitignore` 会忽略 `data/` 下除本说明以外的文件和目录。下载的 CSV、压缩包和其他本地数据无需提交；分析产生的中间结果按项目约定写入 `outputs/`。
