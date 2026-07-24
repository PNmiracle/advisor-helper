---
name: phd-supervisor-selector
description: 博士导师筛选与表格管理工具。面向留学机构，为不同方向的学生快速建立可核验的导师列表。搜索各大学官方页面，判断导师是否具备 PhD 指导资格，填写表格各列。主力支持 Vika 在线表格直接 CRUD，Excel 作为可选回退模式。
agent_created: true
---

# PhD Supervisor Selector（博士导师筛选器）

## 概述

为博士申请者建立有据可查的导师列表。搜索各大学官方页面，判断导师是否具备 PhD 指导资格，填写表格各列，将不合格或高风险候选人排除在主表之外。

**主力模式：Vika（在线表格直接 CRUD）。Excel 作为可选回退模式。**

## Optional Companion Skill

**tavily-search-pro** — AI-powered search platform that acts as **L4 fallback** when L1-L3 access methods all fail (WAF blocks curl+browser+search). Provides:
- **extract**: Server-side page content extraction (bypasses Cloudflare/Incapsula WAF)
- **search**: Alternative search engine with clean, unencrypted result URLs
- **crawl/map**: Site structure discovery for blocked university sites
- **research**: Deep department analysis with citations

Requires `TAVILY_API_KEY` environment variable. If installed, search automatically escalates to Tavily when all other access methods are blocked. See `references/search-techniques.md` (L4 section) and `references/school-strategies.md` (L4 layer).

## 搜索编排（首先阅读）

开始搜索任务前，阅读 `references/search-orchestrator.md`。它定义了：

- **状态追踪**：续传中断的搜索，跳过已完成的学校
- **智能优先级**：按预期产出（P0-P3 层级）对学校排序
- **分阶段策略**：先快速扫描所有学校（Pass 1），再深入验证有希望的学校（Pass 2）

---

## 数据源检测

收到任务时，检测数据源：

1. **用户提供 Vika 分享链接**（如 `https://vika.cn/share/shrXXX/dstXXX/viwXXX`）：直接通过 Fusion API 操作 Vika 表格
2. **用户上传 `.xlsx` 文件**：操作本地电子表格
3. **用户同时提供**：优先 Vika 做 CRUD，Excel 做参考/填充源

---

## Excel 模式（可选）

当用户上传 `.xlsx` 文件而非 Vika 链接时，使用 Excel 模式。详见 `references/spreadsheet-rules.md`（列格式选择、列偏移检测、备注规则、质量检查）。

**默认输出列**：`导师`, `Location`, `学校名字`, `QS排名`, `美国USNEWS排名`, `Department`, `导师主页`, `导师联系方式`, `博士申请信息`, `其他导师信息`, `备注`

- `美国USNEWS排名`：仅当列表中有美国学校时包含此列
- `导师联系方式`：**国内学校必填**（导师 email），海外学校选填

---

## Vika 集成

**零依赖。** 所有操作使用 Python 3 标准库（`urllib` + `json`）——无需 `vika-cli`、npm 或第三方 SDK。仅需 API token 和任意 Python 3 安装。导入 Excel 时额外需要 `openpyxl`。

### 设置

1. 从 URL 解析：`datasheetId`（`dstXXX`）、`viewId`（`viwXXX`）
2. **Token 安全**：引导用户设置环境变量（不要在聊天中明文传递）：
   ```bash
   echo 'export VIKA_TOKEN=你的token' >> ~/.zshrc && source ~/.zshrc
   ```
3. Base URL: `https://api.vika.cn/fusion/v1`

### 操作前必做：过滤选导意向非空行

**每次执行增删改操作前，必须先获取所有记录并排除 `选导意向（点击选择）` 非空的行：**

```python
# 获取所有记录
result = vika("GET", "/records?maxRecords=200&fieldKey=name")
all_records = result["data"]["records"]

# 仅保留选导意向为空的行（这些是待编辑的行）
editable = [r for r in all_records if not r["fields"].get("选导意向（点击选择）")]

# 选导意向非空的行（这些是受保护的行，只能读取）
locked = [r for r in all_records if r["fields"].get("选导意向（点击选择）")]

# 所有增删改操作只针对 editable 列表中的 recordId
```

### 能力（记录级 CRUD）

| 操作 | 端点 | 备注 |
|------|------|------|
| 列出字段 | `GET /datasheets/{id}/fields` | 发现字段名和类型 |
| 列出记录 | `GET /datasheets/{id}/records?viewId=...&maxRecords=...` | 支持 filterByFormula、sort、分页 |
| 创建记录 | `POST /datasheets/{id}/records` | 每批最多 10 条，使用 `fieldKey: "name"` |
| 更新记录 | `PATCH /datasheets/{id}/records` | 发送 `{"records": [{"recordId":"xxx","fields":{...}}], "fieldKey": "name"}` |
| 删除记录 | `DELETE /datasheets/{id}/records?recordIds=recXXX` | recordIds 在 query parameter 中，非 request body |

### 关键限制

- **DELETE 请求格式**：recordIds 必须在 URL query parameter 中（`?recordIds=recXXX,recYYY`），不能在 request body 中。Helper 函数已自动处理此转换。
- MagicLookUp / OneWayLink 字段不可通过 API 写入。
- 每批最多 10 条记录，批次间加 0.3-0.5 秒延迟。

详细的 API 代码模板和操作指南见 `references/vika-guide.md`。

### 自然语言删除流程

当用户说"删除导师XXX"或"删除这条记录"时，按以下步骤操作：

1. **解析请求**：理解要删除什么（按导师名、按学校、按备注内容等）
2. **查找记录**：用 GET + filterByFormula 找到匹配的记录
   ```python
   # 按导师名查找
   filter_expr = urllib.parse.quote('{导师}="张三"')
   result = vika("GET", f"/records?filterByFormula={filter_expr}&maxRecords=200")
   records = result["data"]["records"]
   ```
