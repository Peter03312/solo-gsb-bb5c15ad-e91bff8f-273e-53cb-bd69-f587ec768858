# 联排提示板（Rehearsal CueBoard）

为剧场联排制作的 Web 提示板：舞监（Stage Manager）在草稿中录入提示项的计划开始时间、时长、负责岗位与前置关系，校验通过后发布**只读联排版本**；灯光 / 音响 / 道具等执行岗位在现场页面只登记**本岗位**提示项的实际开始、完成或预计结束时间，系统依据前置项的实际/预计结束自动重算未开始项，并展示顺延导致的岗位冲突。

- 后端：Node.js + Express + better-sqlite3（SQLite，同步事务）
- 前端：浏览器原生 HTML/CSS/JavaScript（无需构建）
- 本地账号体系：服务端 Session 登录，区分**舞监**与**执行岗位**
- 打包：Dockerfile + Docker Compose 一键启动（自动安装依赖、初始化数据）

## 一、快速启动（推荐，Docker 一键）

要求已安装 Docker 与 Docker Compose 插件。

```bash
docker compose up --build -d
```

启动过程会自动完成：

1. 构建镜像并安装 npm 依赖（含 better-sqlite3）；
2. 初始化 SQLite 数据库：建表、创建默认账号、写入一份可直接发布的示例草稿；
3. 在容器内启动 Web 服务。

浏览器访问：

> **http://localhost:3000**

如宿主机 3000 端口被占用，可指定其他端口：

```bash
PORT=8080 docker compose up --build -d
# 访问 http://localhost:8080
```

停止与查看日志：

```bash
docker compose logs -f
docker compose down            # 停止并移除容器（保留数据卷）
docker compose down -v         # 同时删除数据库数据卷（恢复出厂）
```

数据库文件持久化在 Docker 卷 `cueboard-data`（容器内 `/app/data/cueboard.db`）。

## 二、本地直接运行（无需 Docker）

要求 Node.js ≥ 18（推荐 20）。

```bash
npm install
npm run initdb     # 首次运行：建表 + 播种（重复执行安全、幂等）
npm start
```

浏览器访问 **http://localhost:3000**。

## 三、默认账号

首次启动自动创建（请在生产环境及时更改代码/重新播种密码）：

| 角色 | 用户名 | 密码 | 说明 |
| --- | --- | --- | --- |
| 舞监 | `sm` | `sm12345` | 草稿增删改、发布版本、查看全部 |
| 灯光执行 | `light` | `light123` | 仅可登记「灯光」岗位提示项 |
| 音响执行 | `sound` | `sound123` | 仅可登记「音响」岗位提示项 |
| 道具执行 | `props` | `props123` | 仅可登记「道具」岗位提示项 |

## 四、使用说明

### 舞监

1. **草稿编辑**页新增提示项：编号、岗位、计划开始（HH:MM）、时长（分钟）、说明；
2. 在「前置关系」区为提示项指定「某提示项完成后 → 本提示项」，可构成多前置链；
3. 页面实时显示校验结果。满足以下全部条件时才可发布：
   - 每个前置项的**计划结束 ≤ 后项计划开始**；
   - **同一岗位**的计划时段不重叠（首尾相接允许）；
   - 前置依赖**无环**；
   - 失败时明确列出每一项冲突（冲突双方、岗位、时间、环路径）；
4. 点击「发布联排版本」生成只读版本，现场随即以该版本为准；草稿可继续修改并发布新版本。

#### 发布新版前：旧执行版已完成登记交接（本轮新增）

发布面板的「旧执行版登记交接」区会在**发布前自动预览**（也可点击「重新预览」），对照**当前草稿与旧执行版（最新已发布版本）**逐条列出旧版中**已完成**提示项（旧登记同时有实际开始与实际完成）的交接资格：

- **可选条件（同时满足）**：草稿与旧版**同一提示项引用**（即草稿提示项 id）的**编号、说明、岗位、计划开始、时长、前置集合**六项完全一致，且旧登记**同时有实际开始和实际完成**；
- 不满足的项列入「不可选」，逐项注明不合格原因（如「计划开始不一致（旧版 19:12 → 草稿 19:13）」），复选框禁用；
- **无旧版（首次发布）或旧版无已完成项时，列表为空**，不影响正常发布。

