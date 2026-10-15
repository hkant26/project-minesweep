# 项目排雷通用入口(project-minesweep)实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在 claude-review-skills repo 新增通用排雷入口技能 `project-minesweep`(判阶段+路由),README 重写为排雷定位,本地同装,冒烟后 repo 改名 `project-minesweep` 并推送。

**Architecture:** 入口是纯路由器——description 承载模糊触发词与让位子句,body 只做「拿材料→判阶段(三路信号)→路由→退出」;三引擎文件零改动;repo 改名放最后(冒烟通过才对外)。

**Tech Stack:** Claude Code skill(markdown + YAML frontmatter);gh CLI(rename/push);无代码无测试框架,验证=逐字校验+新会话触发冒烟。

**Spec:** `docs/superpowers/specs/2026-10-15-project-minesweep-entry-design.md`

**repo 工作目录:** `~/develop/claude-review-skills`(Task 5 改名为 `~/develop/project-minesweep`)

---

### Task 1: 入口技能 `project-minesweep/SKILL.md`

**Files:**
- Create: `~/develop/claude-review-skills/project-minesweep/SKILL.md`

- [ ] **Step 1: 写入完整文件**

```markdown
---
name: project-minesweep
description: 项目排雷(project-minesweep)——通用排雷入口:用户想给一个项目/方案/方向排雷,但没说或不知道该用哪种评审,以及一切阶段不明的模糊把关请求,先经本技能判阶段再路由到对的引擎。触发①明说 排雷/项目排雷/有什么雷/雷区扫一遍/帮我挑挑雷;②「这项目/这方向/这份 BP 靠谱吗」「上线/立项前帮我把把关」「帮我挑挑毛病」且未明说用哪种评审;③用户问「有没有一套通用的方案评审方法」「该用哪个评审」。本技能不生产分析,只做判阶段+路由——方向未定/只有想法 → hypothesis-challenge;方案定稿且要闯决策关(立项/答辩/融资/评审/客户选型) → red-team;方案定稿且即将执行 → pre-mortem;进行中的项目(无方案文本)→ 明说不在射程并给最近可用形态。Not for——明说 红队/red team/答辩预演/扮演反对者(→ red-team);明说 事前验尸/premortem/这方案会怎么死/上线前评审(→ pre-mortem);明说 假设地图/挖隐含假设/前提链条(→ hypothesis-challenge);网络安全渗透、代码或文档审查;还没有任何方向文本、要的是市场调研或头脑风暴。
---

# 项目排雷(通用入口)

给我一个项目、一份方案、甚至一个想法,我判断它处在哪个阶段,把你交给对的引擎:方向未定 → hypothesis-challenge 挖隐含假设;定稿闯关 → red-team 扮对手;定稿执行前 → pre-mortem 倒推死法。

本技能不生产分析。它的存在理由只有一个——「帮我排雷」这类请求里,用户往往不知道存在三个引擎、更不知道自己在哪个阶段;入口把这两个不知道一次解决,然后退出。入口自己动手做分析是越位:火力全部在引擎里,入口的火力是判对阶段。判错阶段等于用错引擎,整场排雷白做——所以第 1 步的纪律就是本技能的全部价值。

## 适用判断

- 适用:排雷类模糊请求——「帮我排雷」「这项目有什么雷」「靠谱吗」「上线/立项前把把关」「帮我挑挑毛病」——且用户没有明说用哪种评审;或用户明说不知道该用假设挑战/红队/事前验尸中的哪一个。
- 不适用:明说引擎关键词的直接让位(红队/答辩预演 → red-team;事前验尸/会怎么死 → pre-mortem;假设地图/挖隐含假设 → hypothesis-challenge);代码或文档审查、网络安全渗透;还没有任何方向文本、要的是市场调研或头脑风暴(没有可排的雷)。

## 流程

### 第 0 步:拿到材料

材料形态不限,先拿全:

- 文件路径 → 读文件;目录 → 读其中主要文档(方案书/设计文档/README,忽略缓存与无关产物)
- 粘贴的文本 → 直接用
- 前面对话里讨论中的想法 → 完整提取,向用户复述一句话摘要防漏抓

材料完全不存在(用户只有一个空泛念头、拿不出任何文本)→ 走射程外话术,结束,不要凭空发明项目。

### 第 1 步:判阶段(入口的唯一智能)

三路信号,交叉判定:

| 信号 | 方向未定 | 定稿·闯关 | 定稿·执行 |
|---|---|---|---|
| 用户原话 | 「该不该做」「还在讨论」「一个想法」 | 「要过会/立项/融资/答辩/客户选型」 | 「马上开干」「上线/迁移在即」 |
| 文本形态 | 几段讨论、一页提纲、无数字无排期 | 完整方案书 + 明确的评审对象 | 完整方案书 + 时间表/分工/里程碑 |
| 对面是谁 | 谁都不是(还没走到任何关口) | 评审席上有人打分 | 没人打分,对面是现实 |

判定规则:

1. 三路信号同向 → 直接锁定,不空转复述理由(路由说明里一句话带出即可)。
2. 信号冲突或全弱 → 一次问齐三选一,带推断默认值,复用引擎们的「背景锁定」问法,不开放式填空、不替用户瞎猜:

> 阶段确认(回复「确认」即按默认,或直接修改):项目处在哪个阶段?推断:<阶段与一句依据>——方向未定 / 定稿·闯关 / 定稿·执行

3. 用户既怕闯不过关、又怕执行死 → 双引擎:先 red-team 找伤口、后 pre-mortem 排死法,两轮独立输出,不合并(这是引擎们自己约定的顺序)。

### 第 2 步:路由并退出

1. 一句路由说明:「项目处于〈阶段〉——交给〈引擎名〉。」(阶段判不清时,先完成第 1 步的问齐再路由)
2. 调用对应引擎(引擎名逐字:`hypothesis-challenge` / `red-team` / `pre-mortem`),把材料一并交给它。
3. 退出。引擎自己做背景锁定、自己出全量报告;入口不总结、不补充、不加前言后语、不写文件。

## 射程外

两种形态各一句诚实边界:

> **进行中的项目**(已上线/跑了半年的系统,没有方案文本):排雷三引擎都吃方案文本(想法 → 定稿 → 计划),进行中的项目不在射程内。最近的可用形态:把项目现状与下一步计划写成一份文本,再跑 pre-mortem。
>
> **零文本请求**(只有一个空泛念头,拿不出任何文本):现在没有可排的雷。把想法落成几句话或一页提纲再回来——几句话就够,入口会交给 hypothesis-challenge。

不做任何「通用雷区扫描」兜底——对运行中的系统假装能扫雷是编造,不是排雷。

## 边界

- 只路由不分析;引擎产出原样交付。
- 引擎显式关键词让位,见 description 的 Not for。
- 对话内完成,不主动写文件。
- 用户拿不准、问「你们有什么评审手段」时,先用三句话展示三个引擎的分工,再回到第 0 步拿材料。
```