3. **提取 recordIds**：从响应中提取 `recordId` 字段
4. **确认删除**：向用户展示找到的记录，确认是否删除
5. **执行删除**：使用正确格式调用 DELETE
   ```python
   # ✅ 正确格式
   vika("DELETE", "/records", {"records": [r["recordId"] for r in records]})
   ```
6. **验证结果**：删除后再次 GET 确认记录已移除

⚠️ **DELETE 格式陷阱**：必须发送 `{"records": ["recXXX"]}`，不能发送纯数组 `["recXXX"]`。

---

## 搜索工作流（Vika 和 Excel 通用）

1. 解析学生背景、研究方向、硬排除条件、目标地区/学校、排名限制
2. **检测表格格式**（见 `references/spreadsheet-rules.md`）：Vika 模式从 URL 解析 datasheetId；Excel 模式若用户提供模板则沿用其列结构
3. 优先搜索官方大学来源
4. **SPA/动态站点**：先查 `references/search-techniques.md` L0-SPA 策略（找替代来源而非死磕 SPA 壳）；若无替代来源再探测 JS bundle 中的 API 端点
5. 按内容验证每个导师主页
6. 使用 `references/selection-rules.md` 判断指导资格
   - **国内学校**：走「国内学校选导规则」子章节（招生目录优先 → A/B/C 分类 → 邮箱必填 → 主页要求）
7. 使用 `references/spreadsheet-rules.md` 填写表格
   - **国内学校**：额外遵守「国内学校表格规则」子章节（导师联系方式必填、主页来源要求、备注链接格式）
8. **写入后立即设置学校关联字段**（见「学校关联字段必填」规则）：确定学校→查找/创建主表记录→设置OneWayLink→验证Location/QS/国内学校层次已同步
9. 没找到合适导师的学校记录排除原因

---

## 搜索标准

- **搜索引擎为主要策略**查找个人导师资料。搜 `"[导师名] [大学] professor"` 获取真实 URL——不要猜测 URL
  - **国内学校**：使用百度搜索而非 Google。搜索 `"[导师名] [学校名] 教授"` 或 `"[导师名] [学校名] 博士导师"`
- 优先官方大学页面而非个人网站
- **禁止 URL 猜测**：不要通过命名规则构造 URL（失败率约 90%）
- **SPA/JS 渲染页面**：不接受 200 空壳。**优先尝试 L0-SPA 策略**（找替代来源：个人网站、ResearchGate、研究中心页面），不要死磕 SPA 壳。详见 `references/search-techniques.md` L0-SPA 节和 `references/school-strategies.md` 各学校策略
- **并行搜索**：使用 WorkBuddy Agent 工具对不同的学校/域名进行并行检查。给每个 Agent 清晰的独立任务

### 浏览器验证替代方案

WorkBuddy 不能打开真实浏览器，但可以通过以下方式验证：
1. **WebFetch** 工具获取页面实际内容验证
2. **WebSearch** 搜索 `"[Name] [University] professor"` 发现真实 URL
3. **DDG HTML 搜索**：`https://html.duckduckgo.com/html/?q=...` 返回纯 HTML

---

## 深度发现管线

1. **全面扫描**——通过 API 获取所有教师数据，按目标院系筛选
2. **过滤**——检查剩余教师的学位、专业、研究关键词
3. **优先级排序**——按与学生方向的匹配度排名
4. **验证**——通过 WebFetch 打开个人页面，确认研究内容
5. **扩展**——仅在用户明确要求时重新检查之前 404 的链接

---

## 备注风格

**默认使用中文填写备注**，即使 Excel 源数据的研究方向为英文也需翻译为中文关键词。格式：`职称；研究方向（中文关键词）；风险/注意事项。`

### 🎯 语气要求：像面对面和学生说话

备注是给申请学生看的，语气要自然、有"人味"，像坐在旁边帮 ta 梳理信息的口吻。

- **不要**写"符合XX方向""匹配XX方向""老师的XX和你的XX很相关"——太机械，像机器打分
- **不要**写任何评价性连接语（如"很契合""直接对话""很有交集"）——备注只陈述事实
- **不要**堆砌术语关键词——写完要读一遍，想象你是说给一个紧张的申请者听的
- **要**写短句，多用"，"和"；"，少用长定语句
- 信息量要保持，但语气要松弛——是"帮你梳理"不是"给你打分"

### 备注格式：三段式

每条备注严格按三段式书写，用分号分隔：

**导师职称 + 研究方向（可带导师主页中的原文出处）+ 匹配度标识**

| 段落 | 内容 | 说明 |
|------|------|------|
| 第一段 | 导师职称 | 教授/副教授/助理教授/讲师等 |
| 第二段 | 研究方向 | 从导师主页中找原文依据再写研究方向（没有依据不写）；可摘录主页原文作为出处（如 `"research focuses on consumer judgment and decision-making"`） |
| 第三段 | 匹配度标识 | 仅两极标注，见下表 |

**匹配度标识（只在拿不准的时候写）**：

**唯一允许的标识**：方向有一定交叉但非核心对应，或资历/职称偏早期，有点沾边但拿不准 → 写 `可以备选一下呢～` 或 `可以往xx方向靠～`。

**其他所有情况（包括方向直接相关、有合理发表记录、特别推荐的顶尖人选）——不写任何匹配度标识，不写任何评价性连接语（如"很契合""能搭上""直接对话"）。备注只保留职称 + 研究方向，句号结尾即可。**

**禁止事项**：
- 禁止在备注里使用任何 emoji 符号（🔥👍👀⭐✅❌ 等）
- 禁止使用 `建议多看看呢～` 标识（已废弃，所有正面推荐语都不写）
- 禁止使用主观评价用语如 `高度相关`、`比较相关`、`很匹配`、`完美匹配`、`强匹配`、`弱相关`
- 禁止使用 `比较匹配～` 标识（已废弃，中间档不写）
- 禁止写评价性连接语如 `老师的XX和你的XX很相关`、`很契合`、`直接对话`、`很有交集`、`能搭上`
- 描述事实，不做评判

