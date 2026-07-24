# Vika Guide — API 代码模板 + 完整操作指南

零依赖。所有操作使用 Python 3 标准库（`urllib` + `json`）——无需 `vika-cli`、npm 或第三方 SDK。仅需 API token 和任意 Python 3 安装。导入 Excel 时额外需要 `openpyxl`。

---

## 0. Setup

> **Token 安全**：不要把 API Token 贴到聊天框。在终端里跑：
> ```bash
> echo 'export VIKA_TOKEN=uskXXXXXX' >> ~/.zshrc && source ~/.zshrc
> ```
> Agent 从 `$VIKA_TOKEN` 环境变量读取，token 只存在你机器上。

### URL 解析

```
https://vika.cn/share/shrXXX/dstXXX/viwXXX
                              ^^^^^^  ^^^^^^
                           datasheetId  viewId
```

### vika() 函数（所有操作的基础）

```python
import os, json, time
from urllib.request import Request, urlopen

TOKEN = os.environ.get("VIKA_TOKEN", "")
DATASHEET = "dstXXX"  # 从 URL 解析
BASE = "https://api.vika.cn/fusion/v1"

def vika(method, path, body=None):
    url = f"{BASE}/datasheets/{DATASHEET}{path}"
    headers = {"Authorization": f"Bearer {TOKEN}", "Content-Type": "application/json"}

    # DELETE 用 query parameter ?recordIds=recXXX,recYYY（不是 request body）
    if method == "DELETE" and body:
        if isinstance(body, list):
            url += f"?recordIds={','.join(body)}"
            data = None
        elif isinstance(body, dict) and "records" in body:
            url += f"?recordIds={','.join(body['records'])}"
            data = None
        else:
            data = json.dumps(body).encode() if body else None
    else:
        data = json.dumps(body).encode() if body else None

    req = Request(url, data=data, headers=headers, method=method)
    try:
        resp = urlopen(req)
        raw = resp.read()
        return json.loads(raw) if raw else {"code": 0, "message": "OK", "data": {}}
    except Exception as e:
        try:
            if hasattr(e, 'read'):
                raw = e.read()
                body = json.loads(raw) if raw else {}
                raise Exception(f"API {getattr(e,'code','?')}: {body.get('message',str(e))}")
        except Exception:
            pass
        raise
```

---

## 1. View Table Schema

```python
result = vika("GET", "/fields")
for f in result["data"]["fields"]:
    print(f"{f['name']:20s} [{f['type']}]")
    if f['type'] == 'SingleSelect':
        opts = [o['name'] for o in f.get('property',{}).get('options',[])]
        print(f"  Options: {opts}")
```

---

## 2. Read Records

### 2.1 All Records

```python
result = vika("GET", "/records?maxRecords=200&pageSize=200")
records = result["data"]["records"]
total = result["data"]["total"]
```

Page size max 200. For >200 records, paginate with `pageNum` or `pageToken` parameter.

### 2.2 Filtered by Formula

```python
from urllib.parse import quote
filter_expr = quote('{状态}="待发邮件"')
result = vika("GET", f"/records?filterByFormula={filter_expr}&maxRecords=200")
```

### 2.3 By Specific View

```python
result = vika("GET", "/records?viewId=viwXXX&maxRecords=200")
```

### 2.4 Only Specific Fields

```python
from urllib.parse import quote
path = f"/records?fields={quote('导师,Department,备注')}&maxRecords=200"
result = vika("GET", path)
```

### 2.5 String Format (避免嵌套对象)

```python
result = vika("GET", "/records?maxRecords=200&pageSize=200&cellFormat=string")
```

使用 `cellFormat=string` 获取纯文本值，避免 URL 字段返回 `{"text":"...","link":"..."}` 嵌套对象。

---

## 3. Create Records (Batch up to 10)

```python
new_records = [
    {"fields": {"导师": "张三", "Department": "心理学院(XX大学)", "备注": "教授；决策研究；比较匹配～", "待确认导师": "新加待check"}},
    {"fields": {"导师": "李四", "Department": "商学院(YY大学)", "备注": "副教授；消费者行为；比较匹配～", "待确认导师": "新加待check"}},
]
result = vika("POST", "/records", {"records": new_records, "fieldKey": "name"})
print(f"Created {len(result['data']['records'])} records")
```

For batch > 10, loop with delay:
```python
for i in range(0, len(all_records), 10):
    batch = all_records[i:i+10]
    vika("POST", "/records", {"records": batch, "fieldKey": "name"})
    time.sleep(0.3)
```