- [ ] **Step 2: 逐字校验 frontmatter**

Run: `sed -n '3p' ~/develop/claude-review-skills/project-minesweep/SKILL.md | sed 's/^description: //' | grep -c ':' || true`
Expected: `1`(仅「通用排雷入口:用户」一处合法裸标量;>1 才是失败)(description **值**内不得出现 ASCII 冒号;键前缀 `description: ` 的冒号必在,先剥前缀再数;全角 : 不受影响。实施后记:实测值内残留 1 处 ASCII 冒号「通用排雷入口:用户」,冒号后紧跟汉字非空格,YAML 裸标量合法,red-team 引擎 3 处同形既有在载,保留;Task 4 新会话冒烟为最终裁判)
Run: `grep -c '^description: ' ~/develop/claude-review-skills/project-minesweep/SKILL.md`
Expected: `1`

- [ ] **Step 3: Commit**

```bash
cd ~/develop/claude-review-skills && git add project-minesweep/ docs/ && git commit -m "feat: 项目排雷通用入口 project-minesweep(判阶段+路由,三引擎零改动)"
```

---

### Task 2: README 重写(排雷定位)

**Files:**
- Modify(整文件重写): `~/develop/claude-review-skills/README.md`

- [ ] **Step 1: 写入完整文件**