舞监勾选要延续的项后点「发布联排版本」：

- 确认发布时服务器在**同一事务内重新核对**：①草稿版本号、②旧执行版（必须仍是当前执行版）、③每个所选登记行的**行版本号**、④资格是否仍成立；任一变化或资格失效则**整体拒绝**（HTTP 409 `HANDOVER_STALE`，响应逐项列出原因），不会产生新版本，请**重新预览**后再发布；
- 成功后**仅将所选项的实际开始、实际完成带入新版**（不延续预计结束、不复制更正历史），**未选项保持空登记**；新版对应行标注「↩ 交接自第 N 版·编号」标明来源版本与提示项；
- **旧版登记与更正历史原样保留、继续可回看**（「发布版本」页回看，含来源标注与更正记录）；已发布计划仍然冻结，交接只影响新版的现场登记初值；
- 未传 `handover` 字段的发布请求**完全维持原行为**（新版全部空登记）；计划冲突仍然阻止发布（422）。

#### 舞监现场更正（仅当前执行版本）

舞监在「现场联排」页可对任一行登记进行**更正**（点击该行「更正」按钮）：

- 一次只能更正或清除**实际开始 / 实际完成 / 预计结束**中的**任意一项**，并必须填写更正原因（≤200 字）；
- 更正**只改变现场登记**，不修改已发布计划（计划开始/时长/前置关系保持冻结）；
- 提交必须携带所见登记行的**行版本号**，且与岗位登记**共用同一版本校验**：岗位或舞监任一方先提交后，另一方持旧版本提交会收到 **409** 并提示刷新，不能覆盖新登记；
- **归档版本（已有更新版本发布）禁止更正**，现场页对应行显示「归档不可更正」；
- 联动约束（不满足则整体拒绝，不写入任何部分更正与历史）：
  - 更正后的实际完成必须有实际开始，且不得早于实际开始；
  - 清除实际开始时系统**同时清除实际完成**（级联清除，历史中标注「级联清除」）；
  - 保留的预计结束不得早于实际开始；若无实际开始则不得早于原计划开始；
  - 无变化的提交（前后值相同）视为无效。
- 更正成功后按现有规则**重算后续未开始项与同岗位冲突**（顺延不早于原计划，冲突在红条展示）；
- 每次更正都记录**更正项、更正前/更正后值、操作者、时间、原因**；点击行内「更正记录(N)」可回看，历史版本回看页中也可查看（只读）。

#### 现场岗位代班（本轮新增）

现场临时换人时，舞监在「现场联排」页的「岗位代班」面板为**当前执行版**的某个岗位指定一名**已有执行账号**代班：

- **启用 / 撤销**：选择岗位与执行账号后「启用代班」；在任代班行点「撤销」恢复原岗位账号。每次变更都记录**操作者与时间**（启用/撤销各一条），面板与「发布版本」回看页均可查看；
- **每个版本每个岗位至多一名在任代班**：重复启用返回 409 `DEPUTY_EXISTS`，需先撤销再启用；重复撤销返回 409 `NO_ACTIVE_DEPUTY`；
- **权限转移**：代班期间**原岗位账号失去该岗位登记权**（提交返回 403 `DEPUTY_ACTIVE`），**代班账号获得该岗位登记权且保留自己原有岗位权限**；撤销后原岗位账号立即恢复，代班账号持旧页面提交会被 403 拒绝且**不改动任何登记**；
- **仅当前执行版可变更**：启用/撤销只作用于当前执行版；**发布新版后代班不自动延续**（新版无代班），旧版代班记录在回看页**只读可查**；
- **明确报错不留部分记录**：未知账号（404 `UNKNOWN_ACCOUNT`）、非执行账号（400 `NOT_OPERATOR`）、当前版没有的岗位（400 `POSITION_NOT_IN_VERSION`）、页面版本过期（409 `VERSION_STALE`）均整体拒绝；
- 登记提交在**同一事务内**复核当前执行版、代班授权与登记行版本，原有登记请求响应、行版本冲突（409）与时间校验保持兼容。

#### 跳过依赖链（本轮新增，仅当前执行版）

联排中某项无法执行时，舞监在「现场联排」页点该行「跳过依赖链」，对**当前执行版**发起整链跳过：

