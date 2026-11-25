# main.ipynb 解说

这份文档按 `main.ipynb` 的章节顺序，说明每一部分做了什么、用了什么方法、得到了什么结果。章节名、变量名和列名保持英文，和 notebook 一致，方便对照查找。

最后更新：2026-11-25

当前状态：
- Data Cleaning、RQ1、RQ2 和 RQ3：2026-11-25 做完前后一致性修改后，从新 kernel 完整重跑了一遍（61 个代码格，没有报错）。下面的数字都是这次的结果。
- Delivered System：还没有内容。

## 项目概览

- 背景：2017 年底，两个品牌的客服各遇到一次高峰。
  - Apple：iOS 11.1 键盘 bug（10 月 31 日发布，11 月 9 日修复），属于计划外的紧急事件。
  - Amazon：黑色星期五（11 月 24 日）到网络星期一（11 月 27 日），属于计划内的高峰。
- 研究问题：
  - RQ1：发推的时间和措辞会不会影响回复速度？（早上还是晚上；礼貌还是粗鲁）
  - RQ2：客服面对计划内高峰和计划外紧急事件时，反应有什么不同？
  - RQ3：相似案例搜索在什么情况下有帮助：关键词（TF-IDF）还是语义向量（embedding）？
- 最终交付（Delivered System）：一个系统，输入客户的问题，根据措辞、发推时间、是否高峰期给出预计等待时间，并展示几个相似的历史案例。

## 全文约定

- 时间：Data Cleaning 用 UTC；三个 RQ 都转成美东时间（America/New_York）。
- 等待时间：`response_minutes`，从客户发推到第一条品牌回复的分钟数。文字、小标题和图里统一叫 waiting time。
- 高峰事件的日期在 RQ1 和 RQ2 里一样（美东时间）：Apple 10/31–11/9，Amazon 11/24–11/27。
- “对话开头”（conversation start）指 `is_first_tweet` 为 True 的推文；RQ3 的“首次提问”（first question）范围更小，还要求是这个客户在数据里最早的推文。
- 随机种子：全部用开头定义的 `random_seed`（值是 42），包括 langdetect、RQ3 的抽样和随机基线。
- 图的颜色：
  - AmazonHelp 蓝色，AppleSupport 橙色（开头 import 那一格里定义为 `brand_colors`）。
  - 不分品牌的图用灰色。
  - 对比图：基准期和随机基线是灰色，事件期和 embedding 是紫色，TF-IDF 是绿色。
  - 虚线：黑色是参考线（0、月初、整体中位数），红色是重点线（截断点、平均差距、相似案例的中位数）。

## 数据文件

- `data/twcs/twcs.csv`：原始数据，2,811,774 条推文，7 列。
- `data/cleaned_df.pkl`：Data Cleaning 第 8 节的结果，555,973 条推文，比原始数据多 `related_brand`（相关品牌）和 `language`（语言）两列。Data Cleaning 第 9 节、RQ2 和 RQ3 会读它。
- `data/cases.pkl`：Data Cleaning 第 9–11 节的结果，215,852 个案例。一个案例是一条客户推文加上品牌对它的第一条回复。三个 RQ 都用它。列：
  - `case_id`：客户推文的 ID，每个案例唯一。
  - `brand`：`AmazonHelp` 或 `AppleSupport`。
  - `customer_id`、`customer_text`、`created_at`：客户 ID、推文内容、发推时间（UTC）。
  - `is_first_tweet`：客户推文是不是对话的开头。
  - `reply_tweet_id`、`reply_text`、`reply_at`：第一条品牌回复的 ID、内容、时间（UTC）。
  - `response_minutes`：等待时间（分钟），从客户发推到第一条品牌回复。
- `data/rq3_embeddings.npz`：RQ3 第 5 节的 embedding 缓存，和案例 ID 一起保存。下次运行时只有案例 ID 完全对得上才会读取，否则自动重新编码，不用手动删除。
- `data/search_cases.pkl`、`data/search_emb.npy`：RQ3 第 7 节保存的检索库，给最后的系统用。

运行方式：
- 在仓库根目录运行（代码里的路径都是 `./data/...`），依赖见 `requirements.txt`。
- 需要先准备好原始数据 `data/twcs/twcs.csv`。
- 语言检测大约要 6 分钟。`cleaned_df.pkl` 保存以后，后面的部分都直接读文件，不用每次从头跑。
- 每个 RQ 开头都重新读取 `cases.pkl`（RQ2、RQ3 还会读 `cleaned_df.pkl`）。所以 Data Cleaning 跑过一次以后，只要先运行开头 import 那一格，就可以单独运行任何一个 RQ。
- RQ3 第一次运行要下载 `all-MiniLM-L6-v2` 模型，并给约 11 万条推文编码，我们的电脑上用了约 8 分钟；之后会直接读取缓存。
- 有缓存时，整本从新 kernel 跑一遍约 7 分钟，其中语言检测约 6 分钟。

## Data Cleaning

### 1. Read Data

- 读入原始数据：2,811,774 条推文，每条一行，7 列：
  - `tweet_id`：推文 ID。
  - `author_id`：发推的客户或客服账号。
  - `inbound`：`True` 是客户发的，`False` 是客服发的。
  - `created_at`：发推时间。
  - `text`：推文内容。
  - `response_tweet_id`：回复这条推文的推文 ID，可能有多个。
  - `in_response_to_tweet_id`：这条推文回复的推文 ID。