**正确示例**：
- `副教授；做消费者判断与决策、跨期选择。`
- `教授；做文化遗产和建筑史；可以往遗产保护方向靠；可以备选一下呢～`
- `助理教授；消费者决策、道德决策、自我概念清晰度；MIT PhD。`

### 🗣️ 第二人称规则（强制执行）

**所有备注必须以"你"称呼学生，不用"学生"二字。**

- ✅ `研究方向涵盖青少年心理健康与学校支持体系`
- ✅ `涉及跨文化适应与学生发展领域`
- ✅ `可以往你感兴趣的学校心理支持方向靠`
- ❌ ~~`和学生的青少年心理健康方向很契合`~~
- ❌ ~~`和学生发展及跨文化适配方向很契合`~~

**批量替换规则**：
- 所有 `学生` → `你` 或 `你的`（按语境）
- 所有 `和学生` → `和你的`
- 所有 `为学生` → `为你`

### 📛 方向编号禁止规则

**禁止在备注里写"方向1""方向2""方向3"等编号。必须写出具体的学术研究方向名称。**

- ❌ ~~`方向1核心匹配`~~
- ❌ ~~`方向1+2精准匹配`~~
- ❌ ~~`方向2方向对口`~~
- ✅ `研究方向涵盖青少年心理健康与学校支持体系`
- ✅ `涉及跨文化适应与学生发展领域`
- ✅ `在青少年心理健康与学校支持体系的视角下研究家庭干预`

方向编号 → 方向名称映射（按各学生的实际研究方向填写）：
- 方向1 → 青少年心理健康与支持体系
- 方向2 → 跨文化适应与学生发展
- 方向3 → 社会工作与青少年支持

### 🧹 AI 生成内容清理（写备注后强制执行）

**每次写完或批量修改备注后，必须执行以下清理操作：**

1. **移除所有 ⚠️ 及之后的内容**：备注中任何 `⚠️` 字符及其在同一语义段落后的内容都要删除。`⚠️` 会让学生感到不安，且看起来"太像 AI 写的"。

2. **移除所有 Emoji**：备注中不得出现任何 emoji 字符。使用以下 Unicode 范围过滤：
   - 表情符号（U+1F300–U+1F9FF）
   - 杂项符号（U+2600–U+27BF）
   - 装饰符号（U+2702–U+27B0）
   - 补充符号（U+1F900–U+1F9FF）
   - Dingbats（U+2700–U+27BF）
   - 交通/地图符号（U+1F680–U+1F6FF）

3. **清理后检查**：备注应只保留纯文字 + 中文标点，干净、自然、像人说的话。

**处理方式：** 可以在 Python 中用 Unicode 码点范围过滤，或逐字符检查 `unicodedata.category()`。

### 当学校匹配导师极少时（1-2位）

**在备注末尾补充说明该系其他教师的主要方向**，让学生有"已全面搜索"的信心。使用客观描述，不要写"确实没有"等主观判断。

正确示例：
- `教授；决策神经科学、风险与社会决策、无创脑刺激；这个系老师主要做临床和辅导实践方向，这一位的决策方向是唯一交叉点。`
- `副教授；消费者判断与决策；这个系以社会认知和神经犯罪学为主，就这一位做消费者行为相关；可以备选一下呢～`

要点：
- 说明"这个系老师主要做XX方向"——客观陈述事实
- 说明"只有这一位和XX搭边"——给出数量，让学生有"全面看过了"的感觉
- 不要写"没有合适的"、"确实没有"等主观评价

**补充信息行写入规则**：当需要另起一行单独写入系/院的补充说明（如某系无匹配导师）时，只需填写 `导师` 和 `备注` 两个字段，无需填写导师主页、博士申请信息等其他字段。

---

## 操作权限（CRITICAL — 所有操作前首先检查）

### 🚫 表格隔离：只操作用户指定的那张表

用户每次提供 Vika 分享链接时，从 URL 解析 `datasheetId`（`dstXXX`）。**本轮所有 CRUD 操作仅限于用户本次传入的这张表**。

**绝对禁止：**
- 操作用户未明确指定的其他 datasheet（即使 API token 有权限）
- 操作关联的学校主表（OneWayLink/MagicLookUp 引用的表）
- 通过 API 修改关联表的任何字段
- 在用户未给链接时自行假设或复用之前的 datasheetId

### 🚫 选导意向保护：已填写=不可触碰

当 Vika 表中 `选导意向（点击选择）` 字段有值（非空/非 null）时：
- **禁止修改**该行的任何字段
- **禁止删除**该行
- **禁止覆盖**该行的备注、研究方向、导师主页等字段
- **仅可读取**，不可写入

这些是学生已经审阅并反馈过的记录，任何修改都会造成数据损失。

**操作前必须：**
1. 先 GET 所有记录
2. 过滤掉 `选导意向（点击选择）` 非空的行
3. 仅对 `选导意向` 为空的行执行增删改操作

### 📋 当前活跃表（由用户传入）

表的链接和 datasheetId 随每次任务由用户提供，不固定。首次操作时从链接解析并确认。

---

## 关键规则（CRITICAL）

### SPA 壳返回 200 ≠ 链接有效
200 状态码 + SPA 壳（响应相同内容，所有不同 URL 返回相同字节数）→ **不能确认导师存在**。立即切换到 **L0-SPA 策略**：通过 WebSearch 找替代来源（个人网站、ResearchGate、研究中心页面），不要在 SPA 壳上浪费尝试。详见 `references/search-techniques.md` L0-SPA 节。

**经典案例（2026-07-13）**：CUHK STA 的 `/people/faculty/{name}/` 所有 URL 返回完全相同的 114KB 首页 HTML（SPA 壳），浏览器 JS 加载后才显示个人内容。正确格式是 `/peoples/{slug}/`（静态 HTML，可直接验证研究方向）。详见 `references/school-strategies.md` CUHK 章节。