1. **填原因 + 影响预览**：必须填写**非空原因**（≤200 字），点「预览影响」后系统沿前置关系（from → 后项）计算**该提示项及可达的全部后继**（多分支合并链不重复），逐项标明「将新跳过（带登记行版本号）/ 已跳过（只展示既有原因，不重复留痕）/ 阻断：已执行」；
2. **阻断即整次拒绝**：链中任一项**已有实际开始或实际完成**，或确认时**预览所见登记行版本发生变化**（期间有岗位登记或舞监更正），均返回 **409 `SKIP_BLOCKED`** 并逐项列出阻断原因，**不留任何部分标记**，需重新预览后再确认；预览存在阻断项时页面不显示确认按钮；
3. **确认后的效果（同一事务写入）**：
   - 整条依赖链首次跳过的项写入跳过留痕：**操作者、时间、原因**（同一次操作共用原因与时间戳）；
   - **清除各项预计结束**（`estimated_end` 置空），**已发布计划原样保留**（计划开始/时长/前置冻结）；跳过项的实际开始/完成在确认时必然为空（否则已被阻断）；
   - **执行账号不得再登记**被跳过项（start / complete / estimate 一律 409 `CUE_SKIPPED`），**舞监也不得更正其时间**（409 `CUE_SKIPPED`，跳过为终态，留痕不可改）；
   - 现场重算**不把跳过项计入待执行时段**：被跳过的前置不约束后继开始，跳过项**不参与同岗位冲突检测**；其他项仍按现有前置与登记规则重算（顺延不早于原计划）；
   - **已跳过项只展示、不重复留痕**：再次对包含已跳过项的链发起操作时，已跳过项展示既有原因但不参与行版本核对、不新增记录；整链均已跳过时确认返回 409 `SKIP_NOOP`；
4. **版本边界**：**仅当前执行版可操作**，携带过期 `versionId` 返回 409 `VERSION_STALE`；**发布新版本不继承跳过状态**（新版全部未跳过，已完成项的交接资格沿用现有交接约定）；**旧版回看可见**每项跳过原因/操作者/时间（只读），「发布版本」页与 `/api/versions/:id` 均可查。

页面上被跳过项以灰色整行 +「⛔ 已跳过」徽标展示，鼠标悬停可见原因；现场页、回看页一致。

### 执行岗位- 「现场联排」页只能看到当前执行版本，本岗位的行高亮并出现操作按钮：
  - **开始**：登记实际开始（默认取当前时间，可手填 HH:MM）；
  - **完成**：登记实际完成（须先开始，完成时刻不得早于开始）；
  - **预计结束**：未开始或已开始均可登记，不得早于（实际或原计划）开始；
- **代班识别**：若你被指定为某岗位代班，页面顶部黄色横幅提示「你正在代班岗位…」，该岗位行同样高亮并可登记（自己原有岗位权限保留）；若你的岗位被他人代班，横幅提示登记权已移交，本岗位行显示「已由 XX 代班」且不再出现操作按钮；
- 非本岗位提示项只读；对非最新（已归档）版本不能登记；
- **被舞监标记跳过的提示项（含其整条后继链）不可再登记**，行内显示「已跳过 · 操作者 · 时间」与原因，灰色只读；
- 已完成项保留实际时间不再变化；未开始项按前置链重算，**重算开始不早于原计划**；
- 顺延产生的同岗位时段冲突在页面顶部红色横幅逐条展示，同时相关行标注「已顺延」；
- 岗位登记可能被舞监按规定更正（见上节），更正与岗位登记共用行版本号，持旧版本提交会被 409 拒绝。

### 已发布版本回看

「发布版本」页列出全部历史版本（版本号、说明、发布时间、登记数量），点击「回看」查看该版本冻结的提示项、前置关系与当时/最新的现场重算结果；舞监还可在回看页查看每行的**更正记录**（更正前后值、操作者、时间、原因）与**跳过留痕**（原因、操作者、时间），归档版本为只读、不能更正或跳过。

### 版本号（乐观锁）规则