### 2. Check Missing and Duplicate Data

- 方法：统计每列的缺失数，检查完全重复的行和重复的 `tweet_id`。
- 结果：
  - `response_tweet_id` 缺失 1,040,629 条，表示对话的最后一条。
  - `in_response_to_tweet_id` 缺失 794,335 条，表示对话的第一条。
  - 两个 ID 都缺失的推文是 0 条。也就是说每条推文都属于某个对话，开启对话的推文都至少有一条回复，所以这份数据看不出客户完全没人回复的情况有多常见。
  - 没有重复的行，也没有重复的 `tweet_id`。

### 3. Brand Statistics

- 方法：统计每个客服账号发的推文数，画出前 20 名。
- 结果：AmazonHelp 的客服推文最多，AppleSupport 第二。之后只研究这两个品牌。图里 AmazonHelp 是蓝色、AppleSupport 是橙色、其他账号是灰色，之后所有的图都用这两个品牌颜色。
- 给每条推文标上相关品牌（`related_brand`），方法是一轮一轮地传递：
  - 先给客服推文标上它自己的账号名。
  - 每一轮，上一轮新标上的推文把品牌传给它回复的推文（父推文）和回复它的推文。同一轮里两种来源都有时，用父推文的品牌。
  - 没有新推文可以标的时候停止。和任何客服推文都不相连的推文没有品牌。
- 结果：只保留和 AmazonHelp、AppleSupport 有关的推文，从 2,811,774 条减少到 611,145 条。

### 4. Check Data Types

- `tweet_id`、`in_response_to_tweet_id` 转成 `Int64`（可以存缺失值的整数）。
- `response_tweet_id` 原来是 `"5,7"` 这样的文字，转成整数列表。
- `created_at` 原来是文字，转成 UTC 时间，才能做减法算等待时间。

### 5. Check Time Range

- 方法：看最早和最晚的推文，按月、按日统计推文数。
- 结果：
  - 最早 2011-10-13，最晚 2017-12-03 23:13 UTC。2017 年 10 月以前每个月只有很少的推文，可能是后来的对话回复了它们才被收进来。
  - 2017 年 10 月 4 日，每天的推文数从几百条跳到 6,000 条以上，说明连续收集从这一天开始。
- 处理：只保留 2017-10-04 及以后的推文，611,145 → 606,758 条。两个高峰事件都在这之后；RQ2 拿每个事件和它之前的 14 天比较，这些日子也都在 10 月 4 日之后。
- 有三天的推文明显比前后几天少（每日推文图上的低谷）：11 月 9 日和 10 日只有约 5,000–6,000 条，前后的工作日都在 10,000 条以上；11 月 18 日不到 1,000 条。两个品牌在同几天一起减少，可能是数据没收全，但无法确认。保留这几天，但做了标记；RQ2 也做了去掉这三天的对比。

### 6. Detect Language

- 为什么要做：措辞分析、TF-IDF、embedding 都只适用于英文。
- 方法分两步：
  - 先按字符粗分：去掉 @提及和链接后，含平假名或片假名的标为 `japan`，含韩文的标为 `korean`，含汉字的标为 `china`，只有拉丁字母的标为 `latin`，都没有的标为 `other`。
  - 对 `latin` 推文用 `langdetect` 判断具体语言（随机种子 `random_seed`，值是 42）。4 个词以内的推文太短，直接算英文。
- 过滤规则：只删除非英文的客户推文，客服推文全部保留。原因是 `langdetect` 会把一些短的英文客服回复判错（例如 "Please keep us posted. ^MC"）。如果删掉了第一条回复，这个案例就会丢失，或者把后一条回复当成第一条，等待时间被算长。
- 结果：
  - 语言数量前五：en 519,059，japan 19,982，es 19,535，fr 18,370，de 10,318（包括客服推文）。
  - 过滤后剩 555,973 条，其中 36,914 条是没有 en 标签、但被保留的客服推文。

### 7. Fix Broken Reply Links

- 问题：前面的过滤删掉了一些对话中间的推文，有些推文的链接会指向已经删掉的推文。
- 方法：不删这些推文（很多还有用，比如父推文被删的客户推文可能仍有客服回复），只修正链接：
  - 父推文被删了：把 `in_response_to_tweet_id` 设为缺失，当作对话的开头。
  - `response_tweet_id` 里去掉已经删掉的 ID。
- 结果：45,708 条推文的父推文被删，39,749 个回复链接指向已删的推文，最终 555,973 条推文。父推文被删的数量较多，主要来自保留下来的非英文客服推文：它们回复的非英文客户推文被删了。这些客服推文不会被配进案例。

### 8. Save Cleaned DataFrame

- 保存为 `data/cleaned_df.pkl`。后面的步骤直接读这个文件，不用再跑语言检测。

### 9. Pair Customer Tweets with Replies

三个 RQ 都用这里生成的案例（一条客户推文加上品牌对它的第一条回复），所以这一步放在 Data Cleaning 里。