**经典案例（2026-07-23）**：HKUST(GZ) 的 `facultyprofiles.hkust-gz.edu.cn` 全站 SPA，所有页面返回完全相同的 1522 字节 HTML 壳。URL 格式从 `?name=XXX` 查询参数改为 `/faculty-personal-page/NAME/shortname` 路径格式，但两种格式的 SPA 行为一致。自动化工具无法区分旧 URL 失效和新 URL 有效——200 ≠ 内容正确，必须浏览器人工确认。DSA 学域的替代来源为 `dsa.hkust-gz.edu.cn/blog/YYYY/MM/DD/name-slug`（WordPress 格式，可直接抓取内容）。

### CUHK(SZ) 特殊规则：教师 URL ID 不稳定（2026-07-23 验证）

CUHK(SZ) 数据科学学院的 `sds.cuhk.edu.cn/teacher/XXX` 的 ID 不是稳定标识符。2026 年 7 月的大规模重组中，同一学院的教授被分散到多个子域名和路径格式：

| 变化类型 | 示例 | 旧 URL | 新 URL |
|----------|------|--------|--------|
| teacher ID 变更 | 李爽 | `/teacher/273` | `/teacher/472` |
| 路径格式切换 | 王子卓 | `/teacher/254` | `/node/59` |
| 跨子域名迁移 | 罗智泉 | `sds.cuhk.edu.cn/teacher/478` | `sse.cuhk.edu.cn/teacher/184` |
| 个人网站 | 丁宏强 | `sds.cuhk.edu.cn/teacher/378` | `myweb.cuhk.edu.cn/chrisding/Home/Index` |

**关键规则**：

1. **每次会话重新搜索**：不信任上次存储的 teacher ID，使用 WebSearch 搜索 `"[导师名] sds.cuhk.edu.cn teacher"` 找新 ID
2. **不能按旧 ID 推断新 ID**：ID 变更无明显规律（不是简单递增/递减）
3. **多种 URL 格式并存**：同一学院同时存在 `/teacher/XXX`、`/node/XXX`、`myweb.cuhk.edu.cn`、`gklbdc.cuhk.edu.cn`、`sse.cuhk.edu.cn`、`mscfe.cuhk.edu.cn` 等多种格式，每种都需独立搜索验证
4. **Python SSL 握手失败 ≠ 链接失效**：`sds.cuhk.edu.cn` 在 Python urllib 中报 `SSLV3_ALERT_HANDSHAKE_FAILURE`，但 WebFetch 和浏览器可以正常访问。先用 WebFetch 测，再判死

### 导师主页必须是个人 URL

禁止使用通用院系列表页。每位导师必须有自己唯一的个人主页 URL。

**找不到合格个人主页的导师不得写入 Vika 表**（强制规则）：

禁止用以下 URL 代替个人主页：
- 院系师资列表页（如 `psych.zju.edu.cn/27612/list.htm`）
- 实验室成员列表页（如 `www.nakakolab.iis.u-tokyo.ac.jp/member/nakano_e.html`）
- 第三方平台（如 x-mol.com、百度百科、ResearchGate、**ORCID.org**、Google Scholar、aminer.cn）
- 学校/学院首页
- 返回 200 但内容为空或显示"该页面暂未开放"的占位页（如浙大 person.zju.edu.cn 的 577 字节 SPA 壳）

**特别强调**：ORCID（orcid.org）是研究者 ID 注册平台，不是个人主页。即使 ORCID 页面包含研究者的论文列表和简介，也**绝对不能**作为导师主页 URL 使用。ORCID 是学术身份标识系统，不是机构官方个人页面。

**验证标准**：用 WebFetch 或 curl 打开 URL，必须能看到导师的姓名、职称、研究方向等个人信息。如果页面没有导师个人内容，视为无效。

**处理**：找不到合格个人主页 → 不写入 Vika。宁可缺，不可用错误链接凑数。

### 禁止在备注中使用 ✅
✅ 标记是临时验证备注，不是研究事实。⚠️ 仅在页面真的被封锁时用作最后手段。

### 禁止在备注中写入报名/联系方式
不要写 "预计2026年入学"、"需发简历至xxx" 等内容。

### 禁止在备注中写入方向偏向
不要写 "方向偏X"、"没有非常符合的老师" 等主观评价。

### 禁止猜测 URL

**永远不要通过命名规则构造 URL。** 大学网站经常重组，URL 路径会变。必须通过以下方式获取 URL：
1. WebSearch 搜索 `"[Name] [University] professor"` 找个人页面
2. WebSearch 搜索 `"[University] [Department] faculty staff listing"` 找系列表页
3. **WebFetch 逐条验证**——200 状态码不代表内容正确（可能是 SPA 壳或 404 软页面）

**「其他导师信息」字段的特殊规则**：这些是系级教职员列表页 URL。每次会话开始、或用户说"检查链接"时，必须对每个 Department 的列表页 URL 逐一 WebFetch 验证。不要依赖 `references/school-strategies.md` 中记录的 URL——它们可能在你上次访问后已经变了。

**教训（2026-07-07）**：12 个「其他导师信息」链接中 4 个已失效（NUS × 2, NTU × 1, CityU × 1），都是因为大学重组了网站结构。

#### 大规模 404 复盘（2026-07-14）

黄肇启选导任务中，60 条记录的 `博士申请信息` 和 `其他导师信息` 出现大面积 404。复盘发现三个根因：

**根因 1：URL 猜测（占 90%+）**

所有出错的 URL 都是在写表时按路径命名规则直接构造的，没有经过 WebSearch 搜索验证。典型的猜测模式 vs 真实路径：

