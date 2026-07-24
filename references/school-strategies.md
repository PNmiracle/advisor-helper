# School Strategy Knowledge Base

Living registry of which access layer to use for each university. Discipline-agnostic — works for fashion, engineering, psychology, anything.

---

## Universal Access Framework (L0-SPA → L1 → L2 → L3 → L4)

Not every school needs a unique trick. Most fall into one of five layers:

| Layer | Name | When to use | Method | Requires |
|:---:|------|-------------|--------|----------|
| **L0-SPA** | Alternative Source Workaround | SPA returns empty shell, no accessible API, but scholars have personal sites/ResearchGate/etc. | WebSearch → find alternative domains → WebFetch personal sites/ResearchGate/research center pages | Nothing |
| **L1** | Direct Access | curl returns full HTML content | `curl` → parse staff list → extract names/links | Nothing |
| **L2** | API Mining | curl returns empty shell (SPA) but API exists | Download JS → grep `api\|baseUrl` → call API with `?size=1000` | JS bundle access |
| **L3** | Search Engine Fallback | curl blocked (403/WAF) OR JS renders but no API | Google `site:{uni} "{name}" professor` → click result in browser | In-app browser |
| **L4** | Tavily Extract/Crawl | L3 fails (WAF blocks even search, encrypted URLs, DDG empty) | `tavily-search-pro extract/crawl/search/research` → bypass client-side WAF via server-side fetch | `TAVILY_API_KEY` env var |

### Decision Tree

```
For each professor, identify the school →
Check school-strategies.md for that school's known layer

├── L0-SPA: Skip staff portal → WebSearch for alternative sources
│   ├── Found personal website → WebFetch (rich HTML, full content)
│   ├── Found ResearchGate → WebFetch (publications + research areas)
│   ├── Found research center page → WebFetch (role + projects)
│   └── Combine all sources → verified research profile
│
├── L1: curl → 200 + HTML table/links → parse directly
│
├── L2: curl → 200 + empty shell (<5KB) → hunt API in JS bundle
│   ├── API found → call API, filter results
│   └── No API → escalate to L3
│
├── L3: curl blocked (403/WAF) → search engine per name
│   ├── Search results found → verify in browser
│   └── Search blocked/encrypted → L4
│
└── L4: Tavily fallback for WAF/blocked pages
```

### L0-SPA — Alternative Source Strategy (NEW)

When a university's staff profile portal is pure SPA (returns <5KB empty HTML shell, status 200 but no content):

**Why SPA portals fail**: React/Vue/Angular render content client-side. WebFetch, curl, and even Tavily extract only receive the JS shell. The actual professor data never reaches the server response.

**Key insight**: Scholars maintain academic identities on MULTIPLE platforms — most are server-rendered and accessible:

| Source | Reliability | Access | Typical Content |
|--------|-------------|--------|-----------------|
| Personal academic website | **High** | WebFetch | Research areas, projects, CV, PhD students, teaching |
| Research center/lab page | **High** | WebFetch | Role description, funded projects, collaborators |
| ResearchGate | **Med-High** | WebFetch | All publications, research areas/tags, citation stats, co-authors |
| ORCID | **Medium** | WebFetch | Structured profile, affiliations, limited publication list |
| Google Scholar (snippet) | **Medium** | WebSearch | Citation count, research field tags (from search result snippet) |

**Discovery workflow**:
1. WebSearch `"[Name] [University] professor research"`
2. Scan results for non-staffportal domains (wordpress.com, .net, researchgate.net, .org.au, ORCID)
3. WebFetch 2-3 alternative sources for each professor
4. Cross-reference: research areas from personal site + publications from ResearchGate + citations from Scholar

**Reliability rule**: A research direction confirmed by 2+ independent alternative sources = verified. Single-source data = noted with source.

### L4 — Tavily Search-Pro Integration

When all L1-L3 methods fail (WAF blocks curl+browser+search, DDG returns empty, Google encrypts URLs), use tavily-search-pro as a server-side proxy that bypasses client-side anti-crawl protections.

**Command location**: `$HOME/.workbuddy/skills/tavily-search-pro/lib/tavily_search.py`
**Python runtime**: `$HOME/.workbuddy/binaries/python/envs/default/bin/python3`
**Required**: `TAVILY_API_KEY` environment variable set
**Note**: 如果 `tavily-search-pro` skill 未安装，L4 层不可用。通过 WorkBuddy 推荐市场安装，或跳过 L4 直接用 L3 搜索引擎兜底。

| Tavily Mode | When to Use | Example |
|-------------|-------------|---------|
| **extract** | WebFetch/curl blocked by WAF (403, Cloudflare, Incapsula) | Extract a professor's profile page content when direct access is blocked |
| **search** | Search engines return encrypted URLs or no results; DDG HTML empty | Search `"John Smith NUS professor"` when Google/DDG fail |
| **crawl** | Need to discover all faculty pages on a university site | Crawl a department's website to find all professor profile URLs |
| **map** | Need a sitemap of faculty-related URLs | Map `nus.edu.sg` to discover all staff directory paths |
| **research** | Deep analysis of a department's faculty composition | Research `"NUS Psychology department PhD supervisors"` for comprehensive results |

**L4 Command Syntax**:
```bash
TAVILY_API_KEY="$TAVILY_API_KEY" "$HOME/.workbuddy/binaries/python/envs/default/bin/python3" "$HOME/.workbuddy/skills/tavily-search-pro/lib/tavily_search.py" <mode> "query" [options]
```

**L4 Escalation Rule**: Only escalate to L4 after L3 has been attempted and confirmed to fail. L4 consumes credits (1-2 per call), so reserve it for genuine blockers.

### Why this works for any discipline

The layer is about **how the university website is built**, not what subject you're searching. A Vue SPA at an engineering school needs the same L2 API-hunting as a Vue SPA at an art school. The keywords change; the access method doesn't.

---

## Architecture → Layer Mapping

| Architecture | Default Layer | Notes |
|-------------|:---:|-------|
| Static HTML | **L1** | Direct curl + parse |
| Worktribe | **L1** | `/staff/profiles/` pattern |
| Pure Portal (with API) | **L2** | `?format=json` or `ws/api/persons/search` |
| Pure Portal (no API) | **L3** | JS-rendered, API blocked → search engine |
| Vue SPA | **L2** | JS bundle → API endpoint |
| React SPA | **L2** | JS bundle → API endpoint |
| Cloudflare WAF | **L3→L4** | curl blocked → browser + search engine → Tavily extract/crawl |
| Imperva/Incapsula | **L3→L4** | Browser may also be blocked → Tavily extract fallback |
| Custom CMS | **L1→L3** | Try L1 first, escalate if blocked |

---

## School Registry

Format: Architecture + Layer + Access method + Endpoints + Failures

### Per-School Template

```markdown
#### School Name
- **Layer**: L1 / L2 / L3
- **Architecture**: Static HTML / Pure Portal / Vue SPA / etc.
- **Best method**: [one sentence]
- **Key URL / endpoint**: [URL]
- **Failed**: [what didn't work]
- **Last verified**: YYYY-MM-DD
```

---

## School Entries

### 🇬🇧 United Kingdom

#### University of the Arts London (UAL)
- **Architecture**: Pure Portal (JS-rendered)
- **Layer**: L2
- **Best method**: L3 — Google `site:researchers.arts.ac.uk "{name}"` → click first result in browser. Profile URL format: `researchers.arts.ac.uk/{id}-{slug}`.
- **Key endpoint**: Profile URLs use format `https://researchers.arts.ac.uk/{numeric-id}-{slug}` (NOT `/en/persons/{slug}`)
- **Failed**: `?format=json` does NOT work on UAL Pure; `/en/persons/{slug}` returns 404; `curl` alone cannot extract data
- **Last verified**: 2026-06-26

#### Royal College of Art (RCA)
- **Architecture**: —
- **Layer**: —
- **Access**: —
- **Key endpoint**: —
- **Failed**: —
- **Last verified**: —

#### University of Leeds
- **Architecture**: Static HTML (staff profile pages)
- **Layer**: L1
- **Access**: Direct URL access to staff profiles at ahc.leeds.ac.uk/design/staff/
- **Key endpoint**: https://ahc.leeds.ac.uk/design/staff/
- **Failed**: —
- **Last verified**: 2026-06-26

#### University of Southampton
- **Architecture**: —
- **Layer**: —
- **Access**: —
- **Key endpoint**: —
- **Failed**: —
- **Last verified**: —

#### Kingston University
- **Architecture**: —
- **Layer**: —
- **Access**: —
- **Key endpoint**: —
- **Failed**: —
- **Last verified**: —

#### De Montfort University
- **Architecture**: Static HTML (staff profile pages)
- **Layer**: L1
- **Access**: Direct URL access to academic staff pages at dmu.ac.uk/academic-staff/
- **Key endpoint**: https://www.dmu.ac.uk/academic-staff/
- **Failed**: —
- **Last verified**: 2026-06-26

#### University of Edinburgh
- **Architecture**: JS-rendered profile directory (edwebprofiles.ed.ac.uk)
- **Layer**: L3
- **Access**: https://www.ed.ac.uk/profile/{name} → redirects to JS profile
- **Key endpoint**: https://www.ed.ac.uk/profile/alison-lenton
- **Failed**: —
- **Last verified**: 2026-06-26

#### University of Glasgow
- **Architecture**: —
- **Layer**: —
- **Access**: —
- **Key endpoint**: —
- **Failed**: —
- **Last verified**: —

#### University of Brighton
- **Architecture**: Pure Portal (research.brighton.ac.uk)
- **Layer**: L2
- **Access**: Pure Portal JSON API → parse researcher profiles
- **Key endpoint**: https://research.brighton.ac.uk/persons/
- **Failed**: —
- **Last verified**: 2026-06-26

#### Nottingham Trent University (NTU)
- **Architecture**: —
- **Layer**: —
- **Access**: —
- **Key endpoint**: —
- **Failed**: —
- **Last verified**: —

#### Heriot-Watt University
- **Layer**: L2
- **Architecture**: Pure Portal (researchportal) + Static HTML (hw.ac.uk)
- **Best method**: Use hw.ac.uk profiles (not researchportal UUIDs). Format: `hw.ac.uk/profiles/uk/school/tex/faculty/{slug}`
- **Key URL / endpoint**: `https://www.hw.ac.uk/profiles/uk/school/tex/faculty/{slug}`
- **Failed**: researchportal.hw.ac.uk UUID-based URLs → 404
- **Last verified**: 2026-06-26

#### University College London (UCL) — Psychology
- **Architecture**: profiles.ucl.ac.uk numeric ID system (RESTRUCTURED 2025-2026)
- **Access**: Old PALS URLs 404 → new system https://profiles.ucl.ac.uk/{numeric-id}-{name}
- **Key endpoint**: https://profiles.ucl.ac.uk/2667-david-shanks
- **Failed**: All old pals.ucl.ac.uk URLs broken
- **Last verified**: 2026-06-26