- 方法：
  - 读入 `cleaned_df.pkl`，分成客户推文和客服推文。
  - 用客户推文的 `tweet_id` 匹配客服推文的 `in_response_to_tweet_id`。
  - 只保留回复者是本案例品牌的配对；有多条回复时，取最早的一条。
  - 没有回复的客户推文不算案例，因为没有等待时间。
  - `is_first_tweet`：`in_response_to_tweet_id` 为空，就是对话开头。开头和追问都保留。
  - `response_minutes`：从客户发推到第一条回复的分钟数，也就是等待时间。
- 结果：配对 231,529 个 → 每条客户推文只留第一条品牌回复后 219,324 个。

### 10. Check Waiting Time

- 去掉等待超过 24 小时的案例。只有 75 个（约 0.03%），分散在很长的尾巴上（最长约 45 天），不像正常的第一次回复，但会拉高平均数。
- 去掉在数据结束（2017-12-03 23:13 UTC）前 24 小时内发的案例。这些推文还不到 24 小时数据就结束了，慢的回复没有被记录，等待时间会偏短。去掉以后，每个案例都有完整的 24 小时来等回复。
- 结果：219,324 → 去掉超过 24 小时的 219,249 个 → 去掉最后 24 小时的 215,852 个。等待时间中位数 21.7 分钟，平均 77.7 分钟，25% 和 75% 分位数是 9.2 和 69.9 分钟。

### 11. Save Cases

- 选出 10 列，改成清楚的列名，按时间排序后保存为 `data/cases.pkl`（每列的含义见上面的“数据文件”）。

## Effect of Posting Time and Wording (RQ1)

RQ1 问的是：发推时间和措辞会不会影响回复速度。开头重新读取 Data Cleaning 保存的 `cases.pkl`。比较时用中位数，因为少数很长的等待会把平均数拉高；两个品牌分开分析。

### 1. Effect of Posting Time

- 方法：
  - 把 UTC 时间转成美东时间（假设大部分客户在美国），取出发推的小时。
  - 把 24 小时分成四段：Morning (6-12)、Afternoon (12-18)、Evening (18-24)、Night (0-6)。
  - 每个品牌有自己的高峰事件：Apple 是 iOS 11.1 键盘 bug（10/31–11/9），Amazon 是黑五到网络星期一（11/24–11/27）。这一节的结果都算两遍：全部日期，以及去掉本品牌的高峰事件。高峰事件占 Amazon 案例的 7.8%（9,008 个），占 Apple 的 28.5%（28,446 个）。Apple 这段时间也包括 iPhone X 发售（11/3），原因不止一个，notebook 仍简称为 iOS bug 期间。
  - 画每小时的中位等待时间（纵轴是对数刻度），再按四个时段算中位数。
- 结果：
  - AmazonHelp 每个时段都很快，中位数 9–16 分钟。夜间 16.4 分钟，下午 9.3 分钟。去掉黑五期间几乎不变（9.5–16.7 分钟），黑五期间反而略快，所以这是 Amazon 平时的规律。
  - AppleSupport 很依赖发推时间。全部日期下，夜间中位 439 分钟（7 个多小时），下午 45 分钟，差约 10 倍。曲线在美东 23 点突然升高，之后慢慢下降，到 11 点左右恢复，说明 Apple 深夜基本不回复。
  - iOS bug 期间 Apple 白天明显变慢，但夜间几乎不受影响。去掉 bug 期间后，下午和晚上降到 32 和 24 分钟，夜间仍是 418 分钟（全部日期是 439 分钟）。所以夜间的延迟不是 bug 造成的；不算 bug 期间，夜间是下午和晚上的 13–18 倍。
- 限制：不知道每个客户的真实时区，统一按美东时间算。

### 2. Effect of Wording

- 方法：
  - 去掉 @提及和链接后，用正则表达式的词表给推文打标签：
    - `has_thanks`：含 thanks、thx、ty 等感谢词。
    - `is_polite_other`：含其他礼貌词，如 please、could you、sorry、hi。
    - `is_polite`：上面两者有一个就算。
    - `is_rude`：含脏话、侮辱或强烈负面的词，如 fuck、stupid、worst、useless、scam。
    - `\b` 保证整词匹配，所以 hate 不会匹配 whatever；`thank(?:s|you|u)?` 不会匹配 Thanksgiving。
  - 每条推文分到一个语气组：Polite（只有礼貌词）、Rude（只有粗鲁词）、Mixed（两种都有）、Neutral（都没有）。
  - 检查混杂因素：粗鲁推文是不是更多出现在夜间或 bug 期间，也就是 Apple 本来就慢的时候。然后在同一时段内（去掉 bug 期间）再比较不同语气。
  - 回归：每个品牌分别拟合 `log(response_minutes) ~ 措辞 + is_first_tweet + C(hour) + C(date)`。
    - 等待时间分布很偏，所以取对数。系数 b 表示等待时间大约变化 (e^b − 1) × 100%。
    - 小时和日期作为类别变量，控制每个小时、每一天的差别，包括 bug 和黑五这些很忙的日子。
    - 礼貌词拆成 thanks 和其他礼貌词，因为 thanks 常是反讽（"Thanks for ruining my presents"），也常出现在对话结尾。`is_first_tweet` 控制推文是不是对话开头。
    - 标准误用 HC1（允许方差不相等）。
  - 时间和措辞哪个更重要：比较回归里最慢小时和最快小时的差距与最大的措辞效应；再比较只用小时、只用措辞时的 R²。