| 字段 | 猜测的路径 | 真实路径 |
|------|-----------|---------|
| HKU SAAS 博士申请 | `/programmes/research-postgraduate` | `/programme/rpg/mphil-phd` |
| HKU Nursing 博士申请 | `/education/postgraduate/` | `/education/doctor-of-philosophy-programme` |
| CUHK SPHPC 博士申请 | `/programmes/` | `/mphil-phd-programme/` |
| HKUST ECE 博士申请 | `/pg/` | `/admissions/postgraduate` |
| HKUST LifeSci 博士申请 | `/programmes/` | 在 `prog-crs.hkust.edu.hk` 子域名 |
| HKU BS 博士申请 | `/programmes/research-postgraduate/` | `phd.hkubs.hku.hk/admissions/...` |
| PolyU Nursing 博士申请 | `/sn/study/` | `/study/pg/rpg/2026/sn` |
| HKU SAAS 院系列表 | `/staff` | `/staff_teaching.php` |
| HKUST CSE 院系列表 | `/people/faculty/` | `/admin/people/faculty/` |
| HKUST LifeSci 院系列表 | `/people/` | `/faculty-members/` |
| HKUST ECE 院系列表 | `/people/` | 无系级列表页，用 `facultyprofiles.hkust.edu.hk` |
| HKUST Math 院系列表 | `/people/faculty/` | 无系级列表页，用 `facultyprofiles.hkust.edu.hk` |

**核心教训**：博士申请信息页和院系列表页的 URL 结构在各校各系之间**没有统一规律**。`/programmes/`、`/people/`、`/staff`、`/pg/`、`/study/` 这些常见路径在大量真实站点上都是 404。**必须先 WebSearch 搜索到真实页面，再把 URL 写进表里，一步都不能省。**

**根因 2：大学网站持续重组**

CUHK SPHPC 的院系列表页从 `/people/` 迁移到了 `/academic-staff/`，原有 URL 直接 404。这类重组是常态，不是偶发事件。

**根因 3：CityU WAF 干扰诊断**

CityU 全站 Incapsula WAF 会使自动化工具看到的返回码异常（403/404/超时/空壳），容易和真正 404 混淆。必须区分"WAF 拦截"和"真 404"——前者是所有 CityU URL 的系统性问题，后者是 URL 本身失效。

**防范措施（写入前强制执行）**：

对于 `博士申请信息` 和 `其他导师信息` 两个字段，**写入每条记录前**必须：
1. 用 WebSearch 搜索 `"[学校] [院系] PhD research postgraduate admission"` 或 `"[学校] [院系] faculty academic staff"`
2. 打开搜索结果中的真实页面
3. 确认 URL 可访问（非 404、非 SPA 壳）
4. 把验证过的 URL 填入字段
5. **批量写入后，对所有非 CityU 的 URL 逐条 curl/WebFetch 验证**

### CityU（港城）特殊规则：WAF 防护 + 双域名策略

#### WAF 防护（2026-07-13 验证）

CityU 全站部署了 **Incapsula WAF**，会拦截所有来自自动化工具（curl、Python urllib、WebFetch）的请求。这不是链接失效，而是反爬机制。

| 域名 | 自动化访问现象 | 浏览器访问 |
|------|-------------|-----------|
| `cityu.edu.hk/stfprofile/` | 200 返回 Incapsula JS 挑战页（200-1000 字节） | ✅ 正常 |
| `scholars.cityu.edu.hk` | **403 Forbidden** | ✅ 正常 |

**关键认知**：
- 自动化审计脚本检测 CityU 链接时，**必然全部报死链**（或 SPA 壳），这是一个已知的系统性问题，不作为链接质量缺陷
- 链接有效性必须由人在浏览器中逐条确认
- 不要因为 WAF 拦截就删除或替换有效的 CityU 导师链接
- 自动化审计的链接阈值应考虑 CityU 的 WAF 影响（例如：从 55 条总量中扣除 15 条 CityU 后再计算通过率）

#### 链接格式策略

CityU 有两种导师主页 URL 格式：

1. **stfprofile 格式**（推荐优先使用）：`https://www.cityu.edu.hk/stfprofile/xxx.htm`
   - 部分旧版教授页面使用此格式
   - 自动化访问返回 Incapsula JS 挑战页（约 200-1000 字节）

2. **scholars 格式**：`https://scholars.cityu.edu.hk/en/persons/xxx/`
   - Pure Portal 全校统一入口，信息更完整
   - 自动化访问返回 403

**选择策略**：两种格式都可以用，stfprofile 优先级更高（因为至少不返回 403）。最终都需人在浏览器确认。

#### 写入后复验规则

由于 WAF 的存在，"写入后复验"规则做以下调整：

1. 写入 Vika 后，用 WebFetch 或 urllib 快速检查
2. **预期结果**：stfprofile 返回 200 + 约 200-1000 字节（JS 挑战）；scholars 返回 403
3. 以上两种结果都**不代表链接失效**——这是 WAF 的正常表现
4. **仅当以下情况才删除记录**：Google 搜索找不到该导师、系页面已删除、或浏览器确认 404
5. **交付前**：提示用户在浏览器中抽查 CityU 链接（特别是 scholars 格式的）

### 排除退休/名誉教授
不添加 Emeritus、退休、约 70 岁以上、或不在当前教职员目录中的导师。

### 禁止修改已填写选导意向的记录（见上方「操作权限」）

当 Vika 表中 `选导意向（点击选择）` 非空时，该行即被锁定——不得修改、删除或覆盖。详见「操作权限 → 选导意向保护」。

### 禁止向 MagicLookUp 或计算字段写入
QS排名、Location、美国USNEWS排名等字段通过 API 只读。

### 新增记录去重
写入新记录前，检查是否已经存在于已有选导意向的记录中。

### 🏫 学校关联字段必填（强制执行 — CRITICAL）

**每条新导师记录写入后，必须立即设置学校关联字段（OneWayLink），让 Location / QS排名 / 美国USNEWS排名 / 国内学校层次 通过 MagicLookUp 自动同步。**

这是最容易遗漏但影响最大的步骤。学校关联字段决定整行的 Location、QS、USNEWS、国内学校层次等信息，不填等于记录残缺。

#### 第一步：识别新学生 / 老学生（强制执行，不可跳过）

每张选导表的 OneWayLink 字段指向的学校主表不同。必须先从字段定义读取 `foreignDatasheetId` 来判断学生类型：