- 草稿维护全局**草稿版本号**，页面显示 `vN`。任何编辑、增删依赖、发布都必须携带页面加载时的版本号；他人已修改导致版本号过期时返回 **409** 并明确提示当前版本号，前端自动刷新，需要重新编辑后提交；
- 单条提示项的修改同时校验**行版本号**；
- 现场登记按「提示项行版本号」校验：两位岗位同时操作同一提示项时，先提交者成功，后提交者收到 409 明确报错并刷新；
- **舞监更正与岗位登记共用同一条登记行版本号**：任一方更新后行版本 +1，另一方（或持旧页面重复提交）会收到 409 拒绝并被要求刷新，确保更正不能覆盖新登记、新登记也不能覆盖更正；
- **跳过链确认同样校验链上每项登记行版本号**：预览后行版本变化则整次 409 `SKIP_BLOCKED`；确认跳过时清除预计结束的项行版本 +1，被跳过项再收到任何登记/更正一律 409 `CUE_SKIPPED`。

## 五、配置项（环境变量）

| 变量 | 默认值 | 说明 |
| --- | --- | --- |
| `PORT` | `3000` | Web 服务监听端口（容器内） |
| `DATA_DIR` | `/app/data`（镜像）/ `./data`（本地） | SQLite 数据库所在目录 |
| `DB_PATH` | `$DATA_DIR/cueboard.db` | 数据库文件完整路径 |
| `SESSION_SECRET` | 开发用默认值 | Session  Cookie 签名密钥，**生产必须修改** |
| `TZ` | `Asia/Shanghai` | 容器时区，影响「取当前时间」登记 |

通过 Compose 修改可在项目根目录创建 `.env`，例如：

```env
PORT=8080
SESSION_SECRET=your-long-random-secret
```

## 六、目录结构

```
.
├── Dockerfile
├── docker-compose.yml
├── entrypoint.sh          # 容器入口：先 initdb 再启动服务
├── package.json
├── src
│   ├── server.js          # Express 路由、权限与版本号校验、发布事务
│   ├── db.js              # SQLite 连接与建表
│   ├── initdb.js          # 幂等初始化：账号 + 示例草稿
│   ├── auth.js            # scrypt 密码哈希
│   └── scheduling.js      # 校验/拓扑环检测/现场重算核心算法
└── public
    ├── index.html
    ├── styles.css
    └── app.js             # 登录、草稿、现场、版本回看三个页面
```

## 七、主要 HTTP API

| 方法 | 路径 | 权限 | 说明 |
| --- | --- | --- | --- |
| POST | `/api/login` `/api/logout` | 公开 | 登录 / 退出 |
| GET | `/api/me` | 公开 | 当前会话与岗位列表 |
| GET | `/api/draft` | 舞监 | 草稿（含实时校验结果） |
| POST/PUT/DELETE | `/api/draft/cues[/:id]` | 舞监 | 提示项增改删（校验草稿版本号/行版本号） |
| POST/DELETE | `/api/draft/deps[/:id]` | 舞监 | 前置关系增删（校验草稿版本号） |
| GET | `/api/publish/handover/preview` | 舞监 | **发布前交接预览**：旧执行版已完成项的交接资格、不合格原因、旧登记行版本；无旧版时列表为空 |
| POST | `/api/publish` | 舞监 | 发布只读版本；冲突返回 422 并列出冲突项；可带 `handover` 交接选择，资格/版本失效返回 409 `HANDOVER_STALE` |
| GET | `/api/live` | 已登录 | 当前最新版本的现场重算结果（含每行更正历史 `corrections`、跳过留痕 `skip`、本版代班记录 `deputies`；执行账号另含有效岗位 `myPositions`；被跳过项 `status` 为 `SKIPPED`） |
| GET | `/api/live/deputies` | 已登录 | 当前执行版代班记录（含已撤销历史）；舞监另返回可选岗位 `positions` 与执行账号 `operators` |
| POST | `/api/live/deputies` | 舞监 | 启用岗位代班（仅当前执行版）；重复启用 409 `DEPUTY_EXISTS` |
| POST | `/api/live/deputies/revoke` | 舞监 | 撤销岗位在任代班；无在任代班 409 `NO_ACTIVE_DEPUTY` |
| POST | `/api/registrations` | 执行岗位 | start / complete / estimate（同一事务内复核当前版、代班授权、行版本号）；已跳过项返回 409 `CUE_SKIPPED` |
| POST | `/api/registrations/corrections` | 舞监 | 更正/清除单个登记字段（校验行版本号、仅当前执行版本；级联清除与一致性校验；原子写入更正历史）；已跳过项返回 409 `CUE_SKIPPED` |
| POST | `/api/skips/preview` | 舞监 | **跳过影响预览**（仅当前执行版）：沿前置关系计算提示项及全部后继，返回将新跳过/已跳过/阻断项与各登记行版本 |
| POST | `/api/skips` | 舞监 | **确认整链跳过**（非空原因 + 预览所见链与行版本）：阻断或行版本变化 409 `SKIP_BLOCKED` 整次拒绝；清除预计结束、保留计划、写入操作者/时间/原因；已跳过项不重复留痕 |
| GET | `/api/versions` `/api/versions/:id` | 已登录 | 版本列表与历史回看（回看含每行更正历史） |