- 结果：
  - 17.5% 的推文有礼貌词，14.6% 有粗鲁词。语气组：Polite 16%，Neutral 70%，Mixed 2%，Rude 13%。
  - 直接看中位数：Amazon 各语气差不多（11.6–13.3 分钟）；Apple 的粗鲁推文（95 分钟）和混合推文（96 分钟）比中性推文（66 分钟）慢。
  - 但 Apple 的粗鲁推文集中在它本来就慢的时候：bug 期间 25.0% 是粗鲁的，平时是 17.1%；夜间 22.3%，早上 17.4%。
  - 在同一时段内比较（去掉 bug 期间），差距明显变小：下午的礼貌、中性、粗鲁推文分别等 31、31、33 分钟。早上和晚上粗鲁推文仍然慢一些（107.1 vs 91.5 分钟，30.0 vs 22.0 分钟），但每个时段有 6 个小时，快的和慢的小时混在一起（比如 Evening 包括 Apple 几乎不回复的 23 点），所以剩下的差距可能还是发推时间造成的。后面的回归按同一小时、同一天来比较。
  - 回归（控制小时和日期）：
    - Apple：粗鲁 +0.3%（95% 置信区间 −0.7% 到 +1.3%，不显著）；thanks −2.8%（显著，约快 2 分钟）；其他礼貌词 +0.9%（不显著）。
    - Amazon：粗鲁 +4.8%（约慢 0.6 分钟）；thanks −15.3%（约快 1.8 分钟）；其他礼貌词 +12.8%（约慢 1.5 分钟）。三个都显著。
  - 时间和措辞比较：控制日期和措辞后，最慢的小时（午夜）比最快的小时慢，Amazon 约 2 倍，Apple 约 18 倍；措辞效应最大只有 15%（Amazon）和 3%（Apple）。只用小时能解释 log 等待时间变化的 5.3%（Amazon）和 48%（Apple），只用措辞两个品牌都只有 0.6%。

### RQ1 Summary

- 结论：两个品牌都是“什么时候发”比“怎么写”更重要。Apple 的差距非常大，Amazon 的差距中等。
- 要点：
  - Apple 深夜基本不回复，美东 23 点以后发的推文要等到第二天早上。这不是 bug 造成的。Amazon 全天都在回复。
  - 粗鲁推文只是看起来更慢：它们更多出现在夜间和 bug 期间。在同一小时、同一天比较，几乎没有差别。
  - 礼貌用语对 Amazon 有小的、方向相反的影响（各在 2 分钟左右以内），对 Apple 几乎没有影响。
  - Delivered System 估计等待时间时，发推小时是最有用的输入；对 Apple 来说，那天是否处在高峰事件期间也有用（具体日期没法用在未来的问题上）。措辞只会让预测变化几分钟。
- 限制：不知道真实时区；词表没有人工检查过，读不懂反讽，礼貌标签尤其容易错；数据只有约 9 周，而且有特殊事件；回归只能说明相关，不能说明因果。

## Influence of Peak Events (RQ2)

RQ2 问的是：客服面对计划内的高峰（Amazon 黑五到网络星期一，11/24–27）和计划外的紧急事件（Apple iOS 11.1 键盘 bug，10/31–11/9）时，反应有什么不同。事件日期和时区（美东时间）和 RQ1 一致，每个事件都和它之前的 14 天（基准期）比较。

用三个指标：
- 每天消息数：`cleaned_df` 里的全部客户推文，包括没被回复的，用来衡量需求。
- 等待时间：用 Data Cleaning 生成的 `cases`（第一条品牌回复、24 小时以内），看中位数和 90% 分位数。
- 回复率：对话开头的推文里，24 小时内收到品牌回复的比例。

### 1. Daily Customer Messages

- 方法：读入 `cleaned_df.pkl` 和 `cases.pkl`，转成美东时间。美东时间下数据的第一天和最后一天不完整，所以只保留 10/4–12/1。画两个品牌每天的客户消息数，标出两个事件期，再列出可疑日期前后的数字。
- 结果：两个品牌在同几天一起大幅减少。
  - 11/9 和 11/10 只有平常的一半左右（Amazon 1,336 和 1,258 条，前后的工作日是 2,400–2,900 条）。
  - 11/18 只有平常的十分之一左右（Amazon 230 条，Apple 242 条）。
  - Apple 在 11/9 的减少可能部分是真的（bug 在这天修好了），但 Amazon 同时减少，更可能是数据缺失。11/9 在 Apple 事件期里，11/10 和 11/18 在 Amazon 基准期里，所以第 4 节做了去掉这三天的对比。

### 2. Daily Waiting Time

