
# 中国优先 · 全球新闻晨报

一个面向简体中文晨报的 Codex Skill 与本地新闻编辑流水线：以中国灾害、公共安全和民生为优先，同时覆盖政策金融、国际政治安全、科技与全球重大事件。它收集候选新闻、执行时效过滤与事件去重，并要求编辑核验后才能交付。

## 包含内容

- `SKILL.md`：Codex 使用的编辑与核验规范。
- `agents/openai.yaml`：Skill 的界面元数据。
- `assets/runtime/`：可独立运行的 Python 3.12+ 本地采集、聚类、质检与定稿工具。
- `references/runtime.md`：Skill 调用运行时的操作说明。
- `.github/workflows/tests.yml`：GitHub Actions 回归测试。

仓库不含 API Key、Webhook 地址、历史新闻、SQLite 数据库、日志或虚拟环境。

## 作为 Codex Skill 安装

将仓库克隆（或下载解压）到 Codex 的 skills 目录，目录名保持为 `china-news-morning-brief`：

```bash
git clone https://github.com/<你的用户名>/china-news-morning-brief.git \
  "$CODEX_HOME/skills/china-news-morning-brief"
```

然后在 Codex 中使用：

```text
使用 $china-news-morning-brief 生成今天的中文晨报，补查福建及全国重大灾情，核验来源后直接发送正文。
```

## 单独运行本地流水线

```bash
cd assets/runtime
uv sync --no-editable
uv run python -m unittest discover -s tests -v
uv run news-agent run --hours 24 --dry-run --no-llm
```

正式运行会在 `assets/runtime/outputs/`、`data/` 与 `logs/` 产生本地工作文件，它们默认不会被 Git 跟踪。退出码 `3` 表示采集完成但仍须人工/编辑核验，并非采集失败。

可选的 LLM 配置位于 `assets/runtime/.env`（先从 `.env.example` 复制）；真实凭据绝不可提交。Webhook 只有在显式设置 `NEWS_AGENT_WEBHOOK_URL` 时才会发送。

更详细的运行、审稿和定稿规则见 [运行时说明](references/runtime.md) 与 [编辑工作流](assets/runtime/EDITORIAL_WORKFLOW.md)。

## 上传到 GitHub

```bash
git init
git add .
git status
git commit -m "Initial release: China-first news morning brief"
git branch -M main
git remote add origin https://github.com/<你的用户名>/china-news-morning-brief.git
git push -u origin main
```

先在 GitHub 创建一个空仓库，且不要勾选 README、`.gitignore` 或 License 初始化选项，再执行最后三行。发布前务必查看 `git status`，确认未包含 `.env`、`data/`、`outputs/` 或 `logs/`。

## 许可

运行时采用 [MIT License](assets/runtime/LICENSE)。新闻源与链接文章仍受其原出版方的条款与版权约束；详见[第三方说明](assets/runtime/THIRD_PARTY_NOTICES.md)。

---
name: china-news-morning-brief
description: 生成中国优先的全球新闻晨报，核验实时来源、补查重大灾情、去重并交付完整中文简报。用于每日晨报、昨日新闻补报、漏报纠正和现有晨报流水线的编辑定稿。
---

# 中国优先的全球新闻晨报

交付可读、可核查的中文新闻正文，不只交付方案、代码、采集日志或文件链接。此 Skill 不绑定模型；由当前助手承担编辑核验，未配置额外 API Key 不是停止理由。

## 核心编辑标准

- 用户偏好：重要事件不限条数，不固定五条，不为凑数量收录软文。中国灾害、公共安全和民生优先，兼顾政策金融、国际政治安全、科技及全球重大事件；不要退化成仅 AI 行业资讯。
- 福建洪涝曾发生漏报，因此每期单独检查福建及全国主要灾情。福建没有当期重要进展时不要强行写入，更不能复用历史灾情。
- 根据当次真实时间确定日期。默认晨报截止北京时间当天 08:00，向前 24 小时；用户说“昨天”则使用昨日自然日并明确窗口，不悄悄沿用晨报窗口。
- 区分事件发生、报道发布和灾后新进展。严格检查源站日期、正文和时区；搜索摘要、网页 URL 中的日期及历史简报不构成事实核验。
- 伤亡数字注明地区、来源和通报时间。冲突无法消解则说明或省略数字，不混加不同灾害、台风或地区。允许同场灾害不同救援进展合并，但不跨地区误合并。
- 同一政策、事故合并一条；同一新闻机构的转载不能算独立确认。单一可靠来源可收录但注明，预测、指控及官员表态不得写成确定结论。
- 事实与影响判断分开。来源必须是真实打开并读过、直接支持结论的文章链接；遵守引用与来源字数限制。脚本通过仅证明结构与部分规则，不能代替事实核查。

## 执行路线

先阅读 [references/runtime.md](references/runtime.md)，选择已有项目或包内备用引擎。采集与定稿只写工作项目，不在 Skill 安装目录积累日志、数据库、环境或日报。

1. 运行当期采集，检查来源成功率、时间窗口与质检告警。退出码 3 是待编辑，不是网络失败，不要重复采集来解决中文问题。
2. 读取当期 JSON 和候选事件池；同时通过实时网页检索补查全国/福建灾情、重大公共安全、重要政策及国际重大新闻。27 个来源成功不代表完整覆盖。任何新闻都不可从过往对话或本包推断为当前事实。
3. 打开拟引用原文核验，编辑简体中文标题、事实摘要、重要性和来源。无法访问正文时换可靠来源；不得伪称已经核验。对遗漏事件按运行参考处理。
4. 对每个草稿 ID 明确保留或说明排除，执行定稿并检查文件与数据库一致。不要通过清空告警、伪造来源或随意标记 reviewed 绕过质量门槛。
5. 在当前对话发送正文，先列最重要的事情；普通事件简明，重大事件可展开。附完整文件链接与简短覆盖/质量摘要；单源或未解决缺口如实说明，不宣称绝无漏报。

## 调度与边界

Skill 本身不是定时任务，也不改变当前模型。只有用户要求安排或修复调度时才检查现有任务，优先更新已有晨报任务，保留时区和目标会话，不另建重复任务或 launchd。使用目标产品实际支持的自动化工具；不能把本地调度说成云端运行。外部发布、邮件和 Webhook 需在实际发送前按用户授权边界确认；默认只在当前对话交付。

不能联网时交付明确标注的阻塞说明或可核验部分，不把历史新闻冒充新晨报。仅真正的网络/覆盖失败可有限重试一次；仍失败就说明缺口，不无限循环。