```python
r = vika('GET', f'/datasheets/{datasheet_id}/fields')

link_fields = []
for f in r['data']['fields']:
    if f['type'] == 'OneWayLink':
        link_fields.append({
            'field_id': f['id'],
            'field_name': f['name'],
            'main_table_id': f['property']['foreignDatasheetId'],
        })
```

**根据 `foreignDatasheetId` 的值自动识别**：

| foreignDatasheetId | 学生类型 | 主表模式 |
|---------------------|----------|----------|
| `dstMNzQU9Aa58DpgW3` | **新学生** | 统一主表（一张表管全球） |
| `dstNvlYbmD2BTMCB0r` | **老学生** | QS 主表（非美国学校） |
| `dstd7iuffLGUbSnavd` | **老学生** | US 主表（美国学校） |

新老学生的后续操作流程不同，下面分别说明。

---

#### 第二步-A：新学生模式（统一主表 `dstMNzQU9Aa58DpgW3`）

约 2200 条记录，统一管理全球所有学校。**只有一个 OneWayLink 字段**（如「学校名字」），链接到同一张主表。

字段结构：

| 字段 | 类型 | 说明 |
|------|------|------|
| `学校全称` | Text | 完整英文/中文校名 |
| `学校中文名` | Text | 中文名称 |
| `学校英文缩写` | Text | 如 SYSU、CUC |
| `2026年QS排名` | Number | 2026 QS 综合排名（无排名留空） |
| `2027年QS排名` | Number | 2027 QS 综合排名 |
| `2025年QS排名` | Number | 历史参考 |
| `2025年USNews排名` | Number | 美国学校专用 |
| `国家/地区` | SingleSelect | 含 China (Mainland)、United States 等 |
| `所在大洲` | SingleSelect | Asia、Europe、Americas 等 |
| `国内学校层次` | SingleSelect | 985 / 211 / 双非 / 双一流 / 专业院校 |
| `国内学校所在省份` | SingleSelect | 如广东省、北京市、江苏省 |
| `美国学校所在州` | SingleSelect | 美国学校专用（如 加利福尼亚州） |

**创建学校记录**（新学生）：

```python
# 国内学校示例
data = {'records': [{'fields': {
    '学校全称': '广州大学',
    '学校中文名': '广州大学',
    '学校英文缩写': 'GU',
    '国家/地区': 'China (Mainland)',
    '所在大洲': 'Asia',
    '国内学校层次': '双一流',
    '国内学校所在省份': '广东省',
}}], 'fieldKey': 'name'}
resp = vika('POST', f'/datasheets/{main_table_id}/records', data)
school_rid = resp['data']['records'][0]['recordId']

# 关联
data = {'records': [{
    'recordId': supervisor_rid,
    'fields': {link_field_name: [school_rid]}
}], 'fieldKey': 'name'}
vika('PATCH', f'/datasheets/{datasheet_id}/records', data)
```

SingleSelect 字段直接传字符串值即可，API 自动匹配选项。

---

#### 第二步-B：老学生模式（双主表）

两张独立的学校主表，选导表通常有**两个 OneWayLink 字段**：

| 关联字段 | 链接到 | 适用情况 |
|----------|--------|----------|
| `非美国地区学校` | QS 主表（`dstNvlYbmD2BTMCB0r`，~1651 条） | 非美国学校 |
| `美国地区学校` | US 主表（`dstd7iuffLGUbSnavd`，~439 条） | 美国学校 |

**QS 主表字段**（老学生）：

| 字段 | 类型 | 说明 |
|------|------|------|
| `学校` | Text | 英文校名 |
| `排名` | Number | QS 综合排名 |
| `Location` | SingleSelect | 国家/地区 |
| `地区` | SingleSelect | 大洲 |

**US 主表字段**（老学生）：

| 字段 | 类型 | 说明 |
|------|------|------|
| `学校` | Text | 英文校名 |
| `排名` | Number | US News 排名 |
| `所在州` | SingleSelect | 美国所在州 |

**⚠️ 老学生 US 主表无 Location 字段**：`dstd7iuffLGUbSnavd` 只有 `学校`/`排名`/`所在州`，没有 `Location` 字段。设置 `美国地区学校` 后 MagicLookUp 不会自动填充 Location。

**解决方案**：同时设置 `非美国地区学校` 指向一个已有 Location=United States 的学校（如 MIT、Stanford），用作 Location 来源。此时记录会同时有两个链接。

**创建学校记录**（老学生）：

```python
# QS 主表
data = {'records': [{'fields': {
    '学校': 'Ghent University',
    '排名': 159,
    'Location': 'Belgium',
    '地区': 'Europe',
}}], 'fieldKey': 'name'}
vika('POST', f'/datasheets/dstNvlYbmD2BTMCB0r/records', data)

# US 主表
data = {'records': [{'fields': {
    '学校': 'Harvard University',
    '排名': 3,
    '所在州': '马萨诸塞州',
}}], 'fieldKey': 'name'}
vika('POST', f'/datasheets/dstd7iuffLGUbSnavd/records', data)
```

#### 第三步：在主表中查找/创建学校（⚠️ 必须翻页到底，不可只查 200 条）

> **教训（2026-07-20）**：主表实际有 2200+ 条学校记录，`pageSize=200` 只返回第一页。
> 不翻页会误判学校不存在 → 创建重复 → 后续维护混乱。
> **必须用 `pageNum` 翻页直到 `len(records) >= total`，确认遍历全部记录后再决定是否新建。**

```python
def find_school_by_name(main_table_id, school_name, mode='new'):
    """翻页遍历全部主表记录，按名字查找学校。mode: 'new' 或 'old'"""
    name_field = '学校全称' if mode == 'new' else '学校'
    page = 1
    while True:
        r = vika('GET', f'/datasheets/{main_table_id}/records?pageSize=500&pageNum={page}&cellFormat=string')
        for rec in r['data']['records']:
            if rec['fields'].get(name_field) == school_name:
                return rec['recordId'], rec['fields']
            # 新学生表还支持中文名校准
            if mode == 'new' and rec['fields'].get('学校中文名') == school_name:
                return rec['recordId'], rec['fields']
        total = r['data']['total']
        if len(r['data']['records']) < 500 or page * 500 >= total:
            break
        page += 1
    return None, None
```