## 八、登记交接 API 调用示例

以本地默认端口 3000、舞监账号为例（`curl -c/-b` 复用会话 Cookie）：

### 1. 登录并预览交接资格

```bash
# 登录（保存会话 Cookie）
curl -c sm.cookie -s -X POST http://localhost:3000/api/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"sm","password":"sm12345"}'

# 发布前预览：返回草稿版本、旧执行版、可选 / 不可选清单
curl -b sm.cookie -s http://localhost:3000/api/publish/handover/preview
```

响应示例（旧版中 L1、S1 已完成且与草稿完全一致；S2 已完成但草稿计划开始被改过）：

```json
{
  "draftVersion": 1,
  "oldVersion": { "id": 1, "versionNo": 1, "note": "首版", "createdAt": "2026-10-03 19:40:00" },
  "eligible": [
    { "cueRef": 1, "code": "L1", "title": "开场灯光就位", "position": "灯光",
      "actualStartText": "19:00", "actualEndText": "19:06", "rowVersion": 3, "selected": true },
    { "cueRef": 2, "code": "S1", "title": "开场音乐", "position": "音响",
      "actualStartText": "19:05", "actualEndText": "19:08", "rowVersion": 3, "selected": true }
  ],
  "ineligible": [
    { "cueRef": 5, "code": "S2", "title": "风声效果", "position": "音响",
      "actualStartText": "19:12", "actualEndText": "19:17", "rowVersion": 3,
      "reasons": ["计划开始不一致（旧版 19:12 → 草稿 19:13）"],
      "diffs": [{ "field": "plannedStart", "label": "计划开始", "oldText": "19:12", "newText": "19:13",
                  "reason": "计划开始不一致：旧版为 19:12，草稿为 19:13" }] }
  ]
}
```

- 首次发布（无旧版）时 `oldVersion` 为 `null`，`eligible` / `ineligible` 均为 `[]`；
- 仅「旧登记同时有实际开始和实际完成」的项会出现（只开始未完成的项不在列表中）。

### 2. 携带交接选择发布

```bash
curl -b sm.cookie -s -X POST http://localhost:3000/api/publish \
  -H 'Content-Type: application/json' -d '{
    "version": 1,
    "note": "第二场联排",
    "handover": {
      "oldVersionId": 1,
      "draftVersion": 1,
      "items": [
        { "cueRef": 1, "rowVersion": 3 },
        { "cueRef": 2, "rowVersion": 3 }
      ]
    }
  }'
```

- `version`：沿用现有发布请求的**草稿版本号**校验（不一致返回 409 `VERSION_CONFLICT`）；
- `handover.oldVersionId` / `draftVersion` / `items[].rowVersion` 均取自上一步预览；
- `items` 只放要延续的项；为空数组 `[]` 表示一项都不延续（仍会核对预览基于的草稿版本与旧版）；
- **不传 `handover` 字段时完全维持原有发布行为**（新版全部空登记）。

成功响应沿用原发布返回（新版本完整负载），并追加 `handover` 汇总；每个被延续的提示项负载中带有 `handover` 来源标注：

```json
{
  "id": 2, "versionNo": 2, "note": "第二场联排",
  "cues": [
    { "code": "L1", "status": "DONE", "actualStartText": "19:00", "actualEndText": "19:06",
      "estimatedEndText": null,
      "handover": { "sourceVersionId": 1, "sourceVersionNo": 1, "sourceCueRef": 1, "sourceCode": "L1" } }
  ],
  "handover": { "carriedCount": 2, "sourceVersionId": 1, "sourceVersionNo": 1 }
}
```