- 方法：画每天的中位等待时间，纵轴用对数刻度（和 RQ1 一样）；另外列出 Apple 事件前后每天的中位数和星期几。
- 结果：
  - Apple 平时就常在周五、周六变慢（10/6–7、10/20–21、10/27、11/10–11）。事件期间连周初也慢：周一 11/6、周二 11/7 分别等 116.6 和 130.8 分钟，而周一 10/30、周二 10/31 只有 12.7 和 13.4 分钟。周五 11/3 和周六 11/4（276.5 和 237.2 分钟）是整份数据里最慢的两天。11/3 也是 iPhone X 发售日，所以 Apple 的事件期是 iOS 11.1 更新（包括键盘 bug）和一次计划内的新品发售叠在一起。
  - Amazon 从 11/24 开始变快，网络星期一之后仍然很快（11/28–12/1 中位 7.9 分钟），而这几天的消息量（每天约 3,600–3,900 条）比之前任何一天都多。所以变快不只是那 4 天。

### 3. Baseline vs Event

- 方法：
  - Apple：基准期 10/17–30，事件期 10/31–11/9（10 天）。Amazon：基准期 11/10–23，事件期 11/24–27（4 天）。
  - 天数从数据里直接数，每天消息数 = 客户推文数 ÷ 天数。
  - 对话开头的推文如果出现在 `cases` 里，就算 24 小时内被回复。数据里每条开启对话的推文都至少有一条回复，所以这个比例看不出完全没人理的情况；“没被回复”是指第一条品牌回复超过了 24 小时，或者只有别人回复。
- 结果：
  - Amazon：每天消息 2,101 → 2,937 条（约 +40%）；中位等待 11.2 → 8.4 分钟；90% 分位数 31.2 → 27.4 分钟；回复率 94.6% → 95.6%。
  - Apple：每天消息 1,727 → 3,584 条（约翻倍）；中位等待 46.5 → 153.8 分钟（约 3.3 倍）；90% 分位数 401 → 537 分钟；回复率 93.8% → 90.3%。

### 4. Robustness Checks

- 方法：去掉 11/9、11/10、11/18 三天再比较一次；只看对话开头推文的等待时间。
- 结果：
  - 去掉这三天：Amazon 消息增加约 26%（2,327 → 2,937），中位等待仍然下降（11.2 → 8.4 分钟）；Apple 中位等待 46.5 → 155.7 分钟。方向不变。
  - 只看对话开头：Amazon 10.9 → 8.0 分钟，Apple 57.4 → 159.4 分钟。

### RQ2 Summary

- 结论：Amazon 在计划内的高峰期间回复反而更快；Apple 在事件期间等待变成原来的约 3 倍。
- 限制：
  - 每个品牌只有一个事件，品牌之间的差别（人手、问题类型）和事件之间的差别分不开；而且 Apple 的需求增长（约 +108%）比 Amazon（约 +40%）大得多。
  - Apple 的事件期包括 iPhone X 发售（11/3），变慢不只是键盘 bug 造成的；没有检查哪些推文在讲 bug、哪些在讲新机。
  - 11/9、11/10、11/18 的数据可能不完整。
  - 回复率看不出完全没人理的客户。
  - Amazon 事件期 4 天里有 2 天是周末，基准期 14 天里有 4 天。

## Retrieving Related Cases (RQ3)

RQ3 问的是：客户问一个新问题时，从历史案例里找相似的案例，用关键词（TF-IDF）还是语义向量（sentence embeddings）更有帮助，在什么情况下。两种方法都按余弦相似度排序，只在同一品牌里找。第 1–3 节建检索库，第 4–5 节用两种方法找匹配，第 6 节比较，第 7 节把选中的方法做成给交付系统用的搜索函数。

### 1. Read Data

- 方法：读入 `cases.pkl`（Data Cleaning 第 9–11 节）和 `cleaned_df.pkl`，转成美东时间。检索库只放“首次提问”：`is_first_tweet` 为 True，而且是这个客户在数据里最早的推文（排除用新推文追问的情况，例如 "Just to be clear.. I have been home ALL day!"）。
- 结果：215,852 个案例 → 对话开头 129,352 个 → 首次提问 112,287 个（Amazon 47,745，Apple 64,542）。

### 2. Clean Text

- 方法：定义 `clean_text()`：还原 HTML 符号（数据里只有 `&amp;`、`&lt;`、`&gt;`），去掉 @提及和链接，转小写，合并多余空格；保留标点、停用词和 emoji（embedding 模型在完整句子上效果更好）。第 6 节清洗回复、第 7 节清洗新问题都用这个函数，规则一致。去掉 3 个词及以下的推文（看不出是什么问题）。
- 结果：112,287 → 110,359 个。

### 3. Split Data

- 方法：按美东时间切分：11/27 以前是 `history`（检索库），11/27–12/2 是 `query`（新问题）。每个客户只有一条首次提问，所以同一个客户不会同时出现在两边。只在同一品牌里搜索。
- 结果：history 99,245 个（Amazon 40,262，Apple 58,983）；query 11,114 个（Amazon 6,597，Apple 4,517）。query 期间从 Cyber Monday 开始，Amazon 的问题比 Apple 多，和 history 正好相反。

### 4. Method A: TF-IDF

- 方法：
  - 只用 history 拟合 `TfidfVectorizer(stop_words="english", min_df=2)`，再转换 query。`min_df=2` 表示只在一条 history 推文里出现的词被丢掉。
  - 向量长度为 1，所以点积就是余弦相似度；其他品牌的分数设为 −1，保证不会被选中。
  - `get_chain(case_id)`：从第一条推文开始，不断加入回复这段对话的推文，按时间排序，用来展示完整对话（时间用美东时间）。