#### Birkbeck, University of London
- **Architecture**: Static HTML + numeric profile IDs
- **Access**: https://www.bbk.ac.uk/our-staff/profile/{id}/{name}
- **Key endpoint**: https://www.bbk.ac.uk/our-staff/profile/8009667/ulrike-hahn
- **Failed**: Profile IDs can redirect to wrong person; verify the ID
- **Last verified**: 2026-06-26

#### University of Bristol
- **Architecture**: Main dept pages redirect to generic homepage; use RIS portal
- **Access**: https://research-information.bris.ac.uk/en/persons/{name}
- **Key endpoint**: https://research-information.bris.ac.uk/en/persons/stephan-lewandowsky
- **Failed**: All bristol.ac.uk/psychology people pages redirect to dept homepage
- **Last verified**: 2026-06-26

#### University of Warwick — Business School
- **Architecture**: Static HTML (WBS separate from main Warwick)
- **Access**: https://www.wbs.ac.uk/about/person/{name}/
- **Key endpoint**: https://www.wbs.ac.uk/about/person/neil-stewart/
- **Failed**: Old warwick.ac.uk/fac/sci/psych/people URLs 404 for Stewart (moved to WBS)
- **Last verified**: 2026-06-26

### 🇦🇺 Australia

#### University of Melbourne
- **Architecture**: findanexpert SPA (Pure Portal variant) — /display/person-* returns same SPA shell for all slugs ⚠️; /profile/{id}/{slug} has unique individual profile IDs but still 0-byte SPA via curl
- **Layer**: L3 (curl useless for content; browser required)
- **Access**: curl returns 200 with 0 bytes (SPA shell). MSD pages (msd.unimelb.edu.au) return 403 via curl. Browser renders findanexpert correctly.
- **Key endpoint**: `https://findanexpert.unimelb.edu.au/profile/{id}/{slug}` — individual profile pages with unique IDs
- **Individual profile URL pattern**: `https://findanexpert.unimelb.edu.au/profile/{numeric_id}/{name-slug}`
- **Staff directory**: `https://findanexpert.unimelb.edu.au/college/1/faculty-of-architecture-building-and-planning`
- **PhD info**: `https://study.unimelb.edu.au/find/study-areas/architecture-building-and-planning/`
- **⚠️ CRITICAL**: /display/person-* endpoint matches ANY slug with same SPA shell → DO NOT use to verify affiliation. Use /profile/{id} format or browser verification.
- **⚠️ David Beynon lesson**: findanexpert returned 200 for "person-david-beynon" but he was actually at UTAS. Always browser-verify or use /profile/{id} format.
- **Failed**: /display/ endpoint is unreliable; MSD direct pages 403
- **Last verified**: 2026-06-30

#### University of Sydney
- **Architecture**: Symplectic Elements Discovery portal (profiles.sydney.edu.au). All individual profiles and group pages are pure SPAs (2118 bytes shell). CSS/JS loaded dynamically.
- **Layer**: L3 (Symplectic Elements SPA)
- **Best method**: Main education page (sydney.edu.au/arts/schools/sydney-school-of-education-and-social-work.html) returns 67KB with some profile links embedded. Individual profiles require browser verification.
- **Key endpoint**: profiles.sydney.edu.au/{username} (SPA)
- **Failed**: All API paths (v1/api/individuals, api/rest/v1, etc.), model.json feeds, JSON/RDF format parameters, print view
- **Last verified**: 2026-06-30

#### UNSW Sydney
- **Architecture**: Adobe Experience Manager (AEM) SPA for listing pages. Individual staff profiles at unsw.edu.au/staff/{slug} return real content (100-500KB) with research data embedded in HTML. Research profile pages (research.unsw.edu.au/people/{slug}) also accessible. Main domain may trigger Cloudflare, but staff sub-pages consistently work with curl.
- **Layer**: L2 (Adobe AEM SPA, staff profiles accessible)
- **Best method**: Use known staff names to construct unsw.edu.au/staff/{slug} URLs. Parse HTML for research keywords, publications, grants. School of Education research strengths include "Sociology of education." VERIFIED 2026-07-01 with 6 profiles.
- **Key endpoint**: https://www.unsw.edu.au/staff/{slug}
- **PhD application**: https://www.unsw.edu.au/study/research-degrees
- **Note**: Ben Newell moved from UCL/others → UNSW; not at Carleton
- **Failed**: AEM model.json returns minimal structure only; Our People listing page is SPA; newsroom expert finder API returns HTML only; newsroom.unsw.edu.au/find-an-expert redirects/times out; research.unsw.edu.au/people search returns SPA
- **Last verified**: 2026-07-01

#### QUT
- **Architecture**: Cloudflare WAF (全站)
- **Layer**: L3
- **Access**: BLOCKED — needs manual browser from user side
- **Key endpoint**: https://www.qut.edu.au/research/our-people
- **Failed**: curl + browser both blocked
- **Last verified**: 2026-06-26

#### Macquarie University
- **Architecture**: Pure research portal (researchers.mq.edu.au)
- **Access**: https://researchers.mq.edu.au/en/persons/{name}/
- **Key endpoint**: https://researchers.mq.edu.au/en/persons/abas-mirzaei/
- **Last verified**: 2026-06-26

#### RMIT University
- **Architecture**: Pure Portal (academics) + Cloudflare WAF
- **Layer**: L3
- **Access**: Cloudflare WAF blocks all curl access; browser required
- **Key endpoint**: https://www.rmit.edu.au/contact/staff-contacts/academic-staff/
- **Profile URL pattern**: academics.rmit.edu.au/{name-slug} (SPA)
- **Last verified**: 2026-07-01

#### University of Technology Sydney (UTS)
- **Architecture**: UTS Profiles (SPA) + Cloudflare/Imperva
- **Layer**: L3
- **Access**: Cloudflare/Imperva blocks all curl access; browser required
- **Key endpoint**: profiles.uts.edu.au/{Firstname}.{Lastname}/about (SPA shell)
- **Last verified**: 2026-07-01

#### Deakin University
- **Architecture**: Pure Portal SPA (experts.deakin.edu.au) + Cloudflare WAF (deakin.edu.au CMS pages)
- **Layer**: L3
- **Access**: experts.deakin.edu.au/{slug} returns "Discovery" SPA shell (~2KB). Individual profiles require browser. deakin.edu.au/about-deakin/people blocked by Cloudflare.
- **Key endpoint**: https://experts.deakin.edu.au/{slug}
- **PhD application**: https://www.deakin.edu.au/study/find-a-course/research-degrees
- **Note**: experts.deakin.edu.au is the consistent URL pattern.
- **Failed**: deakin.edu.au/about-deakin/people/{name} → 403 Cloudflare
- **Last verified**: 2026-07-01

#### Curtin University (2026-07-03 update — SPA CORRECTION)

⚠️ **Previous assessment was incorrect.** staffportal.curtin.edu.au is a **pure React SPA** — all profile pages return ~2KB empty HTML shells to WebFetch/curl, even though browsers render them fine. about/our-people also uses JS accordions with no static content.

- **Architecture**: React SPA (staffportal) + WordPress (personal sites) + JS accordion (about pages)
- **Layer**: L0-SPA (SPA Workaround — skip staffportal entirely, use ALTERNATIVE sources)
- **Accessible** (WORKING):
  - **Personal academic websites**: tamaleaver.net, wishcrys.com (WordPress, static HTML, full content)
  - **Research center pages**: digitalchild.org.au/team-members/ (ARC Centre — server-rendered, full profiles)
  - **ResearchGate**: profiles with publications, stats, research areas (server-rendered)
  - **Google Scholar**: search snippets give research tags + citation counts
  - **ORCID**: structured profile data
- **Blocked**: staffportal.curtin.edu.au ALL pages (SPA shell, ~2KB, zero extractable content), about/our-people (JS accordion, no names visible)
- **SPA Workaround Strategy** (L0):
  1. WebSearch `"[Name] Curtin University professor research"` → find alternative domains
  2. Prioritize: personal WordPress sites > research center pages > ResearchGate > Google Scholar
  3. WebFetch alternative sources for detailed content
  4. Combine data: personal site for research areas + ResearchGate for publications + Scholar for citations
- **Verified examples**:
  - Tama Leaver: tamaleaver.net (full profile, research, books, teaching awards) + digitalchild.org.au (CI role)
  - Crystal Abidin: wishcrys.com (full profile, 250+ pubs, h-index 42, awards) + ierlab.com (lab director)
  - Michele Willson: researchgate.net (43 pubs, Monash PhD, research areas) + ORCID
  - Mike Kent: Google Scholar snippet (cited 2,517, Internet Studies/Disability Studies/eLearning) + ResearchGate
- **Last verified**: 2026-07-03

### 🇭🇰 Hong Kong

#### University of Hong Kong (HKU)
- **Architecture**: Static HTML
- **Layer**: L1
- **Access**: browser → psychology.hku.hk faculty page; business school via hkubs.hku.hk
- **Key endpoint**: https://psychology.hku.hk/faculty-members/
- **Failed**: —
- **Last verified**: 2026-06-26

#### Chinese University of Hong Kong (CUHK)
**CRITICAL — Must follow this workflow for ALL CUHK departments:**

1. **Always start from the staff listing page** (all are server-rendered static HTML, L1)
2. **Extract every name + profile URL from the listing page** — do NOT construct URLs from memory or search snippets
3. **Verify each person is still in the current listing** before writing to Vika

| Department | Staff Listing URL | Individual Profile Pattern | Slug Format |
|-----------|-------------------|---------------------------|-------------|
| Psychology | `psy.cuhk.edu.hk/en/people/faculty.html` | `faculty-members/{slug}.html` | Firstname-lastname (e.g. `qian-wang`, `helene-fung`). Note: NO `prof-` prefix |
| Social Work | `web.swk.cuhk.edu.hk/en-gb/people/full-time-teaching-staff` | `full-time-teaching-staff/{id}-prof-{slug}` | Numeric ID prefix (e.g. `144-prof-ngai-sek-yum-steven`) |
| Educational Psychology | `fed.cuhk.edu.hk/eps/en/people/academic-staff/` | `eps/en/peoples/{slug}/` | `prof-{firstname}-{lastname}` (e.g. `prof-hawk-skyler`, `prof-chui-tsz-yeung-harold`) |
| MAE (Engineering) | Uses `/peoples/` NOT `/people/` for individual profiles | `peoples/{slug}/` | Surname-first (e.g. `lau-darwin-tat-ming`) |
| Statistics & Data Science | `sta.cuhk.edu.hk/pcat/faculty/` | `peoples/{slug}/` | Abbreviated slug (e.g. `jwang`, `hywong`, `bdai`, `ylin`, `ligen`, `tonysit`, `cyyau`, `ywei`, `jiachengzh`, `zxlin`) |