---

## 4. Update Records

### 4.1 Text / SingleSelect / Checkbox Fields

```python
updates = [
    {"recordId": "recXXX", "fields": {"Department": "心理学院(香港大学)", "状态": "待发邮件"}},
    {"recordId": "recYYY", "fields": {"备注": "教授；认知神经科学；在招博士"}},
]
vika("PATCH", "/records", {"records": updates, "fieldKey": "name"})
```

### 4.2 URL Fields (CRITICAL — fieldKey="name" 静默失败)

URL-type fields (导师主页, 博士申请信息, 其他导师信息) do NOT work with `fieldKey="name"`. API returns 200 but silently fails. **Must use field IDs with `fieldKey="id"`.**

```python
# 先获取 field IDs
fields = vika("GET", "/fields")
field_map = {f["name"]: f["id"] for f in fields["data"]["fields"]}

# 用 field ID 做 key，不带 fieldKey
updates = [
    {"recordId": "recXXX", "fields": {
        field_map["导师主页"]: "https://www.university.edu/faculty/prof-name",
        field_map["博士申请信息"]: "https://gradschool.university.edu/phd"
    }}
]
# PATCH with fieldKey="id" (CRITICAL: must include fieldKey="id")
req = Request(
    f"{BASE}/datasheets/{DATASHEET}/records",
    data=json.dumps({"records": updates, "fieldKey": "id"}).encode(),
    headers={"Authorization": f"Bearer {TOKEN}", "Content-Type": "application/json"},
    method='PATCH'
)
urlopen(req)
```

**Note**: This ONLY affects PATCH on URL-type fields. Text/SingleSelect/Checkbox fields work fine with `fieldKey="name"`. Using field IDs without `fieldKey="id"` returns 400 error.

### 4.3 Bulk Department Translation

```python
records = vika("GET", "/records?maxRecords=200&pageSize=200")["data"]["records"]

dept_map = {
    "Department of Psychology": "心理学院",
    "CUHK Business School": "香港中文大学商学院",
}

updates = []
for r in records:
    dept = r["fields"].get("Department", "")
    if dept in dept_map:
        updates.append({"recordId": r["recordId"], "fields": {"Department": dept_map[dept]}})

for i in range(0, len(updates), 10):
    vika("PATCH", "/records", {"records": updates[i:i+10], "fieldKey": "name"})
    time.sleep(0.3)
```

---

## 5. Delete Records

### 5.1 Delete by ID

```python
# vika() 函数自动把 list 转成 ?recordIds=recXXX,recYYY query parameter
vika("DELETE", "/records", ["recXXX", "recYYY"])
# or
vika("DELETE", "/records", {"records": ["recXXX", "recYYY"]})
```

### 5.2 Deduplicate (Keep Most Complete)

```python
from collections import defaultdict

records = vika("GET", "/records?maxRecords=200&pageSize=200")["data"]["records"]

name_map = defaultdict(list)
for r in records:
    name = r["fields"].get("导师", "").strip()
    if name:
        name_map[name].append(r)

to_delete = []
for name, recs in name_map.items():
    if len(recs) > 1:
        recs.sort(key=lambda r: sum(1 for v in r["fields"].values() if v), reverse=True)
        to_delete.extend(r["recordId"] for r in recs[1:])

if to_delete:
    vika("DELETE", "/records", to_delete)
    print(f"Deleted {len(to_delete)} duplicate records")
```

---

## 6. Import from Excel to Vika

```python
import openpyxl

wb = openpyxl.load_workbook("select.xlsx", data_only=True)
ws = wb["Sheet1"]
headers = [ws.cell(1, c).value for c in range(1, ws.max_column+1)]

existing = vika("GET", "/records?maxRecords=200&pageSize=200")["data"]["records"]
existing_names = {r["fields"].get("导师","").strip() for r in existing}

FIELD_MAP = {
    "导师": "导师", "Department": "Department", "导师主页": "导师主页",
    "博士申请信息": "博士申请信息", "其他导师信息": "其他导师信息", "备注": "备注",
}

new_records = []
seen = set()
for row in range(2, ws.max_row+1):
    name = ws.cell(row, headers.index("导师")+1).value
    if not name: continue
    name = name.strip()
    if name in existing_names or name in seen: continue
    seen.add(name)

    fields = {}
    for excel_col, vika_col in FIELD_MAP.items():
        if excel_col in headers:
            val = ws.cell(row, headers.index(excel_col)+1).value
            if val and str(val).strip() not in ("", "None"):
                fields[vika_col] = str(val).strip()
    if fields.get("导师"):
        new_records.append({"fields": fields})

for i in range(0, len(new_records), 10):
    vika("POST", "/records", {"records": new_records[i:i+10], "fieldKey": "name"})
    time.sleep(0.3)

print(f"Imported {len(new_records)} new records")
```