- 结果：
  - 词表有 14,673 个词。
  - 示例问题（随机抽到的一条 Apple 问题）"my music app keeps crashing on my iphone 6S. can i get some help with this"：前 5 个里 4 个是 Music app 闪退，另 1 个是 iPhone 整体闪退（"My IPhone 7 keeps Crashing what should I do?"）。
  - 1,000 个抽样 query（Amazon 554 个，Apple 446 个）的最佳匹配相似度中位数：Amazon 0.44，Apple 0.45。

### 5. Method B: Sentence Embeddings

- 方法：
  - 模型 `all-MiniLM-L6-v2`（sentence-transformers），把每条推文变成 384 个数字，表示意思而不只是用词。例如 "package never arrived" 和 "didn't receive my order" 没有共同的词，但向量很接近。
  - 模型在这里加载一次，叫 `embedder`，第 6、7 节复用。
  - 向量连同案例 ID 存到 `data/rq3_embeddings.npz`，ID 对得上才读取，否则重新编码。
  - 用 NumPy 归一化后做矩阵乘法：每个品牌一次矩阵乘法就算出所有 query 的分数，不用循环。
- 结果：
  - 同一个示例问题，前 5 个都是 Music app 闪退，前 3 个还都提到 iPhone 6，和提问的 iPhone 6S 接近。
  - 1,000 个 query 的搜索只要约 0.1 秒。
  - 最佳匹配相似度中位数：Amazon 0.73，Apple 0.73。两种方法的分数尺度不同，不能直接比较。

### 6. Evaluate

- 方法：数据里没有“哪些案例相似”的标签，不能直接算准确率，所以用同样的 1,000 个 query 做间接比较：
  - 6.1 两种方法前 5 个结果重合多少。
  - 6.2 回复相似度：相似的问题应该得到相似的回复，所以比较 query 的真实回复和前 5 个匹配的回复。
    - 加入随机基线（同品牌随机 5 个案例），因为 "send us a DM" 这类模板回复会让任意两个案例都显得相似。
    - 用 TF-IDF 和 embedding 两个“评委”打分，避免评委偏向自己的方法。
    - 回复先去掉客服签名（如 `^KH`），再用 `clean_text()` 清洗。
  - 6.3 逐个 query 看两种方法的差距，并阅读差距最大的两个例子。
  - 6.4 按品牌和问题长度（4–10 词、11–20 词、21 词以上）分组，看哪种方法在哪类问题上更好。
- 结果：
  - 两种方法找到的案例很不一样：前 5 个平均只有 0.49 个相同，645 个 query 完全不重合。
  - history 的回复里约 20%（19,615 / 99,245）是重复使用的模板。
  - TF-IDF 评委：随机 0.045，TF-IDF 0.081，embedding 0.096。Embedding 评委：随机 0.291，TF-IDF 0.367，embedding 0.399。
  - 逐个 query：embedding 赢 569 次、TF-IDF 赢 408 次（TF-IDF 评委）；628 次对 371 次（embedding 评委）。平均差距 0.032，634 个 query 的差距在 ±0.1 以内。
  - 例子（两个方向差距最大的 query）：
    - Embedding 赢得最多：Amazon 客户要更多 Kannada 电影。TF-IDF 第 1 个也是 Kannada 请求，但第 2、3 个只是含有 "films" 和 "available"（Prime 上没有某部电影、某个地区看不了电影）；embedding 前 3 个都在要 Kannada 电影，它们的回复和真实回复一样（"stay tuned as we expand our content selection"）。
    - TF-IDF 赢得最多：Apple 客户创建 Apple ID 时不能选 “None”、只能填信用卡，还说已经发了 DM。真实回复只是确认收到 DM。TF-IDF 找到的是“请回复我的 DM”这类推文，回复也只谈 DM；embedding 找到的是 Apple ID 和付款问题，第 3 个就是同一个问题（没有 “None” 付款选项），但这些回复在讲问题本身。所以这次 TF-IDF 赢是因为真实回复的用词，不是因为匹配更有用。
  - 分组：
    - embedding 在每一组、两个评委下都更好（例如 Amazon 赢 352 对 201，Apple 赢 276 对 170）。
    - 两个品牌在 embedding 评委下差不多（0.033 对 0.031）；TF-IDF 评委下 Apple 的优势更大（0.020 对 0.012）。
    - 按长度：短问题（4–10 词）在两个评委下优势都最大（0.050 和 0.028），但只有 113 个；中等长度（11–20 词）最小（0.027 和 0.007）。
    - 没有哪一组 TF-IDF 平均更好。TF-IDF 在各组仍赢 31–39% 的 query，其中一部分是回复用词造成的（例如上面 Apple ID 的例子）。组间差别很小，没有做显著性检验。
    - 注意：上一版用种子 1007 抽样时，优势最大的是中等长度的问题；换成现在的种子 42 后变成了短问题。所以长度分组之间谁大谁小并不稳定，展示时不要当成结论。

### 7. Search Function for the Delivered System