### 3. 资格 / 版本失效被整体拒绝

预览之后若草稿又被编辑、旧版被新版替代、或岗位/舞监对所选登记行又有提交（行版本推进），发布整体拒绝（不产生任何新版本、不写部分登记）：

```json
HTTP 409
{
  "error": "HANDOVER_STALE",
  "message": "交接资格核对未通过（1 项），本次发布已整体拒绝，请重新预览后再选择发布",
  "items": [
    { "cueRef": 1, "code": "L1",
      "reason": "「L1」旧登记行版本已变化（预览时 v3，当前 v4，可能被岗位登记或舞监更正更新），请重新预览后再发布",
      "currentRowVersion": 4 }
  ]
}
```

其余既有行为不变：计划冲突仍返回 **422 `PUBLISH_CONFLICT`** 并列出冲突项；草稿版本号过期返回 **409 `VERSION_CONFLICT`**。页面上的勾选与上述 API 完全等价：草稿编辑页发布面板内完成「预览 → 勾选 → 发布」，现场页与版本回看页均可看到延续项的来源标注，旧版登记与更正历史保持可回看。

## 九、岗位代班 API 调用示例

以本地默认端口 3000 为例（`curl -c/-b` 复用会话 Cookie），假设已发布第 1 版为当前执行版。

### 1. 舞监查看当前执行版代班管理视图

```bash
curl -c sm.cookie -s -X POST http://localhost:3000/api/login \
  -H 'Content-Type: application/json' -d '{"username":"sm","password":"sm12345"}'

curl -b sm.cookie -s http://localhost:3000/api/live/deputies
```

响应（舞监含可选岗位与执行账号；`deputies` 含已撤销历史，最新在前）：

```json
{
  "versionId": 1,
  "versionNo": 1,
  "deputies": [],
  "positions": ["灯光", "道具", "音响"],
  "operators": [
    { "username": "light", "displayName": "灯光执行", "position": "灯光" },
    { "username": "sound", "displayName": "音响执行", "position": "音响" }
  ]
}
```

### 2. 启用代班（灯光 → 音响执行）

```bash
curl -b sm.cookie -s -X POST http://localhost:3000/api/live/deputies \
  -H 'Content-Type: application/json' \
  -d '{"position":"灯光","username":"sound","versionId":1}'
```

- `versionId` 可选但建议携带：若页面基于的版本已不是当前执行版，返回 409 `VERSION_STALE`，避免误操作新版本；
- 成功返回 201 与最新代班管理视图，记录含 `enabledBy` / `enabledAt`（操作者与时间）。

### 3. 代班期间的登记行为

```bash
# 原岗位账号 light 登记灯光提示项 → 403，登记权已移交
{"error":"DEPUTY_ACTIVE","message":"岗位「灯光」当前由 音响执行（sound）代班，你的该岗位登记权已暂时移交，请刷新页面"}

# 代班账号 sound 可登记灯光提示项，也可继续登记自己音响岗位（请求格式不变）
curl -b sound.cookie -s -X POST http://localhost:3000/api/registrations \
  -H 'Content-Type: application/json' \
  -d '{"versionId":1,"cueRef":1,"rowVersion":1,"action":"start","time":"19:00"}'
```

执行账号调用 `GET /api/live` 时，响应 `version.myPositions` 给出当前有效岗位（如 sound 为 `["音响","灯光"]`，light 为 `[]`），现场页面据此显示可操作项；`version.deputies` 列出本版代班记录。

### 4. 撤销代班

```bash
curl -b sm.cookie -s -X POST http://localhost:3000/api/live/deputies/revoke \
  -H 'Content-Type: application/json' -d '{"position":"灯光","versionId":1}'
```

- 撤销后原岗位账号立即恢复登记权；代班账号持旧页面提交返回 403 且**不改动任何登记**；
- 记录保留 `revokedBy` / `revokedAt`，与启用记录一起在回看页只读可查；重复撤销返回 409 `NO_ACTIVE_DEPUTY`。

