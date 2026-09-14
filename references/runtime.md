# 运行与编辑定稿

## 选择工作项目

优先使用用户指定的工作项目。若用户未指定项目，复制或使用本仓库的 `assets/runtime/`；只在目录同时包含 `config/settings.yaml` 与 `scripts/finalize_review.py` 时将其作为运行时根目录。不要假定任何用户目录或历史项目路径存在。

迁移时，包内 `assets/runtime/` 含可运行源码、来源配置与回归测试，不含历史新闻、数据库、虚拟环境或密钥。把该目录复制到用户允许的可写工作目录中的新子目录，不覆盖已有项目。在那个目录使用 Python 3.12+：

```sh
uv venv .venv
uv pip install --python .venv/bin/python .
.venv/bin/python -m unittest discover -s tests -q
```

不使用 editable 安装：此前该环境出现过安装成功但导入失败。修改源码后用 `uv pip install --python .venv/bin/python --reinstall --no-deps .` 刷新安装。网络依赖安装须遵循环境权限。不需要用户配置 LLM API，编辑由当前助手完成。

## 采集

先核查项目是否有 Webhook 配置，避免未授权外发；包内配置不含 Webhook 地址。所有命令从选定项目根目录执行：

```sh
.venv/bin/news-agent run --hours 24 --date YYYY-MM-DD --no-llm
```

替换为实际日期。日期字符串表示该日 08:00；昨日自然日使用今天零点的显式 ISO 时间，例如 `YYYY-MM-DDT00:00:00+08:00` 和 `--hours 24`。默认标题/输出文件名按截止日，不可误述成事件发生日。

退出码 0：规则质检通过，仍需核实内容；3：采集完成待编辑；2：所有来源失败；1：执行异常。读取 `outputs/截止日.json` 的 `stats` 和 `items`。数据库 `data/news.db` 的 `event_clusters` 存候选；按本次 `run_id` 查询 `id,payload_json`，禁止跨运行混入旧数据。

## 完整编辑稿

用当前 run_id 和真实事件 ID 写 `outputs/截止日.review.json`：

```json
{
  "run_id": "本次真实运行ID",
  "items": [
    {
      "id": "真实事件ID",
      "title": "核验后的中文标题",
      "summary": "标明发生时间或更新口径的中文事实摘要",
      "why_important": "与事实区分的影响判断",
      "sources": [{"name": "媒体名称及报道时间", "url": "https://原文地址"}]
    },
    {"id": "需要排除的真实ID", "exclude_reason": "具体排除理由"}
  ]
}
```

每个原草稿 ID 覆盖且只覆盖一次。可以添加本次数据库事件池已有、但自动排名遗漏的 ID。若网页补查发现的重大事件不在池中，定稿脚本不能凭空导入：以来源支持的“补充核验”正文独立交付并明确其不计入流水线统计，或在用户要求修复采集时完善采集后重跑；不得伪造数据库记录或悄悄漏掉。

```sh
.venv/bin/python scripts/finalize_review.py outputs/截止日.review.json
```

成功后核对 `quality_status=passed`、告警、run_id、最终条目数、数据库选中 ID 和 JSON 相同。脚本仅校验中文、链接形式和部分事件规则，不会自动打开链接证明事实。原始 `.draft.json`、编辑稿和最终 `.md/.json` 留在项目用于复查。

质量门槛无法通过时，不把草稿称为正式完成；交付可核验部分并具体解释缺口。不要把来源返回成功率当作事实正确率。