```markdown
# project-minesweep

给 Claude Code 的项目排雷套件——一个通用入口 + 三个引擎,把一份方案从「方向未定」到「定稿闯关」再到「执行前夜」的地雷全部挖出来:先挖地基,再对抗手,最后倒推死法。

## 排雷怎么用

装好后对 Claude 说「帮我排雷」就够了。入口(project-minesweep)会判断你的项目处在哪个阶段,自动交给对的引擎并退出;你也可以直接点名某个引擎。

## 一个入口 + 三个引擎

| 技能 | 时机 | 核心机制 | 一句话 |
|------|------|----------|--------|
| **project-minesweep**(入口) | 任何时候,说不清阶段就找它 | 判阶段 + 路由 | 「帮我排雷」「这项目靠谱吗」的统一落点,交给对的引擎后退出,自己不生产分析 |
| **hypothesis-challenge** | 方向未定、早期想法(几句话也算) | 崩塌烈度 ÷ 检验成本 | 挖出方案没写出来但必须成立的隐含假设,给出验证顺序、Top 3 与红线假设 |
| **red-team** | 方案定稿,即将闯决策关(立项/答辩/融资/评审/客户选型) | 利益对立的对手人设 | 扮演评审席上最危险的那个人,在真实对手出场前先被打一遍 |
| **pre-mortem** | 方案定稿,即将执行 | prospective hindsight(假定失败已发生) | 倒推致死因果链、最早征兆指标、可逆窗口与 ≤5% 成本的前置动作 |

入口的判定信号:你说的话(该不该做 / 要过会 / 马上开干)× 材料形态(几段讨论 / 完整方案书)× 对面是谁(没有关口 / 评审席 / 现实)。判不清时入口会一次性问清,不瞎猜。

**进行中的项目**(已上线、没有方案文本)不在射程内——入口会明说,并给出最近可用形态:把现状与下一步计划写成文本,再跑 pre-mortem。

三个引擎互为生命周期分工,显式点名时入口让位:拿定稿方案来挖假设照做但提示更对口的是红队;问「会怎么死」转 pre-mortem;要对手攻击转 red-team。

## 共同的质量合同

三个引擎共享同一套硬性约束,每条针对一种具体的失效模式:

1. **反谄媚** — 禁止「总体来看方向不错」式先扬后抑缓冲句
2. **禁通用风险** — 「市场变化」「执行不到位」这类放之四海皆准的空话一律作废重写,每条必须引用方案原文的具体段落/数字/时间节点
3. **打强不打弱** — 火力必须指向方案自认最强的支柱,不挑边角毛病凑数
4. **反稻草人** — 每条攻击附「作者最可能的有效反驳」并判定,反驳不成立的才算有效攻击
5. **可证伪** — 每条结论配检验动作(方法/时长/成本)与明确判据
6. **结论前置** — 唯一的「最致命一条」+「只能改一件事改什么」
7. **最强一环兜底** — 挖不出致命缺陷时,写明最强支柱是什么、为什么撑得住、唯一拷问路径是什么,不许「没有明显问题」敷衍结案

此外每个引擎都内置两类**廉价复核**:算术复核(方案自己的数字链条对不对得上)与外部事实核对(日期/法规时限/行业常识),查实的不一致直接构成独立发现。

## 安装

克隆后把四个技能目录复制到 Claude Code 的技能目录:

```bash
# 用户级(所有会话可用)
cp -R project-minesweep hypothesis-challenge red-team pre-mortem ~/.claude/skills/

# 或项目级(仅当前项目可用)
cp -R project-minesweep hypothesis-challenge red-team pre-mortem <项目>/.claude/skills/
```

入口与三引擎互相引用,建议整套安装。

## 使用

不需要任何命令——技能的触发描述覆盖了自然语言形态,直接说出你的意图即可:

- 「帮我**排雷**」「这项目**靠谱吗**」「立项前**把把关**」→ 入口判阶段后路由
- 「帮我把这份 BP 的**隐含假设**挖一遍」「我们在赌什么、哪些前提最悬」→ hypothesis-challenge
- 「**红队**这份方案」「扮演最苛刻的投资人拆解它」「答辩预演」→ red-team
- 「这个方案**会怎么死**」「上线前帮我挑挑致命伤」→ pre-mortem

方案(或想法)可以是文件路径、目录、粘贴的文本,或前面对话里刚讨论的内容。输出语言跟随方案语言:中文方案出中文报告,英文方案出英文报告。

## 目录结构

```
├── project-minesweep/SKILL.md        # 排雷入口:判阶段 + 路由
├── hypothesis-challenge/SKILL.md     # 假设挑战:挖隐含假设、排验证顺序
├── red-team/SKILL.md                 # 红队评审:对手视角攻击、闯关预演
└── pre-mortem/SKILL.md               # 事前验尸:倒推致死路径、征兆与前置动作
```
```

- [ ] **Step 2: 校验引擎文件未被误动**

Run: `cd ~/develop/claude-review-skills && git status --short hypothesis-challenge red-team pre-mortem`
Expected: 空(三引擎零改动)

- [ ] **Step 3: Commit**

```bash
cd ~/develop/claude-review-skills && git add README.md && git commit -m "docs: README 重写为项目排雷定位(入口+三引擎)"
```

---

### Task 3: 本地安装入口 + 记忆更新

**Files:**
- Create: `~/.claude/skills/project-minesweep/SKILL.md`(cp,与 repo 逐字相同)
- Delete: `~/.claude/projects/<本地项目>/memory/claude-review-skills-repo.md`
- Create: `~/.claude/projects/<本地项目>/memory/project-minesweep-repo.md`
- Modify: 同目录 `MEMORY.md`(替换对应行)

- [ ] **Step 1: 安装入口**