### 5. 错误一览（均整体拒绝，不留部分记录）

| 场景 | 状态码 | error |
| --- | --- | --- |
| 账号不存在 | 404 | `UNKNOWN_ACCOUNT` |
| 账号不是执行账号 | 400 | `NOT_OPERATOR` |
| 当前执行版没有该岗位 | 400 | `POSITION_NOT_IN_VERSION` |
| 该岗位已有在任代班 | 409 | `DEPUTY_EXISTS` |
| 撤销时没有任代班 | 409 | `NO_ACTIVE_DEPUTY` |
| 尚未发布任何版本 | 409 | `NO_LIVE_VERSION` |
| 页面版本已过期 | 409 | `VERSION_STALE` |
| 原岗位账号在代班期间登记 | 403 | `DEPUTY_ACTIVE` |

发布新版后代班不自动延续（新版 `deputies` 为空）；旧版记录通过 `GET /api/versions/:id` 的 `deputies` 字段只读可查。

## 十、跳过依赖链 API 调用示例（本轮新增）

以本地默认端口 3000、舞监账号为例，假设已发布第 1 版为当前执行版（示例计划：`L1 → S1 → L2 → L3`，`S1 → S2`，`L1 → P1`，提示项引用 `cueRef` 即发布时的草稿 id）。

### 1. 登录并预览跳过影响

```bash
curl -c sm.cookie -s -X POST http://localhost:3000/api/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"sm","password":"sm12345"}'

# 预览：以 S1（cueRef=2）为起点，沿前置关系可达的全部后继
curl -b sm.cookie -s -X POST http://localhost:3000/api/skips/preview \
  -H 'Content-Type: application/json' \
  -d '{"versionId":1,"cueRef":2}'
```

响应（此时链上均无实际开始/完成；多分支合并链不重复，已跳过项以 `alreadySkipped` 标出）：

```json
{
  "versionId": 1,
  "versionNo": 1,
  "rootCueRef": 2,
  "rootCode": "S1",
  "totalCount": 4,
  "newSkipCount": 4,
  "alreadySkippedCount": 0,
  "blocked": false,
  "blockers": [],
  "chain": [
    { "cueRef": 2, "code": "S1", "title": "开场音乐", "position": "音响",
      "plannedStartText": "19:05", "plannedEndText": "19:08",
      "actualStartText": null, "actualEndText": null, "estimatedEndText": null,
      "rowVersion": 1, "isRoot": true, "alreadySkipped": false, "skip": null },
    { "cueRef": 4, "code": "L2", "rowVersion": 1, "isRoot": false, "alreadySkipped": false, "...": "..." },
    { "cueRef": 5, "code": "S2", "rowVersion": 1, "isRoot": false, "alreadySkipped": false, "...": "..." },
    { "cueRef": 6, "code": "L3", "rowVersion": 1, "isRoot": false, "alreadySkipped": false, "...": "..." }
  ]
}
```

链中任一项已有实际开始/完成时 `blocked` 为 `true`，`blockers` 逐项列出，例如对已完成的 L1 发起诉览：

```json
{
  "blocked": true,
  "blockers": [
    { "cueRef": 1, "code": "L1",
      "reason": "「L1」已有实际开始与完成（开始 19:00，完成 19:06），不能跳过" }
  ],
  "chain": [ { "cueRef": 1, "code": "L1", "isRoot": true, "...": "..." } ]
}
```

### 2. 确认整链跳过

携带**非空原因**与预览所见链中**未跳过项**的 `cueRef / rowVersion`（已跳过项可不传；服务端会复核链构成必须与当前依赖闭包一致）：

```bash
curl -b sm.cookie -s -X POST http://localhost:3000/api/skips \
  -H 'Content-Type: application/json' -d '{
    "versionId": 1,
    "cueRef": 2,
    "reason": "音响设备故障，相关链取消",
    "items": [
      { "cueRef": 2, "rowVersion": 1 },
      { "cueRef": 4, "rowVersion": 1 },
      { "cueRef": 5, "rowVersion": 1 },
      { "cueRef": 6, "rowVersion": 1 }
    ]
  }'
```

成功返回 **201**，响应体为最新现场负载 `{ version, skip }`；被跳过项 `status` 为 `SKIPPED`，且带有留痕：

