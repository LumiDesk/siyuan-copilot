---
name: check-api-update
description: 拉取思源官方最新 API 文档，与本地 docs/ 比对，并校验 tools/ 各脚本依赖的端点是否仍然有效。当用户说「检查 API 更新」「看看思源 API 有没有变」「校验工具」等时使用。
---

# 检查思源 API 更新并校验工具

目标：把本地 `docs/` 里的思源 API 文档与官方最新版对齐，找出与本仓库 `tools/` 相关的变化，
并实际跑一遍工具确认没坏。**改动文档/工具前先汇报 diff，删改文件要经用户同意（见 CLAUDE.md 约定）。**

## 上游来源

官方仓库 `siyuan-note/siyuan` 的 master 分支：

| 本地文件 | 上游 raw 地址 |
|---|---|
| `docs/siyuan-api.md` | `https://raw.githubusercontent.com/siyuan-note/siyuan/master/API.md` |
| `docs/siyuan-api_zh_CN.md` | `https://raw.githubusercontent.com/siyuan-note/siyuan/master/API_zh_CN.md` |

## tools/ 实际依赖的端点（重点核查对象）

这些是工具真正调用的端点，API 变化只有牵涉到它们才会真正影响仓库：

| 端点 | 用在 | 关注点 |
|---|---|---|
| `/api/notebook/lsNotebooks` | `siyuan.py` / `structure.py` | 返回字段 `notebooks[].{id,name}` 是否还在 |
| `/api/query/sql` | `siyuan.py` 及多数脚本 | `blocks` 表字段：`type='d'`、`hpath`、`root_id`、`updated`、`created`、`content` |
| `/api/export/exportMdContent` | `read.py` | 入参 `{id}` → 返回 `content`（Markdown） |

> 端点清单可随时用 `grep -rEoh "/api/[a-zA-Z/]+" tools/ | sort -u` 重新生成，确保不漏新加的脚本。

## 执行流程

1. **拉新版到临时文件**（先不覆盖本地）：
   ```bash
   curl -s https://raw.githubusercontent.com/siyuan-note/siyuan/master/API.md       -o /tmp/siyuan-api.new.md
   curl -s https://raw.githubusercontent.com/siyuan-note/siyuan/master/API_zh_CN.md -o /tmp/siyuan-api_zh_CN.new.md
   ```
2. **比对差异**：
   ```bash
   diff docs/siyuan-api.md       /tmp/siyuan-api.new.md
   diff docs/siyuan-api_zh_CN.md /tmp/siyuan-api_zh_CN.new.md
   ```
   - 无 diff → 文档已是最新，跳到第 4 步只做工具冒烟测试。
   - 有 diff → 重点看是否触及上表 3 个端点的「入参 / 返回字段 / SQL 表结构」。无关变化（新增其它端点、文字润色）只需在汇报里提一句。
3. **更新本地文档**（征得同意后）：用 `cp /tmp/...new.md docs/...` 覆盖，使 `docs/` 跟上游一致。
4. **冒烟测试所有工具**（需思源客户端在运行且 `config.json` 有 Token）：
   ```bash
   uv run python tools/structure.py -f markdown   # 验 lsNotebooks + query/sql
   uv run python tools/stats.py                   # 验 query/sql 聚合
   uv run python tools/recent.py -l 3             # 验 query/sql 时间字段
   uv run python tools/search.py 测试 --docs      # 验 query/sql content 检索
   uv run python tools/read.py -s React           # 验候选查询
   ```
   任一脚本报错或返回空/字段缺失，对照第 2 步的 diff 定位是哪个端点/字段变了。
5. **汇报**：用一句话结论 + 三栏小结（文档是否有更新 / 是否触及依赖端点 / 工具是否全部跑通）。
   只有端点确实变化导致工具受影响时，才提出改 `tools/` 的具体方案，并先经用户同意再改。

## 注意

- 思源没开 / 没填 Token 时第 4 步会失败——这不代表 API 变了，先确认客户端状态再下结论。
- 不要凭文档 diff 就断言工具坏了；以第 4 步「实际跑通」为准（参考记忆 [[verify-notes-before-claiming]]：先验证再下结论）。