#### 第四步：创建学校记录（仅在确认不存在时）

第三步搜不到才创建。新老学生字段名不同，按模式选择：

```python
if mode == 'new':
    data = {'records': [{'fields': {
        '学校全称': '广州大学',
        '学校中文名': '广州大学',
        '学校英文缩写': 'GU',
        '国家/地区': 'China (Mainland)',
        '所在大洲': 'Asia',
        '国内学校层次': '双一流',
        '国内学校所在省份': '广东省',
    }}], 'fieldKey': 'name'}
elif mode == 'old':
    data = {'records': [{'fields': {
        '学校': 'Ghent University',
        '排名': 159,
        'Location': 'Belgium',
        '地区': 'Europe',
    }}], 'fieldKey': 'name'}
resp = vika('POST', f'/datasheets/{main_table_id}/records', data)
school_rid = resp['data']['records'][0]['recordId']
```

#### 第五步：设置导师记录的 OneWayLink

```python
data = {'records': [{
    'recordId': supervisor_rid,
    'fields': {link_field_name: [school_rid]}
}], 'fieldKey': 'name'}
vika('PATCH', f'/datasheets/{datasheet_id}/records', data)
```

#### 第六步：验证

Patch 后重新 GET 记录，确认 `Location`、`QS排名`、`国内学校层次` / `美国USNEWS排名` 等已通过 MagicLookUp 自动填充。

**禁止事项**：
- 禁止假设学生类型（必须从 `foreignDatasheetId` 动态识别新/老学生）
- 禁止新老学生混用字段名（新学生用 `学校全称`/`2026年QS排名`，老学生用 `学校`/`排名`）
- 禁止在不创建主表记录的情况下直接设置 OneWayLink（会导致幽灵链接）
- 禁止在未翻页遍历全部主表记录前就判断"学校不存在"并创建
- 禁止 OneWayLink 值写成字符串而非数组

**常见错误**：
- 不识别学生类型 → 用错字段名 → API 报错或写入无效字段
- OneWayLink 值写成字符串而非数组 → API 报错 → 链接未生效
- 翻页不全就判断学校不存在 → 创建重复记录 → 主表数据混乱

### 国内学校搜索（不使用 Google）
国内（中国大陆）学校的导师搜索**不使用 Google**，改用百度搜索：
- `[导师名] [学校名] 教授`
- `[导师名] [学校名] 博士导师`
- 辅助来源：百度学者、百度百科、知网、万方

所有国内学校导师必须遵守 `references/selection-rules.md` 中的「国内学校选导规则」。

---

## 飞行前检查清单（添加导师前强制执行）

```
[ ] 1. 搜索引擎搜索: "[Name] [University] professor"
[ ] 2. 打开个人页面: 确认姓名、职称、院系
[ ] 3. 检查研究: 阅读 Research Interests 部分
[ ] 4. 检查活跃状态: 非 Emeritus/Retired/Visiting
[ ] 5. 匹配方向: 研究关键词与学生方向对比
[ ] 6. 个人 URL: 验证是个人页面而非通用列表页
[ ] 7. 写入记录: 所有 6 步检查通过后再写入
[ ] 8. 获取学校主表 ID: GET /datasheets/{id}/fields → 找 OneWayLink 字段的 property.foreignDatasheetId
[ ] 9. 设置学校关联: 写入后立即设置 OneWayLink → 验证 Location/QS/国内学校层次 已同步
```

---

## 常见错误速查表

| 错误 | 示例 | 正确做法 |
|------|------|---------|
| 猜测 URL | 拼 `/programmes/research-postgraduate`、`/people/faculty/`、`/pg/` 等 | WebSearch 搜索真实页面 URL |
| 批量写入「博士申请信息」时未逐个验证 | 42 条新记录全空，填表时按命名规则拼写 | 每条 URL 写入前必须 WebSearch → 打开验证 → 再填入 |
| 依赖上次会话的院系列表 URL | CUHK SPHPC `/people/` 已变更为 `/academic-staff/` | 每次会话重新 WebFetch 验证 |
| SPA 壳当有效 | 200 响应但无内容（HKUST Math `~macyang` 86B） | 交叉验证姓名和内容，换替代 URL |
| CityU WAF 404 当真实 404 | CityU MKT/EF 系页面返回 404（WAF 拦截） | 浏览器人工确认，区分 WAF 和真 404 |
| 老学生主表有 100 条硬限制（旧表） | 旧 QS/US 主表写入 >100 条时静默丢弃 | 不重新验证 GET；主表只增不删 |
| CDU/CDU 等小型大学研究人员页 404 | RD3 确认有效后 RD4 再查变 404 | 交付前逐条复验所有链接，不信任上一轮验证结果 |
| 备注中 SPA 标注和匹配标识混在同一段 | `Macquarie为SPA系统需浏览器验证；可以备选一下呢～` | SPA 标注放在第二段末尾，第三段仅放匹配标识 |
| CUHK(SZ) teacher ID 跨会话失效 | 上次存的 `/teacher/254` 已变 `/node/59`、`/teacher/273`→`/teacher/472` | 每次会话重新 WebSearch 搜新 ID，不信任历史 URL |
| HKUST(GZ) SPA 全站 1522B 壳 | 新旧 URL 格式均返回相同字节数，自动化无法判定有效性 | 标记 SPA，提示浏览器确认；同步搜索 DSA 博客替代来源 |
| Python SSL 失败当 404 | `sds.cuhk.edu.cn` Python urllib 报 handshake failure | 换 WebFetch 验证，不直接判死 |
| Vika URL 字段 PATCH 静默失败 | `fieldKey="name"` 返回 200 但 URL 未更新 | 必须 `fieldKey="id"` + 字段 ID |
| PATCH 后回读键名错误 | GET 返回字段名键，用 field ID 读永远 None | PATCH 前后各拉字段列表，回读用字段名 |

---

