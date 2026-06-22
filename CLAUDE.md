# siyuan-copilot

这是一个 **AI 笔记副驾仓库**：用户不在这里手写代码，仓库纯粹用来「和 AI 讨论思源笔记」。
里面放的是给你（Claude）用的工具脚本和 API 文档——你来运行脚本读取笔记内容，然后陪用户讨论。

思源笔记（SiYuan）把内容存成自有的块格式、**不是普通 Markdown**，无法用 `Read` 直接打开，
一切内容都要走思源 kernel API（脚本已封装好）。

## 只读铁律（最高优先级）

**Claude 对思源笔记只有「读」权限，永远不得修改。** 具体：

- **禁止以任何方式新建 / 编辑 / 移动 / 删除用户的笔记、文档、块、笔记本**，也不得调用任何会写入思源的 kernel API。
- 即使用户主动要求修改笔记，也要拒绝并提醒：本仓库定位为「只读副驾」，写入操作请用户自己在思源客户端完成；Claude 至多能给出可照抄的大纲 / 草稿文本。
- **新增工具时只能做「读取类」工具**（查询、导出、检索、统计等只读端点）。不得编写任何调用写入 / 删除型思源 API 的脚本。

## 怎么读用户的思源笔记

当用户让你读 / 总结 / 检索 / 对比 / 分析他的思源笔记时，使用 **siyuan-read** Skill
（`.claude/skills/siyuan-read/SKILL.md`，里面有每个工具的详细用法）。

工具都在 `tools/`，从项目根用 `uv run` 跑，输出走 stdout（可直接读）：

| 想做的事 | 命令 |
|---|---|
| 看结构（有哪些笔记本/文档） | `uv run python tools/structure.py -f markdown` |
| 读某篇正文 | `uv run python tools/read.py "关键词"` → 多个候选再 `--id <id>` |
| 读整个主题（含子文档） | `uv run python tools/read.py --id <id> --tree` |
| 全文搜内容（不只标题） | `uv run python tools/search.py "关键词"` |
| 最近编辑了啥 | `uv run python tools/recent.py` |
| 各笔记本体量统计 | `uv run python tools/stats.py` |

前提：思源客户端在运行，且 `config.json` 填了 API Token。

## 项目结构

- `tools/` — 给 AI 运行的工具脚本
  - [tools/siyuan.py](tools/siyuan.py) — 共享 API 客户端：`post()` / `load_config()` / 查询助手，**新增能力时复用它**
  - [tools/structure.py](tools/structure.py) — 导出笔记本结构（树 / JSON / Markdown）
  - [tools/read.py](tools/read.py) — 读单篇或整个子树的 Markdown 正文
  - [tools/search.py](tools/search.py) — 全文检索笔记正文
  - [tools/recent.py](tools/recent.py) — 最近编辑的文档
  - [tools/stats.py](tools/stats.py) — 各笔记本文档数/字数统计
- `config.json` — URL / Token / 深度（gitignore，本地私有）；`config.example.json` 是提交进仓库的模板
- `output/` — 结构导出文件（gitignore）
- [docs/siyuan-api_zh_CN.md](docs/siyuan-api_zh_CN.md) — 思源完整 API 文档（扩展能力时查这里）

## 约定

- **新增的思源能力只能是只读工具**（见上方「只读铁律」）；在 `tools/` 下加脚本并 `import siyuan` 复用 `post()` / `load_config()`，不要另起 HTTP 客户端。
- 配置（URL / Token / max_depth）只放 `config.json`，不要再硬编码进代码。
- 脚本一律从项目根目录用 `uv run python tools/<name>.py` 运行（依赖和 venv 都在根目录）。
- 删除文件需先经用户同意；新增工具/功能可自行进行。
- 提交（commit）信息不要包含 Claude 的署名（不加 `Co-Authored-By: Claude` 等尾注）。

## 维护 API 文档

当用户说「**检查 API 更新**」（或「看看思源 API 有没有变」「校验工具」之类）时，调用 **check-api-update** Skill
（`.claude/skills/check-api-update/SKILL.md`）：拉取官方最新 `API.md` / `API_zh_CN.md`，与本地 `docs/` 比对，
并跑一遍 `tools/` 确认依赖端点未失效。文档/工具的改动先汇报 diff，经同意再动。