**STA 特殊规则（2026-07-13 验证）：**
- `/people/faculty/{name}/` 是 SPA 壳 — 所有自动化工具拿到相同首页 HTML（114KB），必须用 `/peoples/{slug}/` 替代
- pcat/faculty 列表页是静态 HTML，可直接抓取获取正确 slug
- 个人 `/peoples/` 页面为静态 HTML，包含完整研究方向信息

**Common pitfalls (learned 2026-07):**
- Psych dept changed slug format — removed `prof-` prefix from URLs (e.g. `prof-wang-qian` → `qian-wang`)
- SWK dept completely rebuilt their site — old `/en/people/teaching-staff/` → new `/en-gb/people/full-time-teaching-staff/{ID}-{slug}`
- Education dept moved from `/en/people/` → `/eps/en/peoples/`
- Several professors listed on old sites had moved to other institutions (HKU, etc.) — always cross-check against current listing page
- **Do NOT trust search engine snippets alone** — always open the listing page to confirm the person is still there

- **Architecture**: Static HTML (all 3 departments above) + JS-rendered (business school)
- **Layer**: L1 (Psych, SWK, Edu Psych) / L2-L3 (Business)
- **Failed**: Psych/SWK/Edu pages occasionally change URL patterns without redirect; always verify via listing page
- **Last verified**: 2026-07-07
- 

#### HKUST
- **Architecture**: Static HTML
- **Layer**: L2→L3
- **Access**: Web accessible; no Psychology dept (only Division of Social Science). Target Business School Marketing dept.
- **Key endpoint**: https://mark.hkust.edu.hk/faculty-and-staff/directory (Business School Marketing)
- **Also**: https://bm.hkust.edu.hk/faculty (Business School general) — has "Leadership and Behavioral Decision-making" research area. Management dept may have additional behavioral faculty.
- **PhD link**: https://prog-crs.ust.hk/pgprog/2022-23/mphil-phd-mark
- **Failed**: No Psychology PhD
- **Last verified**: 2026-06-26

#### City University of Hong Kong (CityU)
- **Architecture**: Incapsula WAF (main domain + stfprofile) + Cloudflare WAF (scholars subdomain)
- **Layer**: L3→L4 (Fully blocked from curl/WebFetch/urllib; search engine fallback required)
- **WAF 现状（2026-07-13 验证）**:
  - CityU 全站 Incapsula WAF，自动化工具（curl、Python urllib、WebFetch）**全部被拦截**
  - stfprofile 页面：200 返回 Incapsula JS 挑战页（约 200-1000 字节），不是真正死链
  - scholars.cityu.edu.hk：**403 Forbidden**（自动化访问），浏览器正常
  - **自动化审计脚本检测 CityU 链接时必然全部报死链/SPA壳，这是已知的系统性问题**
  - 链接有效性必须由人在浏览器中确认，不要因为 WAF 拦截就删除记录
- **Primary workflow — Scholars Portal (ALL departments, mandatory)**:
  1. 浏览器打开 https://scholars.cityu.edu.hk/en/persons/ — CityU 统一教师数据库
  2. 搜索导师姓名
  3. 打开个人页面，复制 URL（格式: `scholars.cityu.edu.hk/en/persons/{name}({uuid})`）
  4. 写入 Vika 的 `导师主页` 列
  5. **备用格式**: stfprofile 页面 `www.cityu.edu.hk/stfprofile/{slug}.htm`，优先级高于 scholars（因为至少返回 200 而非 403）
- **写入后复验规则（WAF 调整版）**:
  - WebFetch 验证 CityU 链接时，预期返回：stfprofile → 200 + ~200-1000字节（JS挑战）；scholars → 403
  - 以上两种结果都不是死链信号，**不删除记录**
  - **仅以下情况才删除**：Google/DDG 搜不到该导师、系页面已移除、或浏览器确认 404
  - 交付前提示用户在浏览器中抽查 CityU 链接
- **自动化审计的 CityU 豁免**：
  - 计算链接通过率时，可将 CityU 记录从分母中扣除
  - 例如：55 条中 15 条 CityU，非 CityU 40 条全通过 → 实际链接率 100%（而非 73%）
  - WAF 拦截 ≠ 链接质量缺陷