- 方法：
  - 评估结束后，把 query 期间也加入检索库（表格用 `pd.concat`，向量用 `np.vstack`），保存为 `data/search_cases.pkl` 和 `data/search_emb.npy`。
  - `search(text, brand, k=5)`：用 `clean_text()` 清理新问题 → 用 `embedder` 编码 → 返回同品牌最相似的 k 个案例，包括分数、案例 ID、客户推文、第一条回复和等待时间（列名 `Waiting Time (min)`）。
- 结果：
  - 检索库有 110,359 个案例。
  - "My package says delivered but I never received it"（Amazon）：最高分 0.86，前 5 个里 4 个是“显示已送达但没收到”，另一个是“没收到包裹”。
  - "Battery drains very fast after updating to iOS 11"（Apple）：最高分 0.91，前 5 个都是升级后耗电的问题。
  - 第二个问题最相似的 20 个案例，等待时间中位数 114.2 分钟；检索库里全部 Apple 首次提问是 90.2 分钟。怎样把相似案例变成预测的等待时间，留给交付系统。

### RQ3 Summary

- 结论：语义向量找到的相似案例比关键词略好，两个品牌、短中长问题都是这样，但优势很小。
- 决定：交付系统用 embedding（第 7 节的 `search()`），1,000 个 query 只要约 0.1 秒，够快。TF-IDF 作为基线：不用下载模型，容易解释，匹配结果和问题用词相同。
- 限制：
  - 回复相似度是间接指标：同一个问题的回复可能用词不同，而且约 20% 的回复是模板。6.3 里 TF-IDF 赢 Apple ID 那个问题，只是因为真实回复只确认了 DM。
  - embedding 评委和 embedding 方法是同类模型，可能偏向它；TF-IDF 评委的方向一致，但差距更小。
  - 只测了最后 6 天的 1,000 个 query，匹配结果没有人工检查。
  - 6.4 的组间差别很小，没有做显著性检验。

## Delivered System

- 还没有内容。

## 已知问题和待办

- RQ3 可以补充：人工抽查一部分匹配结果（例如 30 个 query，每种方法前 3 个，不看方法名标“相关/不相关”），这是“到底有没有帮助”最直接的证据。
- RQ1 的 Limitations 还没写三点：数据缺口（11/9–10、11/18）、`is_first_tweet` 约 1.4% 标错、4 个词以内的推文直接算英文。
- 写 Delivered System 时要注意：
  - `cases` 在 RQ2、RQ3 开头被重新读取并覆盖，最后内存里只剩 RQ3 的首次提问；RQ1 加的 `hour`、`in_event`、`tone` 等列需要重新算（RQ1 的回归结果 `models` 还在）。RQ2 的对比表 `summary` 也被 RQ3 的同名变量覆盖了。
  - RQ1 的 `flag()` 依赖全局变量 `text`，而 RQ3 第 7 节的循环把 `text` 改成了一个字符串，所以不能直接用 `flag()` 判断新问题的用词。
- 随机种子现在全部用 `random_seed`，值仍是 42；AGENTS.md 写的是 1007。改成 1007 会让 langdetect 的结果和 RQ3 的抽样都变，需要整本重跑并重写数字。
- RQ2 可以补充的分析（审查时在 notebook 外试算过，还没加进 notebook）：
  - 控制发推时间：按时段比较，Amazon 事件期每个时段都更快或差不多，Apple 每个时段都更慢；回归（控制小时和星期几）Amazon −22%，Apple +182%。
  - Amazon 换基准期：用 10/17–30 作基准，高峰期消息只多约 15%，但等待时间仍然更短（13.2 → 8.4 分钟）。
  - 用关键词看 Apple 客户在说什么：提到键盘或自动更正的比例最高约 8%（11/2–4），提到 iPhone X 的比例在 10/27 和 11/3 有高峰。
  - 每天新开的对话数比全部消息更能代表需求（回复慢时客户会追问）。
- RQ1 Limitations 写词表 “not checked by hand”，但之前做过小样本人工检查（`data/rq1_manual_check.csv`：粗鲁标签约 96% 正确，礼貌标签约 69%）。要不要补充，待定。
- RQ1 回归的样本不独立：62.6% 的案例来自有多个案例的客户。在 notebook 外用按客户聚类的标准误检查过，p 值和置信区间几乎不变，结论不受影响；可以考虑在 notebook 里加一句说明。
- Data Cleaning：
  - 父推文被删的推文会被当成对话开头，`is_first_tweet` 约有 1.4% 标错。
  - 4 个词以内的推文直接算英文，会混进少量非英文短推文。
- 格式：RQ1 的四级小标题没有编号，RQ3 第 6 节的小标题有编号（6.1–6.4），这次没有统一。
- `data/rq1_manual_check.csv` 已经不被 main 使用，可以考虑删除。
- `requirements.txt` 里的 `vaderSentiment` 已经没有 notebook 使用。

## 更新记录