```bash
cp -R ~/develop/claude-review-skills/project-minesweep ~/.claude/skills/
diff ~/develop/claude-review-skills/project-minesweep/SKILL.md ~/.claude/skills/project-minesweep/SKILL.md && echo IDENTICAL
```
Expected: `IDENTICAL`

- [ ] **Step 2: 记忆换名换内容**

删除 `claude-review-skills-repo.md`,写入:

```markdown
---
name: project-minesweep-repo
description: 排雷技能套件 repo(hkant26/project-minesweep,2026-10-15 由 claude-review-skills 改名);源=~/.claude/skills 四件套,改后需手动同步推 repo
metadata:
  type: project
---

2026-10-15 repo 由 claude-review-skills 改名并重定位为**项目排雷**:通用入口 project-minesweep(判阶段+路由,2026-10-15 新增)+ 三引擎(hypothesis-challenge/red-team/pre-mortem,内容零改动)。

- 真源 = `~/.claude/skills/` 四件套;repo 工作副本 `~/develop/project-minesweep/`(Task 5 起)。改任何一件 → 手动 cp 到 repo 副本再 commit/push,两处不自动同步。
- 入口是纯路由器(射程外=进行中项目明说不在射程,无第四引擎);spec 在 repo `docs/superpowers/specs/2026-10-15-project-minesweep-entry-design.md`。
- repo 无 LICENSE(用户未决定)。
- 相关:[[review-skill-series-status]]
```

`MEMORY.md` 中旧行 `- [claude-review-skills repo 已发布](claude-review-skills-repo.md) — ...` 替换为:
`- [project-minesweep repo](project-minesweep-repo.md) — 排雷套件(入口+三引擎)公开在 hkant26/project-minesweep;源=~/.claude/skills 四件套,改后需手动同步推`

- [ ] **Step 3: 校验**

Run: `ls ~/.claude/skills/ | grep -c project-minesweep` → Expected: `1`

---

### Task 4: 触发冒烟(需新会话,用户协助)

- [ ] **Step 1: 用户开一个全新 Claude Code 会话,逐条输入下表短语**(每条可同一会话顺序测;第 1 条必须新会话——技能列表会话启动时载入)

| # | 输入短语(附材料) | 预期归属 |
|---|---|---|
| 1 | 「帮我排雷」+ 附任意一段方案文本 | project-minesweep 接住,路由说明一句,交给某引擎 |
| 2 | 「这个方向靠谱吗?」+ 几句早期想法 | 入口或 hypothesis-challenge 直接管(两者皆可接受,产出必须是假设挑战式报告) |
| 3 | 「红队这份 BP」+ 定稿方案 | red-team 直接接,入口不抢 |
| 4 | 「这方案会怎么死」+ 定稿方案 | pre-mortem 直接接,入口不抢 |
| 5 | 「帮我给这个跑了半年的系统排排雷」(无方案文本) | 入口射程外话术(诚实边界+最近可用形态),不发明扫描 |
| 6 | 半页提纲 + 「下周要去找投资人」 | 入口问齐三选一(信号冲突),不瞎猜 |
| 7 | 「我有个模糊念头,帮我排雷」(无任何文本) | 射程外话术,零文本分支指引 hypothesis-challenge(几句话就够)而非 pre-mortem |

- [ ] **Step 2: 结果判定**

全过 → 进 Task 5。有不过(典型:入口抢了 3/4,或 1/2 没人接)→ 只改入口 description 的让位/占位措辞,回到本任务重测;引擎文件不动。

---

### Task 5: repo 改名 + 推送(冒烟通过后)

**Files:**
- Rename: GitHub repo `hkant26/claude-review-skills` → `hkant26/project-minesweep`
- Rename: 本地目录 `~/develop/claude-review-skills` → `~/develop/project-minesweep`

- [ ] **Step 1: 改名(remote 自动跟随,旧 URL GitHub 重定向)**

```bash
cd ~/develop/claude-review-skills && gh repo rename project-minesweep --yes
git remote -v
```
Expected: remote 显示 `github.com:hkant26/project-minesweep.git`

- [ ] **Step 2: 本地目录改名**

```bash
mv ~/develop/claude-review-skills ~/develop/project-minesweep
```

- [ ] **Step 3: 推送**

```bash
cd ~/develop/project-minesweep && git push -u origin main
```

- [ ] **Step 4: 验证远端**

```bash
gh repo view hkant26/project-minesweep --json name,visibility -q '.name + " | " + .visibility'
gh api repos/hkant26/project-minesweep/contents -q '.[].path'
```
Expected: `project-minesweep | PUBLIC`;路径含 `README.md`、`project-minesweep`、`hypothesis-challenge`、`red-team`、`pre-mortem`、`docs`