---

## 7. Fill Missing Links

```python
records = vika("GET", "/records?maxRecords=200&pageSize=200")["data"]["records"]

LINK_MAP = {
    "导师名": ("phd-program-url", "staff-list-url"),
}

updates = []
for r in records:
    name = r["fields"].get("导师", "").strip()
    phd = r["fields"].get("博士申请信息", "")
    other = r["fields"].get("其他导师信息", "")

    phd_ok = isinstance(phd, dict) and phd.get("text", "")
    other_ok = isinstance(other, dict) and other.get("text", "")

    if name in LINK_MAP and (not phd_ok or not other_ok):
        phd_url, other_url = LINK_MAP[name]
        fields = {}
        if not phd_ok: fields["博士申请信息"] = phd_url
        if not other_ok: fields["其他导师信息"] = other_url
        if fields:
            updates.append({"recordId": r["recordId"], "fields": fields})

for i in range(0, len(updates), 10):
    vika("PATCH", "/records", {"records": updates[i:i+10], "fieldKey": "name"})
    time.sleep(0.3)
```

---

## 8. Check for Blank Records

```python
records = vika("GET", "/records?maxRecords=200&pageSize=200")["data"]["records"]

blanks = [r["recordId"] for r in records if not r["fields"].get("导师", "").strip()]

if blanks:
    print(f"Found {len(blanks)} blank records: {blanks}")
    # vika("DELETE", "/records", blanks)  # Uncomment to delete
```

---

## 9. Cross-School Link Audit

审查已有表格时，必须检查「博士申请信息」和「其他导师信息」的 URL 域名是否与导师实际所在学校一致。

```python
import re

records = vika("GET", "/records?maxRecords=200&pageSize=200")["data"]["records"]

SCHOOL_DOMAINS = {
    "Nanyang Technological University": ["ntu.edu.sg"],
    "National University of Singapore": ["nus.edu.sg"],
    "Hong Kong Polytechnic University": ["polyu.edu.hk"],
    "Chinese University of Hong Kong": ["cuhk.edu.hk"],
    "University of Hong Kong": ["hku.hk"],
    "University of Sydney": ["sydney.edu.au"],
}

issues = []
for r in records:
    f = r["fields"]
    name = f.get("导师", "")
    dept = f.get("Department", "")

    expected_domains = []
    for school_key, domains in SCHOOL_DOMAINS.items():
        if school_key.lower() in dept.lower():
            expected_domains = domains
            break
    if not expected_domains:
        continue

    for field_name in ["博士申请信息", "其他导师信息"]:
        val = f.get(field_name, "")
        if isinstance(val, dict):
            val = val.get("text", "")
        if val:
            domain = re.search(r'https?://([^/]+)', val)
            if domain:
                domain = domain.group(1)
                if not any(d in domain for d in expected_domains):
                    issues.append(f"  {name}: {field_name} domain ({domain}) != expected ({expected_domains})")

if issues:
    print(f"Found {len(issues)} cross-school link issues:")
    for issue in issues:
        print(issue)
else:
    print("No cross-school link issues found.")
```

---

## 10. Batch Research Area Completion

当表格中多条记录的「导师研究领域」为空时，通过 WebFetch 逐个抓取导师主页，提取 research interests 并批量写入。

```python
records = vika("GET", "/records?maxRecords=200&pageSize=200&cellFormat=string")["data"]["records"]

needs_research = []
for r in records:
    f = r["fields"]
    if not f.get("导师研究领域", ""):
        homepage = f.get("导师主页", "")
        if isinstance(homepage, dict):
            homepage = homepage.get("text", "")
        if homepage:
            needs_research.append({
                "recordId": r["recordId"],
                "导师": f.get("导师", ""),
                "导师主页": homepage
            })

print(f"Found {len(needs_research)} records needing research area")
# Step 2: WebFetch each homepage (via WebFetch tool)
# Step 3: Batch PATCH with extracted research areas (text fields work with fieldKey="name")
```

---

## 11. Non-Writable Fields

These fields **cannot** be written via Fusion API — they are computed or linked:

| Field Type | Examples | Why |
|-----------|----------|-----|
| MagicLookUp | Location, QS排名, USNEWS排名 | Computed from linked records |
| OneWayLink | 学校名字, Location | Link to another datasheet |
| Formula | 等邮件几天 | Auto-calculated |
| LastModifiedTime | 状态变更日期 | Auto-generated |
| CreatedBy / CreatedTime | From, 选导时间 | Auto-generated |

### OneWayLink / MagicLookUp Write Workaround (verified 2026-07-03)

**Problem**: Vika API rejects OneWayLink/MagicLookUp fields with "Lookup field can't be edited" when using `fieldKey=name`.

**Solution**: Use **field NAMES** as keys, with a **list of record ID strings** as the value. Works with or without `fieldKey` in body.

```python
# Works — field name key + list of record ID strings
body = {
    "records": [
        {"recordId": "recXXX", "fields": {"学校名字": ["recSchoolId123"]}}
    ]
    # fieldKey can be "name" or omitted (defaults to name mode)
}

# Fails (400) — field ID key without fieldKey="id"
body = {
    "records": [
        {"recordId": "recXXX", "fields": {"fldFfXtdDSST1": ["recSchoolId123"]}}
    ]
}
```

**Workaround for creating records with linked fields**:
1. POST record WITHOUT `学校名字` and `Location` fields (use `fieldKey=name` for regular fields)
2. PATCH the record WITHOUT `fieldKey` to set `学校名字: [school_record_id]`
3. `Location` and `QS排名` will auto-fill after school link is set

**Note**: This only works for PATCH, NOT for POST.

---

## 12. Known API Issues

### DELETE Format (Critical)

Vika DELETE API uses query parameter, NOT request body. The `vika()` function handles this automatically.

### GET Cache Staleness

After DELETE or PATCH, subsequent GET requests may return stale data from an API cache. The Vika UI typically reflects changes faster than the API. Always verify in the UI after making changes.

### URL Field PATCH with fieldKey="name" Returns 200 but Silently Fails

See section 4.2 above. URL-type fields require field IDs without `fieldKey` parameter.

### 回读验证的两个陷阱（2026-07-17 实战教训）

1. **GET 回读必须用字段名，不能用 field ID**。PATCH 用 `fieldKey="id"` 写入后，GET（不带 fieldKey 参数）返回的 fields 字典是**字段名键**。用 `fields.get("fldXXX")` 读取永远是 None，会误判写入失败。
2. **用户可能在两次会话之间改了字段名**（例如 `导师主页` → `导师主页（⭐️看这里）`）。字段 ID 不变，写入照常成功，但按旧字段名回读会得到 None，造成"写入没生效"的假象。**批量写入前后都应重新拉一次 `GET /fields` 确认当前字段名**，回读以字段名为准。判读写入是否成功的可靠依据：PATCH 响应体回显的字段值 + 按当前字段名 GET 的值一致。

---

## 13. Rate Limits & Best Practices

- **10 records max** per POST/PATCH/DELETE request
- **0.3-0.5s delay** between batches
- 175 records = 15 batches = 15-20 seconds total
- Use `fieldKey: "name"` for Chinese field names (except for URL field PATCH)
- Always `strip()` names before comparison
- URL fields accept plain strings (API wraps them)

---

## 14. 待确认导师自查流程

当用户说"检查新加导师"时，获取所有 `待确认导师` = `新加待check` 的记录进行自查：

```python
from urllib.parse import quote

# 获取所有待确认导师=新加待check 的记录
filter_expr = quote('{待确认导师}="新加待check"')
result = vika("GET", f"/records?filterByFormula={filter_expr}&maxRecords=200&cellFormat=string")
pending = result["data"]["records"]

print(f"待检查记录: {len(pending)} 条")
for r in pending:
    f = r["fields"]
    print(f"  [{r['recordId']}] {f.get('导师','?')} | {f.get('备注','?')[:60]}")
```

自查项目（逐条执行，不自动放行）：
1. 导师主页 URL 可访问且为个人页面
2. 姓名/职称/院系与主页一致
3. 研究方向匹配度合理
4. 备注格式三段式
5. 备注无禁止内容
6. 博士申请信息 & 其他导师信息 URL 有效
7. 无遗漏风险标注
8. 非 Emeritus/退休/Visiting
9. 无重复（与选导意向非空记录比对）

检查完毕输出通过/修正/建议删除三类清单。**不自动清空 `待确认导师`**，提醒用户人工抽检后手动清空。