- 2026-11-24
  - 把 RQ2（`piplines/analysis.ipynb`）和 RQ3（`piplines/modelling.ipynb`）整合进 main，只改了数据路径和标题层级。
  - Data Cleaning：语言过滤改成只删除非英文客户推文，保留全部客服推文；RQ1 第 1 节加入“去掉数据结束前 24 小时内的案例”。重跑了 Data Cleaning 和 RQ1 第 1 节。
  - RQ1 第 2、3 节和 Summary 换成 PR #4 的简化版：去掉 weekday、"!!"、shouting、VADER 和人工检查，加入“去掉高峰事件”的对比和时间与措辞的比较。用最新数据重跑，并更新了结论里的数字。
  - 新建这份文档。
  - 按审查结果修改了 Data Cleaning 和 RQ1 的文字（只改 Markdown，不用重跑）：修正错别字和语法；说明两个 ID 都缺失的推文为 0 意味着什么；改写品牌传递算法的描述；在时间范围部分标出 11 月 18 日；Amazon 的 “every hour” 改成 “every time of day”；补充同一时段内比较的说明；Summary 第 4 条不再把日期当作预测输入。
  - 修正本文档里 `cases.pkl` 使用者的描述：RQ1 和 RQ3 用它，RQ2 没有用。
  - 重写 RQ2：改用 RQ1 的 `cases` 和同样的事件日期（美东时间），回复率只看对话开头的推文；分成 4 个小节（每天消息数、每天等待时间、基准期和事件期对比、稳健性检查）和 RQ2 Summary；用最新数据重跑。新发现 11/9–10 可能有数据缺口，并写明 Apple 事件期和 iPhone X 发售重叠、Apple 常在周五周六变慢。从这次起 RQ2 也使用 `cases.pkl`。
  - 为了前后一致：Data Cleaning 第 5 节补充 11/9–10 的说明；RQ1 定义 Apple 事件期时注明和 iPhone X 发售重叠，“The iOS bug made...” 改为 “The iOS bug period made...”。
- 2026-11-25
  - 重写 RQ3 并用最新数据重跑：加入 RQ3 简介和 “## RQ3 Summary”；改按美东时间切分（query 期间是 11/27–12/2）；文本清洗写成 `clean_text()`，问题、回复和新问题共用；模型在第 5 节加载一次（`embedder`），`search()` 不再依赖 6.2 里的变量；embedding 缓存改成连同案例 ID 一起保存，对不上时自动重新编码；新增 6.4 按品牌和问题长度分组的比较；删掉重复的 import 和多余的时间转换，修正过时的引用（`cleaning2.ipynb`、`prediction.ipynb`）；图的配色和其他章节的品牌颜色保持一致。
  - 开头的 import 加入 `html`、`os`、`time`（原来散在 RQ3 里）。
  - 前后一致性修改，然后从新 kernel 完整重跑整本 notebook（61 个代码格，没有报错）：
    - 结构：建案例的三步（配对、检查等待时间、保存 `cases.pkl`）连同说明，从 RQ1 第 1 节挪到 Data Cleaning 第 9–11 节。RQ1 开头改为读取 `cases.pkl`，小节编号改为 1–2；RQ2、RQ3 里 “cases from RQ1” 的引用改为 Data Cleaning。
    - 叫法：
      - 开头的 “Delivery” 和 RQ1 Summary 的 “Prediction step” 都改为 Delivered System。
      - 开头的 RQ3 问题改成和 proposal、RQ3 开头一致（“When does similar-case search help?”）。
      - RQ1 标题改为 “Effect of Posting Time and Wording (RQ1)”，“Effect of Words” 改为 “Effect of Wording”。
      - Data Cleaning 第 7 节 “Final Deleting” 改为 “Fix Broken Reply Links”，因为这一节只修正链接，不删推文。
      - 文字、小标题和图里的 response time 统一为 waiting time；RQ2 表格列名里的 “first tweets” 改为 “conversation starts”；`search()` 输出的 “Response Minutes” 列改为 “Waiting Time (min)”。
    - 描述：
      - Data Cleaning 第 5 节“Oct 4 – Oct 30 可以当作平时”改为 RQ2 实际用的“事件前 14 天”。
      - RQ1 的 “every result twice” 改为只指发推时间这一节；“The regression below” 改为 “The regression (next point)”。
      - 英式拼写（analyse、behaviour、labelled）改为美式；“Member” 改为 “Members”。
    - 格式：
      - RQ1 的两个 Markdown 表格（Tone Groups、RQ1 Summary）改成 `-` 列表，Summary 的编号要点也改成 `-` 列表。
      - RQ1 的输出改成带清楚列名的 DataFrame 或带说明的 `print()`，数字不变。
      - `find_related_brands()`、`time_of_day()`、`flag()` 的说明改成 Input/Output 写法，和 RQ3 一致。
    - 随机种子：RQ3 的 `random_state=1007` 和 `default_rng(1007)` 改为 `random_seed`（42）。示例问题、1,000 个抽样 query 和随机基线因此换了一批，RQ3 的数字和例子按新结果重写，结论方向不变。
    - 图的颜色：开头定义 `brand_colors`，所有分品牌的图都是 Amazon 蓝色、Apple 橙色（包括品牌统计图、回归置信区间图和 RQ3 最后一张图）；不分品牌的图改为灰色；参考虚线用黑色，重点虚线用红色，不再用橙色画参考线。
    - 清理输出：去掉 torch 警告、HF Hub 警告和模型加载进度条，语言检测的进度条只留最后一行。
    - 核对：Data Cleaning、RQ1、RQ2 的数字和改动前完全一样。