- **Subdomain probe results** (2026-07-13):
  - www.cityu.edu.hk/stfprofile/*: Incapsula WAF（200 + 约 200-1000 字节 JS 挑战）
  - scholars.cityu.edu.hk: 403 Forbidden（自动化工具）；浏览器正常
  - www6.cityu.edu.hk: Incapsula
  - www2/www3.cityu.edu.hk: SSL EOF
  - sgs.cityu.edu.hk: SSL EOF
  - cb.cityu.edu.hk/staff/: 可直接访问（商学院）
- **ORCID Discovery Method** (when all search engines fail):
  1. `curl "https://pub.orcid.org/v3.0/search/?q=affiliation-org-name:%22City+University+of+Hong+Kong%22"` → get ORCID IDs
  2. `curl "https://pub.orcid.org/v3.0/{orcid}/person"` → get researcher names and details
  3. Cross-reference found names with target department
  4. Search found names on scholars.cityu.edu.hk for profile verification (in browser)
  5. If profile found, write to Vika; if not, mark with note in 备注
- **Failed**: 全部自动化方案对 CityU 主页/教师资料页无效；ssweb.cityu.edu.hk blocked by Incapsula
- **Last verified**: 2026-07-13

#### Hong Kong Polytechnic University (PolyU)
- **Architecture**: JS-rendered
- **Layer**: L3
- **Access**: polyu.edu.hk/apss/ for Applied Social Sciences; polyu.edu.hk/mm/ for Marketing
- **Key endpoint**: https://www.polyu.edu.hk/apss/people/academic-staff/
- **PhD link**: https://www.polyu.edu.hk/apss/study/research-postgraduate-programme/
- **Individual profile pattern**: `polyu.edu.hk/apss/people/academic-staff/prof-{slug}/`
- **Note**: Recently posted JDM faculty job on SJDM, confirms active JDM research presence
- **Failed**: JS-rendered needs browser
- **Last verified**: 2026-06-26

#### Hong Kong Baptist University (HKBU)
- **Architecture**: Adobe AEM (main domain) + Pure Portal SPA (scholars portal) + separate infra for comm subdomain
- **Layer**: L3→L4 (Scholars portal blocked, comm subdomain network-unreachable)
- **Access**: Psychology lives within Dept of Education and Psychology (educ.hkbu.edu.hk), NOT standalone psych dept. Also check Business School at busrpg.hkbu.edu.hk.
- **Key endpoint**: https://educ.hkbu.edu.hk/?page_id=20398
- **PhD link**: https://educ.hkbu.edu.hk/?page_id=20338
- **PhD programme (Communication)**: gs.hkbu.edu.hk/programmes/doctor-of-philosophy-master-of-philosophy-school-of-communication
- **Accessible**: gs.hkbu.edu.hk (Grad School), scholars.hkbu.edu.hk (Pure Portal, JS-only for people), hkbu.edu.hk (main domain, Adobe AEM — all guessed URLs return 63KB 404 templates)
- **Blocked**: socsc.hkbu.edu.hk → connection refused; comm.hkbu.edu.hk (SSL timeout — all protocols); scholars portal API (403); scholars portal UUID-based person pages return 31-byte empty shells
- **Note**: HKBU has the strongest communication school in HK.
- **Wayback Machine**: scholars organization page archived (235KB) but person data still JS-rendered in archive
- **Failed**: socsc.hkbu.edu.hk connection refused; browser attempt failed; comm.hkbu.edu.hk SSL timeout
- **Last verified**: 2026-07-01

#### Lingnan University
- **Architecture**: Static HTML (Cultural Studies department accessible; other subdomains may be blocked)
- **Layer**: L1 (Cultural Studies) / L4 (other subdomains — all URLs return 234 bytes)
- **Access (Psychology)**: ln.edu.hk/psy/ — fully accessible via browser; applied/counseling focus
  - Key endpoint: https://www.ln.edu.hk/psy/about-us/people/academic-staff
  - PhD link: https://www.ln.edu.hk/rpg/
- **Access (Cultural Studies)**: ln.edu.hk/cultural/ — direct curl works
  - Key endpoint: https://www.ln.edu.hk/cultural/people/faculty
  - Profile URL: https://www.ln.edu.hk/cultural/people/faculty/prof-{name-slug}
- **Blocked**: ln.edu.hk/daci, ln.edu.hk/visualstudies return 234 bytes; uni-directory returns 404; staffdir.htm too thin
- **Last verified**: 2026-07-01

#### Education University of Hong Kong (EdUHK)
- **Architecture**: Static HTML
- **Layer**: L1 (via browser)
- **Access**: eduhk.hk/ps/en/ for Psychology dept; accessible via browser
- **Key endpoint**: https://www.eduhk.hk/ps/en/aboutus.php?s=our_staff
- **PhD link**: https://www.eduhk.hk/gradsch/
- **Failed**: Previous attempt via network blocked; browser works
- **Last verified**: 2026-06-26

### 🇸🇬 Singapore

#### National University of Singapore (NUS)
- **Architecture**: Static HTML + custom profiles (fass.nus.edu.sg); Full SPA (cde.nus.edu.sg)
- **Layer**: L1 (fass.nus.edu.sg) / L3 (cde.nus.edu.sg)
- **Access**: fass.nus.edu.sg/psy for Psychology; bschool.nus.edu.sg for Business
- **Key endpoint**: https://fass.nus.edu.sg/psy/people/
- **NUS CDE (College of Design & Engineering)**: ALL cde.nus.edu.sg pages return SPA shells (212 chars). Use browser for individual profile verification. Individual staff pages accessible via /me/staff/{slug}/ (NOT /me/people/{slug}/). Example: cde.nus.edu.sg/me/staff/wang-hao-victor/ (slug: wang-hao-victor, NOT wang-hao). Discovery: DDG HTML search "Wang Hao NUS mechanical engineering".
- **Failed**: All CDE subpages return SPA via curl; Bing search returns encrypted URLs
- **Last verified**: 2026-06-30

#### Nanyang Technological University (NTU)
- **Architecture**: Static HTML
- **Layer**: L1
- **Access**: ntu.edu.sg/sss for Psychology; ntu.edu.sg/business for NBS
- **Key endpoint**: https://www.ntu.edu.sg/sss/about-us/our-people
- **Failed**: —
- **Last verified**: 2026-06-26

#### Singapore Management University (SMU)
- **Architecture**: SITE RESTRUCTURED 2025-2026; old socsc.smu.edu.sg profiles broken. New platform at faculty.smu.edu.sg (JS-rendered, search required).
- **Layer**: L1→L3 (old static → new JS-rendered)
- **Access**: New platform at https://faculty.smu.edu.sg/ (JS-rendered, search required)
- **Key endpoint**: https://faculty.smu.edu.sg/profile/david-chan (confirmed via Google)
- **Failed**: All socsc.smu.edu.sg individual profile URLs now broken
- **Last verified**: 2026-06-26

### 🇲🇴 Macau

#### University of Macau (UM)
- **Architecture**: Static HTML
- **Layer**: L1
- **Access**: fss.um.edu.mo for Psychology
- **Key endpoint**: https://fss.um.edu.mo/
- **Failed**: —
- **Last verified**: 2026-06-26

#### Macau University of Science and Technology (MUST)
- **Architecture**: Cloudflare WAF
- **Layer**: L3
- **Access**: BLOCKED all automated attempts. No psychology department exists — psych coursework embedded in FHSS.
- **Key endpoint**: —
- **Failed**: curl → 403 (Cloudflare); browser blocked; Google Scholar shows engineering-only decision-making research; Chinese search yields no psych JDM faculty
- **Last verified**: 2026-06-26

### 🇨🇳 Sino-foreign Joint Programs

#### CUHK Shenzhen
- **Architecture**: Static HTML
- **Layer**: L1
- **Access**: hss.cuhk.edu.cn for Applied Psychology
- **Key endpoint**: https://hss.cuhk.edu.cn/en/taxonomy/term/21
- **Failed**: —
- **Last verified**: 2026-06-26

#### NYU Shanghai
- **Architecture**: Static HTML
- **Layer**: L1
- **Access**: shanghai.nyu.edu for Psychology / Neural Science
- **Key endpoint**: https://shanghai.nyu.edu/academics/graduate/phd-programs
- **Failed**: —
- **Last verified**: 2026-06-26

#### HKUST(GZ)
- **Architecture**: Static HTML
- **Layer**: L1
- **Access**: Hub structure; Society Hub has 4 thrusts (CNCC, FTEC, IPE, UGOD). No behavioral/cognitive science thrust. No Psychology dept.
- **Key endpoint**: https://soch.hkust-gz.edu.cn/
- **Failed**: No psychology/behavioral science thrust area; no relevant department
- **Last verified**: 2026-06-26

#### BNBU / UIC (北师香港浸会大学, Zhuhai)
- **Architecture**: Static HTML (general) + Vue SPA (FHSS comm department)
- **Layer**: L1 (general) / L3 (FHSS comm)
- **Access**: 9 PhD programs exist but none in psychology. Applied Psychology is undergraduate-only. FHSS Communication department faculty page is hash-routed SPA.
- **Key endpoint**: https://gs.uic.edu.cn/ (graduate school) / https://fhss.bnbu.edu.cn/comm_en/faculty/faculty.htm (comm faculty SPA)
- **Profile URL**: https://fhss.bnbu.edu.cn/comm_en/faculty/faculty.htm#/{username}/en (hash fragment, SPA-rendered)
- **Failed**: No psychology PhD program; behavioral decision research happens in math/stats depts but not a dedicated track; individual HTML pages don't exist for comm (all SPA hash-routed)
- **⚠️ PhD supervision**: UIC is primarily a teaching institution. MPhil/PhD offered through HKBU co-supervision. Assistant Professors may have limited supervision capacity.
- **Last verified**: 2026-07-01

#### Duke Kunshan University
- **Architecture**: Static HTML
- **Layer**: L1
- **Access**: Masters-only institution; 5 Master's programs, zero PhD programs
- **Key endpoint**: https://dukekunshan.edu.cn/en/academics/graduate
- **Failed**: No PhD programs of any kind; Duke Durham has relevant programs but DKU does not
- **Last verified**: 2026-06-26

#### UNNC (Nottingham Ningbo)
- **Architecture**: Static HTML
- **Layer**: L1
- **Access**: No Psychology dept. Business School (NUBS China) has ICBBR (behavioural business research centre).
- **Key endpoint**: https://www.nottingham.edu.cn/en/business/
- **PhD link**: NUBS China PhD available for behavioural economics/consumer behavior route
- **Failed**: No psychology-specific PhD; business school only
- **Last verified**: 2026-06-26

#### XJTLU (Xi'an Jiaotong-Liverpool)
- **Architecture**: Static HTML
- **Layer**: L1
- **Access**: No Psychology dept in HSS (only China Studies, International Studies, Media, Linguistics). IBSS offers PhD in Business.
- **Key endpoint**: https://www.xjtlu.edu.cn/en/academics/schools/ibss
- **Failed**: No psychology department; IBSS business PhD only
- **Last verified**: 2026-06-26

### 🇳🇿 New Zealand

#### University of Auckland
- **Architecture**: Adobe AEM (www subdomain) + React SPA (profiles subdomain) + Imperva/Incapsula WAF
- **Layer**: L3→L4 (Mostly blocked; dead end for Communication/Advertising/Brand)
- **Media and Screen Studies**: 9 staff found via contact page, ALL film/TV/media studies focus — NOT communication/advertising/brand. Auckland has NO dedicated Communication/Advertising/Brand programme.
- **Accessible**: calendar.auckland.ac.nz (249KB), researchspace.auckland.ac.nz (203KB), arts about/contact page (290-325KB, accessible via curl but names in SPA)
- **Blocked**: profiles.auckland.ac.nz (JS shell 2122B), unidirectory.auckland.ac.nz (0B/SSL error), business school marketing dept (406/Imperva), most dept pages (403/406), all /our-people/ and /staff/ paths (406)
- **ORCID Discovery Method**: Found 16+ Auckland researchers via ORCID keyword search but employment data incomplete. Cross-reference with Google Scholar for verification.
  - `curl "https://pub.orcid.org/v3.0/search/?q=affiliation-org-name:%22University+of+Auckland%22+AND+(keyword:media+OR+keyword:film+OR+keyword:screen)"`
- **Best method**: None working for staff profiles. ORCID fallback yields names but incomplete data.
- **Manual URL** (last resort): https://www.auckland.ac.nz/en/arts/about-the-faculty/school-of-humanities/our-people/contact-an-academic/contact-an-academic-in-media-and-screen-studies.html
- **Failed**: All curl, browser, and search engine approaches exhausted; individual profile pages (SPA shells); GraphQL API; profiles search API
- **Last verified**: 2026-07-01

#### University of Otago
- **Architecture**: Static HTML behind Cloudflare WAF (curl returns 403)
- **Layer**: L3
- **Access**: In-app browser bypasses Cloudflare. Individual profile URLs follow https://www.otago.ac.nz/{dept}/staff/{name-slug} pattern.
- **Key endpoint**: https://www.otago.ac.nz/education/staff (Education) / https://www.otago.ac.nz/marketing/staff (Business School)
- **Failed**: All curl → Cloudflare 403
- **Last verified**: 2026-06-29

#### Victoria University of Wellington (VUW)
- **Architecture**: Hybrid — Squiz CMS for school pages (sim, sam, fhss) with server-rendered staff listings; Angular SPA for /explore/ and people.wgtn.ac.nz
- **Layer**: L1 (static pages) / L3 (Angular SPA profiles)
- **Best method**: School staff pages (e.g., /sam/about/staff) contain server-rendered staff data with data-cy email attributes. Extract emails via `grep -Eo 'data-cy="[^@]+@vuw\.ac\.nz"' | sort -u`. HTML contains embedded JSON in `<script>` tags (`parsedEmails` key) with firstName, lastName, hrOrgUnitName, hrJobTitle, researchInterests, people.wgtn.ac.nz profile links. Use people.wgtn.ac.nz/{username} for individual profile URLs (JS-rendered, content not accessible via curl).
- **Key endpoint**: `https://www.wgtn.ac.nz/sam/about/staff` (368KB server-rendered page) / `https://www.wgtn.ac.nz/fehps/about/our-people/school-of-education` (embedded JSON)
- **Profile URL**: `https://people.wgtn.ac.nz/{email.username}` (e.g., people.wgtn.ac.nz/geoff.stahl) — SPA shell, 4150 bytes
- **Note**: Communication school is under School of Communication, Journalism and Marketing within Wellington School of Business and Government. SoM staff page only shows management professors. Communication-specific page inaccessible (0 bytes). Wayback Machine: comm staff page not archived (2022-2024).
- **Failed**: people.wgtn.ac.nz profiles all return 4150B JS shells; researchers subdomain 404; seftms 404; /communication-studies returns 404; /explore/ pages are Angular SPA
- **Last verified**: 2026-07-01

#### University of Waikato
- **Architecture**: Static HTML staff list + Symplectic/Elsevier JS SPA for individual profiles
- **Layer**: L1 (staff directory) / L2 (Symplectic SPA for individuals)
- **Access**: Get names from static staff page; open Symplectic profiles in browser to verify research content (JS renders research areas).
- **Key endpoint**: https://www.waikato.ac.nz/about/faculties-schools/education/our-staff/
- **Profile URL**: profiles.waikato.ac.nz/{username} (Symplectic SPA, needs browser)
- **Last verified**: 2026-06-29

#### University of Canterbury
- **Architecture**: Modern SPA with redirects
- **Layer**: L1→L3 (redirects to generic study page)
- **Access**: Navigate from main site https://www.canterbury.ac.nz/ then find Education staff links
- **Failed**: Direct URLs redirect to generic study information pages; academic staff page 404; all staff-related URLs dead
- **Last verified**: 2026-06-29

### 🇪🇺 Europe

#### 🇸🇪 Sweden

##### Linköping University (LiU)
- **Architecture**: Employee ID system; JS-rendered profiles
- **Layer**: L3
- **Access**: https://liu.se/en/employee/{employee-id} (IDs are like danva85)
- **Key endpoint**: https://liu.se/en/employee/danva85 (Daniel Västfjäll)
- **Failed**: Employee search by name via URL doesn't work; ID guessing unreliable
- **Last verified**: 2026-06-26

##### Lund University
- **Architecture**: Research portal (portal.research.lu.se) — 403 to curl
- **Layer**: L3
- **Access**: Browser; https://www.nek.lu.se/en/{name} for economics dept
- **Key endpoint**: https://portal.research.lu.se/en/persons/erik-wengstrom
- **Note**: Erik Wengström moved from QUT → Lund
- **Last verified**: 2026-06-26

#### 🇦🇹 Austria

##### University of Vienna
- **Architecture**: German site (doktorat.univie.ac.at) works well; English redirects often 404. UCRIS Pure portal on separate subdomain (behind Cloudflare). u:find JS SPA at ufind.univie.ac.at.
- **Layer**: L2→L3 (UCRIS Pure Portal + DE site works, EN 404s)
- **Access (PhD info)**: Navigate from https://www.univie.ac.at/studium/ → "PhD / Doktorat" → "Zulassung zu Doktorats- und PhD-Studien" → https://doktorat.univie.ac.at/doktoratunivie/zulassung/
- **Access (individual profiles)**: Search Google for "[Name] University Vienna u:find" → find person ID → use `ufind.univie.ac.at/en/person.html?id=XXXX`. Check "Currently not an active member of staff" status on the profile.
- **Key endpoint**: https://doktorat.univie.ac.at/ (DE version) / https://ufind.univie.ac.at/en/person.html?id={numeric_id}
- **Language requirements**: Deutsch- und/oder Englischkenntnisse auf Niveau C1. "Abschlusszeugnisse und optionaler Sprachnachweis" — language certificate is optional, depends on department. For Bildungswissenschaft (Education), English C1 likely sufficient.
- **Failed**: ufind search.html URLs don't work; need direct person ID; UCRIS blocked by Cloudflare; curl 403, browser also blocked by Cloudflare challenge
- **Last verified**: 2026-06-29

##### University of Innsbruck
- **Architecture**: IEZW site protected by Anubis web firewall (v1.25.0)
- **Layer**: L3 (Anubis anti-bot)
- **Access**: Not accessible via automated tools. Manual search needed.
- **Failed**: curl 404 on team pages; browser blocked by Anubis; FODOK search returns 410
- **Last verified**: 2026-06-29

#### 🇨🇭 Switzerland

##### University of Geneva (FPSE)
- **Architecture**: Static HTML with JS enhancements
- **Layer**: L1 (Static HTML for team pages)
- **Access**: curl the SSED research groups page — contains all research group and professor listings in static HTML. Individual profile URLs need browser (JS-rendered).
- **Key endpoint**: https://www.unige.ch/fapse/recherche/ssed
- **Failed**: EN version URL has redirect loop; personal pages may be JS-rendered
- **Last verified**: 2026-06-29

##### University of Basel
- **Architecture**: Static HTML, well-organized German site. English site sometimes 404. Main site (unibas.ch) works but education-specific subdomains (edu.unibas.ch) fail with SSL.
- **Layer**: L1 (main site) + L3 (edu.unibas.ch SSL fails)
- **Access (PhD info)**: Navigate from https://www.unibas.ch/de/Studium.html → "Anmeldung & Zulassung → Zulassung → Doktorat". Key URL: https://www.unibas.ch/de/Studium/Vor-dem-Studium/Bewerbung-Zulassung/Zulassung/Zulassung-zum-Doktorat.html
- **Language requirements**: "Die hauptsächlichen Unterrichtssprachen sind Deutsch und Englisch. Mindestens Niveau C1 gemäss Europäischem Referenzrahmen erwartet." For doctoral programs, specific requirements in faculty Promotionsordnung.
- **Key endpoint (language)**: https://www.unibas.ch/de/Studium/Vor-dem-Studium/Bewerbung-Zulassung/Zulassung/Sprachkenntnisse.html
- **Failed**: curl SSL error on edu.unibas.ch, browser ERR_CONNECTION_CLOSED
- **Last verified**: 2026-06-29

##### University of Bern
- **Architecture**: Static HTML + JS search interface for staff A-Z. Main site works; erz.unibe.ch blocked by browser policy.
- **Layer**: L1-L2 (edu.unibe.ch works, erz.unibe.ch blocked)
- **Access (PhD info)**: Navigate from https://www.unibe.ch/ → "Studies → Doctorate". Key admission page: https://www.unibe.ch/studies/prospective_students/application_and_admission/application_and_admission_for_doctoral_studies/index_eng.html
- **Access (individual profiles)**: Use main site search (unibe.ch/suche/) to find individual professor pages. edu.unibe.ch/ueber_uns/personen_a___z/ has JS search interface (no static listing). Individual URLs at edu.unibe.ch/ueber_uns/personen_a___z/{dept}/prof_dr_{last}_{first}/
- **Language requirements**: Bachelor requires German C1 certificate. Master programs may require English (if English-taught). For doctoral level, requirements specified in faculty Promotionsordnung. General rule: German and English are main languages.
- **Key endpoint**: https://www.unibe.ch/unibe/portal/content/e1006/e15237/e1466586/Zulassungbedingungen_e_26-27_eng.pdf (admission PDF) / edu.unibe.ch/ueber_uns/personen_a___z/ (staff search)
- **Failed**: erz.unibe.ch blocked by browser policy
- **Last verified**: 2026-06-29

#### 🇩🇪 Germany

##### University of Hamburg
- **Architecture**: Static HTML with complex navigation structure. Education faculty (Fakultät für Erziehungswissenschaft) has restructured website. Most English URLs return 404.
- **Layer**: L3 (multiple 404s, navigation needed)
- **Access**: Start from main site (https://www.uni-hamburg.de/) then navigate to "Forschung → Karrierewege". The "Promovieren" link leads to https://www.uni-hamburg.de/forschung/karrierewege.html. Admission requirements at https://www.uni-hamburg.de/campuscenter/bewerbung/promotion.html. Use university person search (Personensuche) from main site — German URLs work better than English.
- **Key discovery path**: DE version works, EN version returns 404 on most sub-pages.
- **Language requirements**: English B2 required; German C1/C2 only for German-language program.
- **Failed**: /en/personal.html → 404; /personen/ → 404; most EW subpages broken
- **Last verified**: 2026-06-29

##### University of Tübingen
- **Architecture**: WiSo Faculty → IfE → Static HTML for department personal pages
- **Layer**: L1 (static Personal/Team pages)
- **Access**: Navigate from https://uni-tuebingen.de/ → "Fakultäten" → "Wirtschafts- und Sozialwissenschaftliche Fakultät" → "Fachbereich Sozialwissenschaften" → "Erziehungswissenschaft" → "Institut". Key personal pages:
  - Sozialpädagogik: .../abteilungen/sozialpaedagogik/personal/
  - Allgemeine Pädagogik: .../abteilungen/allgemeine-paedagogik/personal/
  - Individual profiles: .../institut/team/{name-slug}/
- **Key discovery**: Old URL de/242 returns 404; new path through WiSo Fakultät navigation works. 15 professors across 4 departments.
- **Failed**: Direct old URLs all 404; site completely restructured in 2025/2026
- **Last verified**: 2026-06-29

##### University of Cologne
- **Architecture**: Static HTML. Individual profile pages use numeric IDs, NOT name slugs.
- **Layer**: L1 (static HTML with numeric page IDs)
- **Access**: Search main site (hf.uni-koeln.de/suche/) for professor name → find numeric page ID (e.g., 33878 for Herzmann, 33835 for Proske). Individual URLs: https://www.hf.uni-koeln.de/{numeric_id}
- **Key discovery**: Name-slug URLs like /personen/petra-herzmann/ all return 404. Numeric IDs are the only working format.
- **Failed**: /personen/ directory browsing; /personen/{name}/ all 404; site search returns no JavaScript results in this environment
- **Last verified**: 2026-06-29

##### University of Freiburg (2026-06-30 update)
- **Architecture**: New uni-freiburg.de site (2025/2026 relaunch); old erzwiss domain returns 000/timeout
- **Layer**: L3→L4 (HISinOne getter-protected, site restructured)
- **Key discovery**: `uni-freiburg.link/personensuche` resolves to HISinOne Personensuche which loads publicly ("ohne Anmeldung am System nur öffentlich sichtbare Personen"). BUT: input fields are getter-protected — `Cannot set property value of [object Object] which has only a getter` — preventing any automated form manipulation. Same protection for Vorlesungsverzeichnis semester selector.
- **Working URLs**: Main page (uni-freiburg.de), Personensuche HISinOne (loads), VVZ (loads), Fakultäten overview page (uni-freiburg.de/universitaet/fakultaeten/)
- **Failed**: Erzwiss domain (timeout), search (0 results), philolfak domain (empty page), all faculty sub-pages (404), all form manipulation (getter-protected)
- **German university pattern confirmed**: Personensuche → HISinOne/QIS system, all protected from external automation
- **Last verified**: 2026-06-30

### 🇺🇸 US/Canada

#### Columbia Business School
- **Architecture**: Cloudflare WAF
- **Layer**: L3
- **Access**: Browser ONLY; business.columbia.edu/faculty/people/{name}
- **Key endpoint**: https://business.columbia.edu/faculty/people/eric-johnson
- **Failed**: curl → Cloudflare block
- **Last verified**: 2026-06-26

#### Princeton University
- **Architecture**: Cloudflare WAF (psych.princeton.edu)
- **Layer**: L3
- **Access**: Browser ONLY
- **Key endpoint**: https://psych.princeton.edu/people/elke-weber
- **Note**: Elke Weber moved from Columbia → Princeton
- **Last verified**: 2026-06-26

#### Harvard University — Psychology
- **Architecture**: Mixed; some profile URLs 404
- **Layer**: L3
- **Access**: Personal sites may be more reliable than dept pages
- **Key endpoint**: https://www.joshua-greene.net/ (personal site; dept URL 404)
- **Last verified**: 2026-06-26

#### Carnegie Mellon University — SDS
- **Architecture**: Static HTML
- **Layer**: L1
- **Access**: https://www.cmu.edu/dietrich/sds/people/faculty/{name}.html
- **Key endpoint**: https://www.cmu.edu/dietrich/sds/people/faculty/daniel-oppenheimer.html
- **Note**: Carey Morewedge moved from CMU → Boston University Questrom
- **Last verified**: 2026-06-26

#### Boston University — Questrom
- **Architecture**: Static HTML
- **Layer**: L1
- **Access**: https://www.bu.edu/questrom/profile/{name}/
- **Key endpoint**: https://www.bu.edu/questrom/profile/carey-morewedge/
- **Last verified**: 2026-06-26

#### McGill University — Psychology
- **Architecture**: Incapsula WAF (blocks all automated access)
- **Layer**: L3
- **Access**: Browser ONLY; mcgill.ca/psychology/{name}
- **Key endpoint**: https://www.mcgill.ca/psychology/ross-otto
- **Failed**: All curl → 403; search blocked; faculty directory 403
- **Last verified**: 2026-06-26

#### Columbia University — Engineering (APAM / IEOR)
- **Architecture**: Cloudflare challenge ("Just a moment..." JS page) on ALL Columbia web properties except `math.columbia.edu` — includes `apam/ieor.columbia.edu`, `engineering.columbia.edu`, and even `www.columbia.edu/~netid` personal pages
- **Layer**: L3
- **Access**: Browser ONLY; 403 challenge ≠ dead link. Verify live URLs via WebSearch (search engines index full content); confirm exact slug from search result URLs
- **2026-07 consolidation**: Engineering profiles moved from department sites to the central directory `www.engineering.columbia.edu/faculty-staff/directory/{slug}` (alias: `engineering.columbia.edu/faculty/{slug}`). Old `apam.columbia.edu/people/{slug}`, `apam/ieor.columbia.edu/faculty/{slug}` 404 in browser. APAM also has new root-level profiles `apam.columbia.edu/{slug}` (e.g. `/liliana-borcea`); APAM directory: `apam.columbia.edu/directory`. Some IEOR profiles still live at `ieor.columbia.edu/faculty/{slug}` (Sigman, Xunyu Zhou, Dieker) — mixed state, prefer the central engineering directory
- **Last verified**: 2026-07-17

#### Columbia University — Mathematics
- **Architecture**: Static HTML (NO WAF — the only Columbia property directly accessible)
- **Layer**: L1
- **Access**: Profile pattern `https://www.math.columbia.edu/people/faculty-by-rank/name/{slug}/` (trailing slash; verified for Corwin/Dubedat/Knizel/Karatzas/Nutz)
- **Note**: Old `/people/directory/name/{slug}` and `/people/directory/char/F/2` paths are dead/redirected. Faculty list: `/people/faculty-by-rank/`. Personal sites `math.columbia.edu/~{name}/` also work (no WAF)
- **Last verified**: 2026-07-17

#### Cornell University — ORIE (Duffield Engineering)
- **Architecture**: Static HTML, site migrated mid-2026
- **Layer**: L1
- **Access**: `https://www.duffield.cornell.edu/people/{slug}/`
- **Key endpoint**: https://www.duffield.cornell.edu/orie/faculty-staff/
- **Note**: Old `orie.cornell.edu/people/{slug}` URLs all 404; old `/faculty-directory/{slug}` and `engineering.cornell.edu/faculty-directory/{slug}` redirect to duffield. Some slugs include middle name: `andreea-c-minca`, `robert-a-jarrow`, `david-alan-goldberg`. Faculty-staff list page doubles as title reference (Professor/Associate/Assistant).
- **Last verified**: 2026-07-17

#### UT Austin — Mathematics
- **Architecture**: Static HTML, directory migrated
- **Layer**: L1
- **Access**: `https://math.utexas.edu/directory/{firstname-lastname}`
- **Note**: Old `/people/faculty-directory/{slug}` redirects to generic `/directory` (soft-404 for individuals); old `www.ma.utexas.edu/component/cobalt/item/...` URLs redirect correctly but are outdated. Faculty list: `https://math.utexas.edu/directory`.
- **Last verified**: 2026-07-17

#### University of Chicago — Statistics vs Mathematics
- **Architecture**: Static HTML
- **Layer**: L1
- **Access**: `https://stat.uchicago.edu/people/profile/{slug}/` for Stats appointments ONLY — Math-only faculty (Lawler, Souganidis) 404 there
- **Note**: Math dept personal pages live at `https://math.uchicago.edu/~{name}/` (small static sites, valid). Joint appointments (Gwynne) have stat profiles. `mathematics.uchicago.edu/people/faculty/` is 404 — do not guess it. CAM program: `cam.uchicago.edu` (occasional SSL flakiness, retry).
- **Last verified**: 2026-07-17

#### Stanford University — Mathematics / ICME / MS&E
- **Architecture**: Static HTML + Stanford Profiles (profiles.stanford.edu)
- **Layer**: L1
- **Access**: Faculty list `https://mathematics.stanford.edu/people/faculty-lecturers` (bare `/people` redirects to department-administration, wrong page)
- **Warning**: `profiles.stanford.edu/{firstname-lastname}` matches SAME-NAME people in other departments — e.g. `profiles.stanford.edu/daniel-spielman` is a Radiology professor, NOT the Yale CS Spielman. Always verify the page's stated department before using.
- **Last verified**: 2026-07-17

#### McGill University — Mathematics & Statistics
- **Architecture**: Static HTML (mathstat site, unlike Psychology's Incapsula)
- **Layer**: L1
- **Access**: `https://www.mcgill.ca/mathstat/{slug}` — slug may include middle initial (e.g. `eric-d-kolaczyk`); bare `firstname-lastname` 404s
- **Last verified**: 2026-07-17

#### UCLA — Mathematics
- **Architecture**: Static HTML, NO WAF
- **Layer**: L1
- **Access**: Personal pages `https://www.math.ucla.edu/~{name}/`
- **Trap**: `www.math.ucla.edu/people/ladder/{slug}` profile pages are **soft-404 empty shells** — all return the same ~12KB generic department template (title "UCLA Department of Mathematics", zero personal content) for every slug, real or not. 200 status means nothing here; check page content. Verified shells: cucuringu, andrea-bertozzi, shlyakhtenko, gangbo, killip. Working personal pages: `~mihai/`, `~bertozzi/`, `~shlyakht/` (Shlyakhtenko), `~wgangbo/` (Gangbo), `~killip/`, `~deanna/` (Needell), `seas.ucla.edu/~vandenbe` (Vandenberghe)
- **Last verified**: 2026-07-17

---

## Know-How: People Who Changed Institutions

Record of faculty moves discovered during verification. Check this before assuming a person is at their "known" institution.

| Name | Was at | Now at | When |
|------|--------|--------|------|
| Bahador Bahrami | UCL | LMU Munich + Royal Holloway | 2019 |
| Carey Morewedge | CMU | Boston University Questrom | ~2023 |
| Elke Weber | Columbia | Princeton | ~2023 |
| Yin WU | (listed as HKU) | PolyU | — |
| Ben Newell | UCL/Carleton(误) | UNSW Sydney | — |
| Erik Wengström | QUT | Lund University | ~2024 |
| Warren Mansell | Manchester | Curtin University | — |
| Neil Stewart | Warwick Psychology | Warwick Business School | — |
| Mihai Cucuringu | Oxford Statistics (Assoc Prof) | UCLA Mathematics (full Professor) | 2025-01 |
| Daniel Spielman (CS) | Yale (Sterling Professor, still there) | NOT at Stanford — Stanford profile with same name is a Radiologist | — |

## Know-How: URL Quality Gate

After EVERY batch write to Vika, run this scan:

```python
# Generic URL patterns that indicate a dept page, NOT an individual profile
BAD_PATTERNS = [
    '/faculty-members/', '/faculty.html', '/faculty-directory/',
    '/academic-staff/', '/our-people', '/about/faculty',
    '/people'  # unless followed by /name
]
# If 导师主页 matches any BAD_PATTERN, find the individual URL immediately
# If individual URL is JS-rendered, mark in 备注
```

### URL patterns by university (for quick lookup):

| University | Individual profile pattern |
|-----------|--------------------------|
| CUHK Psych | `psy.cuhk.edu.hk/en/people/faculty-members/{slug}.html` |
| PolyU APSS | `polyu.edu.hk/apss/people/academic-staff/prof-{slug}/` |
| NTU SSS | `dr.ntu.edu.sg/cris/rp/rp{id}` |
| SMU (new) | `faculty.smu.edu.sg/profile/{name}` |
| HKU Psych | `psychology.hku.hk/people/{slug}/` |
| HKU Business | `hkubs.hku.hk/people/{slug}/` |
| CityU | `scholars.cityu.edu.hk/en/persons/{name}({uuid})` — use Scholars Portal search first |
| UCL (new) | `profiles.ucl.ac.uk/{id}-{slug}` |
| UNSW | `unsw.edu.au/staff/{slug}` |
| LiU | `liu.se/en/employee/{employeeId}` (IDs like danva85) |
| Lund | `portal.research.lu.se/en/persons/{slug}` |
| McMaster | `experts.mcmaster.ca/people/{initials}` |
| Carleton | `carleton.ca/psychology/people/{slug}/` |

## PhD Program URL Patterns by University

When filling 博士申请信息, prefer department-level Research Postgraduate programme pages over generic graduate school pages. These have program-specific admission requirements and contact details.

| University | Preferred PhD URL Pattern | Example |
|-----------|--------------------------|---------|
| PolyU | Per-department: /{dept}/study/research-postgraduate-programmes/ | AAE works; ISE SPA -> fallback to /gs/prospective-students/research-postgraduate/ |
| CUHK | Per-department: /research-postgraduate-programme/ | www4.mae.cuhk.edu.hk/research-postgraduate-programme/ |
| HKUST | School-level: /academics/research-postgraduate | seng.hkust.edu.hk/academics/research-postgraduate |
| CityU | Per-department: /academic-programmes/master-philosophy-doctor-philosophy/admissions | cs.cityu.edu.hk/en/academic-programmes/.../admissions |
| NUS | Per-department: /graduate/research-programmes/ | cde.nus.edu.sg/me/graduate/research-programmes/ |
| NTU | General: /admissions/graduate | ntu.edu.sg/admissions/graduate |
| HKU | General: /gradsch.hku.hk/prospective_students | SPA - browser needed |

Rule: Always try /{dept}/study/research-postgraduate-programmes/ (PolyU) or /{dept}/research-postgraduate-programme/ (CUHK) first. If SPA/blocked, fall back to the university graduate school page.

Last verified: 2026-06-30

## ORCID Discovery Pattern (Cross-School)

When all other access methods (curl, browser, search engines) are blocked by WAF:
1. Use ORCID API with affiliation filter: `affiliation-org-name:"[University Name]"`
2. Add keyword filters: `AND+(keyword:media+OR+keyword:communication+OR+keyword:advertising)`
3. Resolve ORCID IDs → researcher names via `/v3.0/{orcid}/person`
4. Cross-reference with external knowledge for department confirmation
5. Note: ORCID employment data requires `/v3.0/{orcid}/employments` (separate endpoint)

**Limitations**:
- ORCID records rely on researchers self-filling data — many have NO employment data
- Department affiliation not always recorded
- Cannot verify research fit without employment context

## HISinOne Tree Bypass Pattern (Freiburg Discovery)

HISinOne tree expand controls are `<input type="image">` elements (not `<button>`). All automated click methods blocked:
- `.click()` → method doesn't exist on input[type="image"]
- `dispatchEvent(new MouseEvent('click'))` → JSF ajax handler not triggered
- `getElementById(id).click()` → IDs regenerate per JSF lifecycle
- Playwright `locator.click()` → selector can't find dynamic tree elements

**Workaround for manual operator:**
1. Navigate to https://uni-freiburg.link/vvz
2. Select semester (e.g., Sommersemester 2026)
3. Expand "1.6 Philosophische Fakultät" by clicking ▶ arrow in browser
4. Browse child departments for Erziehungswissenschaft

This pattern applies to ALL German universities using HISinOne (Freiburg, Göttingen, partially Frankfurt).

## Dead End Diagnosis (exhaustive for Student B, 2026-06)

| School | Last Attempt | Barrier | Manual Bypass |
|--------|-------------|---------|---------------|
| Göttingen | eCampus HISinOne VL catalog | JS-rendered zero-content page; person search needs HRZ login | Google: "Erziehungswissenschaft Professor Uni Göttingen" |
| Frankfurt | QIS/LSF person tab | Needs HRZ login; old site 404 | Google: "Erziehungswissenschaft Professor Goethe Uni Frankfurt" |
| Freiburg | New site search / Fakultäten nav | Full site restructure (all old URLs 404); search returns 0 results; Fakultäten page 404 | Google: "Erziehungswissenschaft Professor Uni Freiburg" |
| Innsbruck | Anubis v1.25.0 anti-bot | Full site WAF; FODOK search returns 410 | Google: "IEZW Innsbruck Professor" |
| Bern | edu.unibe.ch Personen A-Z | JS-only search UI with no accessible API | Google: "Institut für Erziehungswissenschaft Uni Bern Professor" |
| Lausanne | SSP main page | No education-specific prof directory under SSP | Google: "sciences de l'éducation professeur UNIL" |
| Canterbury | Academic staff page | 404; all staff-related URLs dead | Google: "Education lecturer University of Canterbury" |

## Vika School IDs (in dstMNzQU9Aa58DpgW3)

| University | Record ID |
|-----------|----------|
| The University of Melbourne | reco6LXk2RdNg |
| The University of Sydney | recSCi2HYh18f |
| UNSW Sydney | recupYqG8sNJD |
| Australian National University | recd2grjzT9Vf |
| Monash University | recTyuPatx4kC |
| The University of Queensland | recPsoUbww7Zr |
| The University of Western Australia | recr6NH4J8ZxM |
| The University of Adelaide | recMJr4zMuQw8 |
| UTS | recx6JttsSALF |
| RMIT University | rec8y6BX5nvqK |
| Macquarie University | recys0of2k2QN |
| The University of Newcastle | recNh5iNGwa8V |
| Curtin University | recY5NYqlEXX0 |
| Deakin University | recP7hhnKGdHw |

### Melbourne findanexpert URL Pattern Updates (2026-07-01)

**DISCOVERY**: findanexpert.unimelb.edu.au has TWO URL patterns:

| Pattern | Behavior | Status |
|---------|----------|--------|
| `/display/person-{slug}` | Returns 200 SPA shell for **ANY** slug (even non-existent professors) | ❌ BROKEN - cannot verify existence |
| `/profile/{id}/{slug}` | Returns 200 SPA shell for **REAL** profiles only, 404 for fake ones | ✅ CORRECT - has unique profile ID |

**Profile ID discovery**: Each professor has a unique numeric profile ID. The `/profile/{id}/{slug}` URL contains this ID. These IDs can be found by:
- Searching via browser on findanexpert
- Checking Semantic Scholar for profile URLs
- Using the sub-agent approach that worked on 2026-07-01

**Verified profile IDs** (2026-07-01):
- Alan Pert: `/profile/11727/alan-pert`
- Sofia Prado: `/profile/52639/sofia-prado`
- Julie Willis: `/profile/4492/julie-willis`
- Amanda Achmadi: `/profile/48121/amanda-achmadi`
- Sidh Sintusingha: `/profile/66045/sidh-sintusingha`
- Gini Lee: `/profile/66008/gini-lee`
- Margaret Grose: `/profile/14457/margaret-grose`
- Andrew Saniga: `/profile/66248/andrew-saniga`

**David Beynon counter-example**: His `/display/person-david-beynon` returned the same SPA shell, but he's actually at University of Tasmania (UTAS), NOT Melbourne. The `/display/` endpoint accepts any slug and never returns 404 — it's a catch-all route in the React SPA.

**Lesson**: NEVER use the `/display/` endpoint for verification. Always find the `/profile/{id}/` URL. If you can't find the profile ID, mark the record as ⚠️学校归属未验证.

**Cloudflare WAF**: All findanexpert URLs return "Pardon Our Interruption" via curl (6183 bytes). They work correctly in browser. Same situation as UNSW staff pages.

#### Griffith University (2026-07-14)
- **Architecture**: Pure Portal (experts.griffith.edu.au)
- **Layer**: L1 (WebFetch accessible, full HTML content with research interests, supervision records, tags)
- **Best method**: WebSearch "[Name] Griffith University professor" → WebFetch the experts.griffith.edu.au profile page. Teaching tab shows PhD supervision details.
- **Key endpoint**: `https://experts.griffith.edu.au/{id}-{slug}/about` and `/teaching` for supervision records
- **Staff directory**: `https://www.griffith.edu.au/arts-education-law/griffith-law-school/staff`
- **PhD info**: `https://www.griffith.edu.au/apply/research-study`
- **Failed**: —
- **Last verified**: 2026-07-14

#### University of Tasmania (2026-07-14)
- **Architecture**: Pure Portal (discover.utas.edu.au)
- **Layer**: L1→L3 (profile about pages return "Discovery" SPA shell in some cases; teaching tab pages load with full supervision content)
- **Best method**: WebSearch → WebFetch `/teaching` subpage (not `/about`) for supervision records. Alternative: utas.edu.au profiles.
- **Key endpoint**: `https://discover.utas.edu.au/{Name}/teaching` for supervision; `https://discover.utas.edu.au/{Name}/about` may return SPA shell
- **Staff directory**: `https://www.utas.edu.au/law/people`
- **PhD info**: `https://www.utas.edu.au/research/degrees`
- **Note**: Dianne Nicol is Emeritus, not taking new students
- **Failed**: discover.utas.edu.au about pages inconsistent (some SPA shell, some load)
- **Last verified**: 2026-07-14

#### La Trobe University (2026-07-14)
- **Architecture**: Static HTML staff list + scholars.latrobe.edu.au Pure Portal
- **Layer**: L1 (staff listing page) / L3 (scholars portal profiles)
- **Best method**: Use latrobe.edu.au/law/staff for staff list (full HTML). scholars.latrobe.edu.au for individual profiles. LawTech page at latrobe.edu.au/cybersecurity/expertise/lawtech lists key tech-law faculty.
- **Key endpoint**: `https://www.latrobe.edu.au/law/staff` (staff list); `https://scholars.latrobe.edu.au/display/{username}` (individual)
- **PhD info**: `https://www.latrobe.edu.au/research/graduate-research/apply`
- **Failed**: scholars portal individual about pages may return minimal content
- **Last verified**: 2026-07-14

#### Swinburne University of Technology (2026-07-14)
- **Architecture**: Pure Portal (experts.swinburne.edu.au)
- **Layer**: L1 (WebFetch accessible, shows "Available to supervise Doctorate (PhD)" flag clearly)
- **Best method**: WebSearch → WebFetch experts.swinburne.edu.au profile; supervision availability clearly indicated
- **Key endpoint**: `https://experts.swinburne.edu.au/{id}-{slug}/about`
- **Staff directory**: `https://www.swinburne.edu.au/business-law/our-people/`
- **PhD info**: `https://www.swinburne.edu.au/course/research/doctor-of-philosophy-law/`
- **Note**: Dan Hunter (founding dean) has left — now at QUT/King's College London
- **Failed**: —
- **Last verified**: 2026-07-14

#### Western Sydney University (2026-07-14)
- **Architecture**: Pure Portal (researchers.westernsydney.edu.au)
- **Layer**: L1 (WebFetch accessible)
- **Best method**: WebSearch → WebFetch researchers.westernsydney.edu.au profile
- **Key endpoint**: `https://researchers.westernsydney.edu.au/en/persons/{slug}`
- **Staff directory**: `https://www.westernsydney.edu.au/schools/sol/key-people` (research-clusters page returned 406, 2026-07-15)
- **PhD info**: `https://www.westernsydney.edu.au/schools/sol/research/higher_degree_research` (406 on curl, WebFetch OK)
- **Failed**: research-clusters page (406), key-people is the correct staff directory
- **Last verified**: 2026-07-15

#### University of Canberra (2026-07-14)
- **Architecture**: Pure Portal (researchprofiles.canberra.edu.au)
- **Layer**: L1 (WebFetch accessible, shows "Accepting PhD Students" flag)
- **Best method**: WebSearch → WebFetch researchprofiles.canberra.edu.au profile; PhD acceptance status clearly shown
- **Key endpoint**: `https://researchprofiles.canberra.edu.au/en/persons/{slug}`
- **Staff directory**: `https://researchprofiles.canberra.edu.au/en/organisations/school-of-law-and-justice/persons/` (Pure Portal, 403 on curl but accessible via browser)
- **PhD info**: `https://www.canberra.edu.au/future-students/apply-to-uc/research-applications`
- **Note**: UC offers Professional Doctorate in Legal Science (Research) rather than traditional PhD; verify program type with student. Old staff dir `school-of-law-and-justice` returned 404 (2026-07-15).
- **Failed**: Older profile URLs at canberra.edu.au/research/... may 404
- **Last verified**: 2026-07-15

#### Flinders University (2026-07-14)
- **Architecture**: Static HTML + Pure Portal (researchnow.flinders.edu.au)
- **Layer**: L1
- **Best method**: WebSearch → PhD program page lists key supervisors with research areas
- **Key endpoint**: `https://www.flinders.edu.au/study/courses/doctor-philosophy-phd-law/` (lists supervisors)
- **PhD info**: `https://www.flinders.edu.au/study/courses/doctor-philosophy-phd-law/`
- **Note**: Law faculty small; no IP specialist. Vivienne Brand (corp gov + AI) and Marinella Marmo (transnational crime) are listed PhD supervisors. Robert Chalmers (IP + tech law) has personal page but no HDR supervision section — email to confirm PhD supervision eligibility.
- **Failed**: —
- **Last verified**: 2026-07-15

#### Bond University (2026-07-15)
- **Architecture**: SharePoint for staff profiles (bond.edu.au/staff → SharePoint login wall)
- **Layer**: L3 (all individual staff pages behind SharePoint; research.bond.edu.au Pure Portal for profiles)
- **Best method**: WebSearch → research.bond.edu.au profile pages; manual browser for staff directory
- **Key endpoint**: `https://research.bond.edu.au/en/persons/{id-slug}`
- **Staff directory**: `https://bond.edu.au/about-bond/academia/faculty-law/academic-staff/forum` (static HTML, verified)
- **PhD info**: `https://bond.edu.au/researchers/research-degrees/higher-degree-options/how-apply-hdr`
- **Note**: Bond profiles show "Accepting PhD Students" flag on research portal. Staff directory at bond.edu.au/staff just links to SharePoint login.
- **Failed**: bond.edu.au/nz/profile/... returns generic SharePoint directory; bond.edu.au/profile/... returns Access Denied
- **Last verified**: 2026-07-15

#### University of Southern Queensland (UniSQ) (2026-07-15)
- **Architecture**: Static HTML staff profiles + Pure Portal (research.usq.edu.au)
- **Layer**: L2 (staff profiles accessible, research portal has captcha)
- **Best method**: WebSearch → staffprofile.unisq.edu.au profile (bot protection may intermittently block); research.usq.edu.au researcher pages (captcha on automated access)
- **Key endpoint**: `https://staffprofile.unisq.edu.au/Profile/{name}` (bot protection, works in browser)
- **Staff directory**: `https://www.unisq.edu.au/about-unisq/schools/school-of-law-and-justice`
- **PhD info**: `https://www.unisq.edu.au/research/higher-degree-by-research/apply`
- **Note**: Eugenia Georgiades is Associate Professor (IP + privacy + cybersecurity), currently supervising 2 doctoral students. Staff profiles may intermittently block automated access.
- **Failed**: staffprofile.unisq.edu.au/departmentstaff/{id} → 404; research portal has captcha
- **Last verified**: 2026-07-15

#### Victoria University (2026-07-15)
- **Architecture**: Pure Portal (vuir.vu.edu.au) + Drupal CMS
- **Layer**: L1
- **Best method**: WebSearch → vu.edu.au profile pages
- **Key endpoint**: `https://www.vu.edu.au/about-vu/our-people/{slug}`
- **Staff directory**: `https://www.vu.edu.au/node/11362361` (Victoria Law School page, general — not a staff listing)
- **PhD info**: `https://www.vu.edu.au/node/10890586` (vu.edu.au/research/graduate-research returned 404, 2026-07-15)
- **Note**: Two strong IP/AI law supervisors: Christina Tao (IP + AI + legal tech) and Yongqiang Li (IP + disruptive tech, Mandarin speaker). Both have PhD supervision availability.
- **Failed**: vu.edu.au/research/graduate-research → 404; no dedicated law school staff listing page found
- **Last verified**: 2026-07-15

#### Charles Darwin University (2026-07-15)
- **Architecture**: Pure Portal (researchers.cdu.edu.au) + Drupal CMS
- **Layer**: L2 (some researcher pages accessible, others 404)
- **Best method**: WebSearch → cdu.edu.au or researchers.cdu.edu.au pages (unstable, URLs may change)
- **Key endpoint**: `https://researchers.cdu.edu.au/en/persons/{slug}/` (intermittently 404, 2026-07-15)
- **Staff directory**: `https://researchers.cdu.edu.au/en/organisations/law/` (10 staff listed)
- **PhD info**: `https://www.cdu.edu.au/research/research-degrees/how-apply`
- **Note**: Jenny Ng listed as "Registered to supervise postgraduate research" on researchers portal (RD3) but page 404 on RD4 recheck. CDU researchers portal highly unstable. Small law faculty, limited IP/criminal law specialists.
- **Failed**: cdu.edu.au/staff-directory → 404; cdu.edu.au/law → 404; cdu.online subdomain exists but non-standard
- **Last verified**: 2026-07-15

#### CQUniversity (2026-07-15)
- **Architecture**: Acquire repository (acquire.cqu.edu.au) + sportal staff pages
- **Layer**: L2 (acquire portal unstable, sportal requires login)
- **Best method**: WebSearch → acquire.cqu.edu.au author pages (may 404); sportal.cqu.edu.au as fallback
- **Key endpoint**: `https://acquire.cqu.edu.au/authors/{name}/{id}` (intermittently 404)
- **Staff directory**: `https://www.cqu.edu.au/about-us/our-schools/business-law/staff` (formerly `/about-us/school-business-law/staff` → 404 on 2026-07-15)
- **PhD info**: `https://www.cqu.edu.au/research/higher-degree/applying-for-a-hdr`
- **Note**: Amanda-Jane George (Senior Lecturer + Postgraduate Research Coordinator, IP + AI + patent law) has PhD from UQ and thesis cited by Australian High Court, but personal page on acquire.cqu.edu.au returns 404.
- **Failed**: acquire.cqu.edu.au author page → 404; my.cqu.edu.au/login → internal login portal; cqu.edu.au/apply → generic admission (not HDR)
- **Last verified**: 2026-07-15

#### Southern Cross University (SCU) (2026-07-15)
- **Architecture**: Standard university CMS
- **Layer**: L1
- **Best method**: WebSearch → scu.edu.au staff pages
- **Key endpoint**: N/A (only Lecturer-level law staff found)
- **Staff directory**: N/A
- **PhD info**: N/A
- **Note**: Law staff primarily Lecturer level; Brendon Murphy (Professor of Law, 2024) has no personal page URL on SCU site. Not recommended for PhD supervision targets.
- **Failed**: No suitable PhD supervisors found
- **Last verified**: 2026-07-15

#### Edith Cowan University (ECU) (2026-07-15)
- **Architecture**: Standard university CMS
- **Layer**: L1
- **Best method**: WebSearch → ecu.edu.au staff pages
- **Key endpoint**: N/A (only AI regulation / criminology staff found, no IP or criminal law doctrine specialists)
- **Staff directory**: N/A
- **PhD info**: N/A
- **Note**: ECU law school focuses on AI regulation and criminology/policing; Anne Wallace (former professor) has moved to La Trobe as Adjunct. No IP or pure criminal law doctrine supervisors found.
- **Failed**: No suitable PhD supervisors found
- **Last verified**: 2026-07-15

## AU Schools Access Layer Summary (2026-07-01)

| University | Architecture/CMS | Layer | Access Method | Key Endpoint / Profile URL Pattern |
|-----------|-----------------|-------|---------------|-------------------------------------|
| Melbourne | Pure Portal v5.x (React SPA) | L3 | Individual profile pages via findanexpert (browser required; Cloudflare WAF on curl) | findanexpert.unimelb.edu.au/profile/{id}/{slug} |
| Sydney | Symplectic Elements SPA | L3 | profiles.sydney.edu.au/{username} (SPA shell, 2118 bytes) | profiles.sydney.edu.au/{firstname}.{lastname} |
| UNSW | Adobe AEM (staff pages OK) | L2 | unsw.edu.au/staff/{slug} → real HTML content | unsw.edu.au/staff/{slug} |
| ANU | Pure Portal v5.x (React SPA) | L3 | React shell for all pages; individual titles contain researcher name in meta | researchers.anu.edu.au/researchers/{id-slug} |
| Monash | Pure Portal v5.x | L1 | research.monash.edu.au/en/persons/ accessible via web | research.monash.edu.au/en/persons/{name}/ |
| UQ | Drupal/UQ Experts (Numeric ID) | L3 | about.uq.edu.au/experts/{id} → real HTML but no name in URL | about.uq.edu.au/experts/{numeric_id} |
| UWA | Pure Portal (research-repository) | L3 | research-repository.uwa.edu.au/en/persons/{name}/ → SPA | research-repository.uwa.edu.au/en/persons/{name}/ |
| Adelaide | Pure Portal v5.x | L3 | researchers.adelaide.edu.au/profile/{name} → SPA | researchers.adelaide.edu.au/profile/{firstname}.{lastname} |
| UTS | UTS Profiles (SPA) + Cloudflare | L3 | Cloudflare/Imperva blocks all curl; browser required | profiles.uts.edu.au/{Firstname}.{Lastname}/about |
| RMIT | Pure Portal (academics) + Cloudflare | L3 | Cloudflare WAF blocks all curl; browser required | academics.rmit.edu.au/{name-slug} |
| Macquarie | Pure Portal (researchers) | L2 | researchers.mq.edu.au/en/persons/{id} → SPA | researchers.mq.edu.au/en/persons/{id} |
| Curtin | React SPA (staffportal) | L0-SPA | Use personal sites + ResearchGate + research center pages | tamaleaver.net, wishcrys.com, digitalchild.org.au |
| Wollongong | Symplectic Elements | L3 | scholars.uow.edu.au/{name}/about → SPA | scholars.uow.edu.au/{firstname}-{lastname}/about |
| Newcastle | Staff profiles | L2 | Staff profiles may be accessible | newcastle.edu.au/profile/{slug} |
| Deakin | Pure Portal SPA + Cloudflare | L3 | experts.deakin.edu.au/{slug} → SPA; Cloudflare blocks CMS | experts.deakin.edu.au/{slug} |
| Bond | SharePoint staff + Pure Portal research | L3 | research.bond.edu.au profiles accessible; staff pages behind SharePoint login | research.bond.edu.au/en/persons/{id-slug} |
| UniSQ | Static staff profiles + Pure Portal | L2 | staffprofile.unisq.edu.au (bot protection intermittent); research portal has captcha | staffprofile.unisq.edu.au/Profile/{name} |
| Victoria University | Pure Portal + Drupal | L1 | vu.edu.au profile pages accessible; old research/graduate-research URL → 404 | vu.edu.au/node/10890586 |
| CDU | Pure Portal (unstable) + Drupal | L2 | researchers.cdu.edu.au profiles intermittently 404; small law faculty | researchers.cdu.edu.au/en/persons/{slug}/ |
| CQU | Acquire repository + sportal | L2 | acquire.cqu.edu.au pages unstable; sportal requires login | acquire.cqu.edu.au/authors/{name}/{id} |
| SCU | Standard CMS | L1 | Only Lecturer-level law staff; no PhD supervisors found | — |
| ECU | Standard CMS | L1 | AI regulation/criminology focus; no IP or criminal law doctrine supervisors | — |

---

## Auto-Update Protocol

After searching EACH school, update this file. Do NOT defer to end of session — update immediately.

### Update Format

```markdown
#### School Name
- **Architecture**: [one of: Pure Portal | Vue SPA | React SPA | Static HTML | Worktribe | Cloudflare WAF | Custom CMS]
- **Layer**: L1
- **Access**: [concise: what worked]
- **Key endpoint**: [URL if API discovered]
- **Failed**: [what you tried that did NOT work]
- **Last verified**: [today's date]
```

### Rules

1. **Always record architecture** — even if you used a fallback. Knowing "Cloudflare WAF" tells the next session to skip curl and go straight to browser.
2. **Record failures** — "Tried API at /api/staff → 404" is AS VALUABLE as a success. It prevents the next session from wasting time on the same dead end.
3. **Update, don't duplicate** — if the school already has an entry, update its fields rather than adding a second entry.
4. **New schools go at the bottom** of their country section.
5. **After updating, save the file**. (Optional: if the skill directory is a git repo and the user consents, commit the changes. Do not auto-commit without confirmation.)

### Why this compounds

| Searches completed | Schools in registry | Hit rate on next search |
|-------------------|-------------------|------------------------|
| 5 | 5 | ~25% |
| 20 | 20 | ~60% |
| 50 | 50 | ~85% |
| 100 | 100 | ~95% |

After 100 schools, almost every new search hits a known architecture. The skill becomes 3-5x faster.
