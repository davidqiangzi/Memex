<div align="center">

<br />

<img src="dashboard/claude_character.svg" width="100" alt="Memex character" />

<h1>Memex</h1>

<p><strong>一个会自己整理的个人知识库。</strong></p>

<p>
放入资料，Claude 负责归档、总结、交叉链接和维护引用。<br/>
你的知识会持续复利。
</p>

<p>
<a href="README.md"><img alt="English" src="https://img.shields.io/badge/English-README-111?style=flat-square" /></a>
&nbsp;
<a href="README-ko.md"><img alt="한국어" src="https://img.shields.io/badge/한국어-README-111?style=flat-square" /></a>
</p>

<br />

<p>
<em>“Obsidian 是 IDE，Claude 是程序员，wiki 是代码库。”</em>
</p>

</div>

---

## 项目定位

Memex 是一个本地优先的 LLM Wiki / Obsidian Vault。它不把原始文档简单塞进向量数据库后每次重新检索，而是让 Claude 把资料持续整理成可读、可链接、可追溯的 Markdown wiki。

核心数据流：

```text
raw/ 原始资料
  ↓ ingest
wiki/ Claude 维护的知识层
  ↓ query / lint / reflect / write
可持续演化的个人知识库
```

当前核心组件：

- `raw/`：不可变原始资料，只读或追加，不应直接修改。
- `wiki/`：Claude 维护的 Markdown 知识库。
- `wiki/index.md`：内容目录。
- `wiki/log.md`：操作时间线。
- `ingest-reports/`：每次 ingest 的 WHY 报告。
- `dashboard/`：本地 Web Dashboard，入口是 `dashboard/server.py`。
- `mcp-server/`：MCP 服务，让 Claude Code / Claude Desktop 从外部访问 wiki。
- `projects/`：多项目模式，每个主题一个隔离知识库。
- `projects.json`：项目注册表和当前 active project。

---

## 快速开始

```bash
git clone https://github.com/cmblir/memex.git
cd memex
python dashboard/server.py
```

默认访问：

```text
http://127.0.0.1:8090
```

Dashboard 默认绑定到 `127.0.0.1:8090`，即仅本机访问。

如需指定地址或端口：

```bash
MEMEX_HOST=127.0.0.1 MEMEX_PORT=8090 python dashboard/server.py
```

---

## 本地部署与调试

环境要求：

- Python 3.10+
- git
- Claude Code CLI
- 浏览器
- Obsidian 可选，但推荐

常用检查命令：

```bash
curl http://127.0.0.1:8090/api/status
curl http://127.0.0.1:8090/api/projects
curl http://127.0.0.1:8090/api/index/status
curl http://127.0.0.1:8090/api/raw/integrity
curl http://127.0.0.1:8090/api/claude/diagnose
```

常用环境变量：

```bash
MEMEX_HOST=127.0.0.1
MEMEX_PORT=8090
CLAUDE_TIMEOUT=1200
CLAUDE_QUICK_TIMEOUT=30
CLAUDE_TOOLS=Edit,Write,Read,Glob,Grep
```

`MEMEX_HOST=0.0.0.0` 或 `MEMEX_HOST=::` 只应在明确需要服务器部署时使用。Dashboard 有文件写入和 Claude CLI 调用能力，不应直接暴露到公网。

---

## MCP 设置

安装 MCP server：

```bash
bash mcp-server/install.sh
```

安装脚本会创建 `mcp-server/.venv`，安装 `mcp` 依赖，并输出 Claude Code / Claude Desktop 的配置示例。

MCP 暴露的能力包括：

- 读取项目、页面、目录树、raw source 列表
- 搜索 wiki
- 新增 raw source
- 创建/更新 wiki 页面
- 创建目录
- 提交 git commit

Dashboard 本身不依赖 MCP；MCP 是可选增强入口。

---

## VPS 迁移建议

Memex 仍然建议保持 local-first。未来迁移到 VPS 时，推荐架构是：

```text
VPS
├─ Memex repo
├─ Python dashboard bound to 127.0.0.1
├─ Claude Code CLI authenticated as service user
├─ git remote backup
└─ Caddy / Nginx / Tailscale / WireGuard for remote access
```

最低运行环境：

- Python 3.10+
- git，保留完整仓库历史
- Node/npm，仅用于安装 Claude Code CLI
- Claude Code CLI 已完成认证

必须迁移或备份：

- `.git/`
- `wiki/`
- `raw/`
- `projects/`
- `projects.json`
- `ingest-reports/`
- `reflect-reports/`
- `CLAUDE.md`
- `templates/`
- `.dashboard-settings.json`，如果存在
- `query-log.jsonl`，如果存在
- 各项目下的 `.settings.json` 和 `CLAUDE.md`

不要直接把 Dashboard 暴露到公网。推荐保持：

```bash
MEMEX_HOST=127.0.0.1 MEMEX_PORT=8090 python dashboard/server.py
```

然后通过带认证的反向代理或私有网络访问。

---

## 当前搜索与后续向量数据库预留

当前项目没有向量数据库。搜索采用轻量 TF-IDF over Markdown。

推荐升级路线：

```text
当前：Markdown + TF-IDF
  ↓
中期：SQLite FTS / BM25
  ↓
后续：vector search / hybrid search
```

原则：

- `wiki/` 始终是 canonical knowledge layer。
- 向量数据库只作为可重建索引，不作为唯一事实来源。
- 默认优先索引 `wiki/`，必要时再索引 `raw/`。
- 未来可通过统一 search/index 模块平滑替换底层检索实现。

---

## 本次部署准备变更说明

本次变更面向本地部署调试和未来 VPS 迁移准备：

- Dashboard 新增 `MEMEX_HOST` / `MEMEX_PORT` 配置。
- Dashboard 默认只绑定 `127.0.0.1`，避免意外公网暴露。
- 非本地绑定时启动日志会给出安全警告。
- Dashboard 页面/目录相关 API 增加 wiki 路径边界校验，拒绝 `../raw` 等路径穿越。
- MCP `list_pages` / `create_page` 增加路径边界校验。
- raw integrity 检查覆盖 legacy raw 和多项目 raw。
- git 检测改用 `git rev-parse --is-inside-work-tree`，兼容 worktree。
- README 补充本地调试、VPS 迁移和未来搜索升级说明。

---

## 资源消耗判断

Memex 本身资源占用较低：Dashboard 是 Python stdlib HTTP server，wiki 是 Markdown 文件，MCP 只有少量依赖。主要耗时和成本来自 Claude CLI 调用，尤其是 ingest、lint、reflect 等长任务。

个人本地使用推荐：

- 2GB+ 内存
- 5GB+ 磁盘，具体取决于 raw/assets 体积
- 稳定网络和 Claude Code CLI 认证

VPS 推荐：

- 1-2 vCPU
- 2GB RAM
- 20GB+ 磁盘
- 私有访问或反向代理认证

---

## License

MIT License.