## 链接修复工作流（用户说"链接打不开/404"时触发）

当用户反馈已有表格中某学校/学院的导师链接大面积失效时，按以下步骤系统修复：

### 步骤 1：诊断区分失效类型

用 curl/WebFetch 逐一检查每个链接的状态，区分三类情况：

| 类型 | 症状 | 说明 |
|------|------|------|
| 真 404 | HTTP 404 + 页面标题含"404" | URL 确实失效，需搜索新链接 |
| SPA 壳 | 200 + 所有 URL 返回相同字节数 + 无实际教授信息 | 站点是 JavaScript 渲染，自动化无法判断 |
| WAF/SSL 拦 | 403 / SSL 握手失败 / 超时 | 防护机制干扰，需换工具验证 |

### 步骤 2：按学校搜索新 URL

不要逐个导师搜，先搜学校的教师目录页找全员新链接格式，再针对性补充搜索：

```python
# 批量搜索语法示例
for name in broken_professors:
    WebSearch(f'"{name}" sds.cuhk.edu.cn teacher OR node')
```

**CUHK(SZ) 搜索技巧**：
- 搜 `"[导师名] sds.cuhk.edu.cn teacher"` 找标准 teacher 页
- 搜 `"[导师名] cuhk.edu.cn"` 发现跨子域名迁移（myweb/gklbdc/sse/mscfe）
- 搜 `"[导师名] sds.cuhk.edu.cn node"` 发现 node 格式页面

**HKUST(GZ) 搜索技巧**：
- 搜 `"dsa.hkust-gz.edu.cn [导师名] faculty blog"` 找 WordPress 格式替代页
- `facultyprofiles.hkust-gz.edu.cn` 是纯 SPA，自动化只能确认 200，无法确认内容
- 若 DSA 博客有个人页则优先使用（可自动化验证内容）

### 步骤 3：写入前逐条 WebFetch 验证

每个候选 URL 必须用 WebFetch 打开，验证能显示：
- ✅ 导师姓名、职称
- ✅ 研究方向
- ✅ 院系归属

出现以下任一情况即判定无效：
- ❌ 404 / 页面未找到
- ❌ 只有导航栏/空壳（SPA）
- ❌ 显示的是其他不相关人员

### 步骤 4：Vika PATCH 修复

```python
# URL 字段 PATCH 必须 fieldKey="id"
fields = vika("GET", "/fields")
homepage_fid = next(f["id"] for f in fields["data"]["fields"] if '导师主页' in f['name'])

updates = [{"recordId": rid, "fields": {homepage_fid: new_url}} for rid, new_url in fixes.items()]

for i in range(0, len(updates), 10):
    vika("PATCH", "/records", {"records": updates[i:i+10], "fieldKey": "id"})
```

### 步骤 5：PATCH 后回读验证

```python
# 回读时必须用字段名（不是 field ID）
verified = vika("GET", f"/records?recordIds={','.join(fixed_ids)}")
for r in verified['data']['records']:
    # ✅ 正确：用字段名读取
    url = r['fields'].get('导师主页', '')
    # ❌ 错误：用 field ID 读取 → 永远 None
    # url = r['fields'].get(homepage_fid, '')
```
| CUHK(SZ) teacher ID 跨会话失效 | 上次存的 `/teacher/254` 已变 `/node/59`、`/teacher/273` 变 `/teacher/472` | 每次会话重新 WebSearch 搜索新 ID，不信任旧 ID |
| HKUST(GZ) SPA 全站 1522B 壳 | 新旧 URL 格式均返回相同字节数，自动化无法判定有效性 | 标记为 SPA，提示用户浏览器确认；同步搜索 DSA 博客替代来源 |
| Python SSL 失败当链接失效 | `sds.cuhk.edu.cn` Python urllib 报 handshake failure | 换 WebFetch 验证，不直接判 404 |
| Vika URL 字段 PATCH 静默失败 | `fieldKey="name"` 写入 URL 字段返回 200 但未更新 | 必须用 `fieldKey="id"` + 字段 ID（非字段名） |
| PATCH 后回读用错键名 | GET 返回字段名键，但用 field ID 去读永远 None | PATCH 前后各拉一次字段列表，回读用当前字段名 |

---

## 写入后审计（强制执行）

每次批量写入 Vika 后，运行审计脚本：

```bash
python3 scripts/audit.py [DATASHEET_ID] [VIKA_TOKEN]
```

审计检查：
1. 交叉链接（URL 域名与 Department 不匹配）
2. 备注格式（缺少；分隔符、缺少句号）
3. 备注中是否有垃圾内容
4. 缺少必填字段（导师主页、博士申请信息、其他导师信息）
5. **国内学校**：检查 `导师联系方式` 是否为空，是否为有效 email 格式

---

## 协作流程（"开始学生" / "结束学生"）

当用户说**"开始学生"**（或"提交更新"、"同步并提交"等）时，自动执行 pull -> branch -> commit -> push -> PR 全流程。当用户说**"结束学生"**时，先提取本轮搜索经验更新策略库，再自动执行同步流程。

**完整执行步骤见 `references/collaboration-workflow.md`。** 触发后按该文档的 11 步流程执行，无需用户懂 git。

---

## 参考文献

- 决定候选人能否进入主表前，阅读 `references/selection-rules.md`
  - **国内学校**：重点阅读文档末尾的「国内学校选导规则」（招生目录优先、A/B/C 分类、邮箱必填）
- 创建/编辑/验证表格前，阅读 `references/spreadsheet-rules.md`
  - **国内学校**：重点阅读文档末尾的「国内学校表格规则」（导师联系方式列、主页来源、备注链接格式）
- 开始搜索前，阅读 `references/search-orchestrator.md`
- 访问学校前，检查 `references/school-strategies.md`
- SPA/API 发现技巧，见 `references/search-techniques.md`
- Vika 完整 CRUD 操作 + API 代码模板，见 `references/vika-guide.md`
- 协作流程（开始学生/结束学生），见 `references/collaboration-workflow.md`