```json
{
  "skip": {
    "rootCueRef": 2, "rootCode": "S1", "skippedCount": 4,
    "reason": "音响设备故障，相关链取消",
    "skippedBy": "舞监", "skippedAt": "2026-10-04 19:30:12"
  },
  "version": {
    "id": 1, "versionNo": 1, "isLatest": true,
    "cues": [
      { "code": "S1", "status": "SKIPPED",
        "plannedStartText": "19:05", "schedStartText": "19:05",
        "actualStartText": null, "actualEndText": null, "estimatedEndText": null,
        "skip": { "reason": "音响设备故障，相关链取消", "skippedBy": "舞监", "skippedAt": "2026-10-04 19:30:12" } }
    ],
    "conflicts": []
  }
}
```

- 首次跳过清除各项**预计结束**（S2 若登记过预计结束会被置空，其登记行 `row_ver` 相应 +1），**已发布计划不变**；
- 跳过项不约束后继、不参与岗位冲突；其余项按现有前置与登记规则重算。

### 3. 阻断 / 行版本变化：整次拒绝（HTTP 409）

预览之后若岗位/舞监更新了链上登记（行版本推进），或链上某项已有实际开始/完成：

```json
HTTP 409
{
  "error": "SKIP_BLOCKED",
  "message": "依赖链存在 1 个阻断项（已有实际开始/完成或登记行版本变化），本次跳过已整体拒绝，未标记任何项",
  "items": [
    { "cueRef": 2, "code": "S1",
      "reason": "「S1」登记行版本已变化（预览时 v1，当前 v2，可能刚被岗位登记或舞监更正更新），请重新预览后再确认",
      "currentRowVersion": 2 }
  ]
}
```

### 4. 跳过之后的行为

```bash
# 执行账号再登记被跳过项
curl -b sound.cookie -s -X POST http://localhost:3000/api/registrations \
  -H 'Content-Type: application/json' \
  -d '{"versionId":1,"cueRef":2,"rowVersion":1,"action":"start","time":"19:10"}'
# HTTP 409
{"error":"CUE_SKIPPED","message":"「S1」已被舞监标记跳过，执行账号不得再登记实际开始 / 完成 / 预计结束，请刷新页面"}

# 舞监更正跳过项时间
curl -b sm.cookie -s -X POST http://localhost:3000/api/registrations/corrections \
  -H 'Content-Type: application/json' \
  -d '{"versionId":1,"cueRef":2,"rowVersion":1,"field":"estimated_end","estimatedEndText":"19:30","reason":"x"}'
# HTTP 409 {"error":"CUE_SKIPPED","message":"「S1」已标记跳过，跳过时间不可更正，请刷新页面"}
```

### 5. 错误一览（均整体拒绝，不留部分标记）

| 场景 | 状态码 | error |
| --- | --- | --- |
| 未填写 / 空白跳过原因 | 400 | `BAD_INPUT` |
| 未传预览链 `items` 或格式错误 | 400 | `BAD_INPUT` |
| 提示项不在该版本 | 404 | `NOT_FOUND` |
| 提交链与当前后继闭包不一致（漏项 / 多项） | 409 | `SKIP_STALE` |
| 链上已有实际开始/完成，或登记行版本变化 | 409 | `SKIP_BLOCKED`（响应 `items` 列阻断项） |
| 整链均已跳过，重复确认 | 409 | `SKIP_NOOP`（不新增留痕） |
| 跳过项被岗位登记或被舞监更正 | 409 | `CUE_SKIPPED` |
| 页面基于的版本已归档 | 409 | `VERSION_STALE`（跳过预览/确认）/ `VERSION_ARCHIVED`（登记/更正） |
| 尚未发布任何版本 | 409 | `NO_LIVE_VERSION` |
| 执行账号调用跳过接口 | 403 | `FORBIDDEN` |

已跳过项在 `GET /api/live` 与 `GET /api/versions/:id` 中以 `status: "SKIPPED"` 与每行 `skip: { reason, skippedBy, skippedAt }` 只读展示；**新发布版本不带任何跳过状态**，旧版回看始终可见当时的跳过原因；已完成项向新版的交接资格仍沿用第四节的既有交接约定（跳过项没有实际开始/完成，本身不具备交接资格）。

