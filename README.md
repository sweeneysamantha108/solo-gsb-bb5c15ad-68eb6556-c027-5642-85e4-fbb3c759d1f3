# 水质实验室 · 样本分装与混样溯源工作台

基于 **Node.js + Express + SQLite（better-sqlite3）** 的 Web 工作台，用于管理水质样本的
原瓶登记、分装 / 混样操作、**批次预演与两阶段提交**、余量守恒校验、完整来源谱系追溯、
历史操作更正与**更正预演 / 确认两阶段**，以及**质检原瓶隔离与受影响瓶范围判定 / 拦截**。

## 一键启动（Docker Compose）

```bash
docker compose up --build
```

容器启动时会自动安装依赖、建表；数据库为空时写入示例数据（3 个原始瓶 + 2 条操作）。

浏览器访问：**http://localhost:3000**

停止并删除数据：`docker compose down -v`（数据保存在 named volume `lab-data` 中，普通 `down` 不丢数据）。

## 配置项（环境变量）

| 变量 | 默认值 | 说明 |
|---|---|---|
| `PORT` | `3000` | HTTP 监听端口（Compose 已映射 3000:3000） |
| `DB_PATH` | 容器内 `/app/data/lab.db`，本地 `./data/lab.db` | SQLite 数据库文件路径，启动时自动建目录、建表、初始化 |

## 本地运行（不用 Docker）

```bash
npm install
npm start          # 访问 http://localhost:3000
```

要求 Node.js ≥ 18。better-sqlite3 为原生模块，标准 node 镜像 / 常见桌面环境均可直接安装。

## 页面功能

- **瓶子总览**：每个瓶的类型（原始瓶 / 衍生瓶）、创建或采样时间、初始体积、当前余量；
  点击任意行查看该瓶的**完整来源谱系**（向上递归至原始瓶）、**各原始瓶的精确贡献量（分数 mL）**
  及相关操作记录。
- **登记原始瓶**：录入瓶编号、采样时间、初始体积（mL，正整数）。
- **新建分装 / 混样**：录入唯一操作号、目标瓶编号（须为新编号）、操作时间、
  一个或多个来源瓶及各自取用体积；目标瓶体积 = 取用量之和。
- **批量预演 / 提交**：把连续分装 / 混样作为一个批次处理。录入批次号与按录入顺序排列的
  多条操作（每条沿用操作号、目标瓶、操作时间、来源瓶与取用毫升数），先**预演**查看逐条可行性、
  受影响瓶预计余量与提交凭据（不写入数据），再凭凭据**提交**整批；批内可引用前项生成的瓶。
- **操作记录与更正**：列出全部操作（含所属批次号）；点“更正”可修改该操作的来源与取用量。
  支持先**预演更正**（不写库，展示因重放而变化的瓶余量、目标瓶及下游瓶更正前后的原瓶精确贡献）
  再**确认更正**（凭据原子应用，结果与预演一致），也可沿用**立即更正**直接提交；
  提交后按操作顺序重算相关瓶余量。
- **质检隔离**：对已登记原始瓶设置隔离（原因必填）/ 解除；展示全部隔离原瓶与受影响瓶
  （每瓶命中的隔离原瓶及原因），瓶总览以红底 / 标签标出“已隔离 / 受隔离影响”，瓶详情给出原因块。
  受影响瓶在新建操作与批次预演 / 提交中均被拦截（批内前项受影响时后项引用同拦截，整批不落库）；
  多原瓶隔离时解除其一会提示仍受影响的瓶；解除后凭据需重新预演。

## 批次处理（本轮新增）

实验室把一批连续分装 / 混样作为一个批次，分两步完成：

1. **预演** `POST /api/batches/preview`：只校验与推演，**不写入任何数据**。返回逐条可行性、
   受影响瓶的预计余量，以及一枚**提交凭据**（绑定本批计划内容与当前台账版本）。
2. **提交** `POST /api/batches/commit`：在**单个事务**中核对凭据、重跑全部校验并落库，
   随后重放全量台账确认任一时刻无超用；任一环节失败**整批回滚，不落库**。

核算规则与单条操作一致：按（操作时间, 入库顺序）重放；**同一操作时间内，批内按录入顺序**；
来源瓶可引用**批内录入顺序在前**的条目生成的瓶；操作时间不得早于来源瓶创建时间；
任一时刻任何瓶不得超用。

- **幂等**：已成功提交的批次，相同批次号 + 相同内容重复提交 → HTTP 200 幂等返回，不重复扣量；
  相同批次号 + 不同内容 → 409 `BATCH_CONFLICT`。
- **凭据失效**：预演之后若台账发生任何变化（新登记原瓶 / 单条操作 / 更正 / 其它批次提交），
  或提交内容与预演计划不一致，提交将被拒绝（409 `STALE_TOKEN` / `PLAN_CHANGED`），需**重新预演**。
- **逐条报错**：编号冲突（批内或与现有台账）、重复来源、非法体积、缺失来源、超用等，
  均在响应中指出具体条目序号与操作号。

### 调用示例

```bash
# 1) 预演（不写库）
curl -s http://localhost:3000/api/batches/preview -H 'Content-Type: application/json' -d '{
  "batch_id": "BATCH-0001",
  "items": [
    {"op_id":"OP-0101","target_bottle":"MIX-101","op_time":"2026-10-04T09:00",
     "sources":[{"bottle":"W2026-001","volume":200},{"bottle":"W2026-002","volume":100}]},
    {"op_id":"OP-0102","target_bottle":"ALQ-101","op_time":"2026-10-04T10:00",
     "sources":[{"bottle":"MIX-101","volume":150}]}
  ]}'
# → {"ok":true,"items":[{"index":0,"op_id":"OP-0101","ok":true,"errors":[]},...],
#    "projected_balances":[{"bottle":"MIX-101","in_batch":true,"remaining":150},...],
#    "commit_token":"v1.3.<plan_hash>.<hmac>", ...}

# 2) 提交（携带预演返回的 commit_token，单事务整批落库）
curl -s http://localhost:3000/api/batches/commit -H 'Content-Type: application/json' -d '{
  "batch_id": "BATCH-0001",
  "items": [ …与预演完全相同的条目… ],
  "commit_token": "v1.3.<plan_hash>.<hmac>"
}'
# → 201 {"ok":true,"committed":2,"projected_balances":[...]}
#    重复提交相同内容 → 200 {"ok":true,"idempotent":true,...}

# 3) 查询批次
curl -s http://localhost:3000/api/batches            # 全部已提交批次
curl -s http://localhost:3000/api/batches/BATCH-0001 # 批次详情（按录入顺序的条目）
```

提交成功后，批内操作立即进入现有的余量（`GET /api/bottles`）、操作记录
（`GET /api/operations`，含 `batch_id` 字段）与谱系查询（`GET /api/bottles/:id`）。

## 原瓶贡献量查询（本轮新增）

按瓶编号查看**衍生瓶由各原始瓶贡献的精确体积**（工作台瓶详情同步展示）：

```bash
curl -s http://localhost:3000/api/bottles/ALQ-001/contributions
# 瓶详情接口 GET /api/bottles/:id 的响应也追加了同结构的 contributions 字段
```

```jsonc
{
  "bottle": "ALQ-001",
  "kind": "derived",
  "created_by_op": "OP-0002",
  "formation_volume": 100,                 // 目标瓶形成时的总体积（mL，整数）
  "contributions": [                       // 按原瓶编号排序；同一原瓶多路径汇入已合并
    {"bottle": "W2026-001", "contribution_ml": {"numerator": "400", "denominator": "7",
                                                "text": "400/7", "approx_ml": 57.142857142857146}},
    {"bottle": "W2026-002", "contribution_ml": {"numerator": "300", "denominator": "7",
                                                "text": "300/7", "approx_ml": 42.857142857142854}}
  ],
  "sum_ml": {"numerator": "100", "denominator": "1", "text": "100", "approx_ml": 100}
}
```

- **分摊规则**：原瓶贡献 = 自身初始体积；分装 / 混样从来源瓶 S 取 v mL 时，视 S 内容均匀，
  S 中各原瓶贡献按 **v / V_S**（V_S 为 S 形成时的总体积）分摊；同一原瓶经不同路径汇入须合并。
- **精确分数**：全程 BigInt 有理数运算并在每一步约分，`numerator / denominator` 为约分后的
  精确值（字符串承载任意精度），`approx_ml` 仅供展示；**不做逐层四舍五入**，
  `sum_ml` 严格等于 `formation_volume`。
- **与余量无关**：计算只依赖瓶子形成图（原瓶初始体积 + 各形成操作的来源与取用量），
  来源瓶之后的剩余量变化不改变已形成瓶子的配比。
- **更正即重算**：历史来源或取用量更正成功后，查询依据当前已提交台账重建形成图，
  所有下游贡献自动重算；**被现有守恒规则拒绝的更正已回滚，形成图不变，查询结果不变**。
- 原始瓶 / 衍生瓶均可查询（原瓶返回自身初始体积一条贡献）；瓶不存在 →
  404 `BOTTLE_NOT_FOUND`。

### 批次预演中的预计贡献

`POST /api/batches/preview` 响应新增 **`projected_contributions`** 字段：在逐条可行性之外，
为每个**可行条目**给出其目标瓶的预计原瓶贡献（结构同上，含 `index / op_id / target_bottle`），
批内引用前项生成的瓶同样适用（把本批条目作为虚拟形成图叠加到当前台账）。
预演仍不写库；提交后用 `GET /api/bottles/:id/contributions` 查到的结果与预演逐项一致。
引用了谱系不完整前项（如前项因来源不存在而不可行）的条目跳过预计贡献，不影响其余条目；
台账变化导致凭据失效仍按既有 `STALE_TOKEN` / `PLAN_CHANGED` 规则处理。

## 更正预演与确认（本轮新增）

更正历史分装 / 混样前，操作员可提交操作号及拟替换的来源瓶、取用毫升数做**预演**，
确认影响无误后再**确认**，两阶段约定与批次预演 / 提交一致：

1. **预演** `POST /api/operations/:op_id/corrections/preview`（**不写库**），返回：
   - `balance_changes`：因重放而变化的瓶余量（更正前 → 更正后；无变化的瓶不列出）；
   - `contribution_changes`：目标瓶及其下游瓶**更正前后**的原瓶精确贡献
     （沿用约分分数 mL 格式；无变化的瓶不列出）；
   - `confirm_token`：确认凭据（绑定更正计划内容与当前台账版本，域前缀 `c1`）。
   操作号 / 来源不存在、来源重复、体积无效、操作时间早于来源创建、或重放导致任一操作超用时，
   在 `errors` / `conflicts` 中返回具体原因及冲突操作、瓶与取用量，**不签发确认凭据**。
2. **确认** `POST /api/operations/:op_id/corrections/confirm`：携带与预演完全相同的来源及
   `confirm_token`，在**单个事务**中核对凭据、重跑校验并落库，应用结果与预演一致。
   - 确认内容变动 → 409 `PLAN_CHANGED`；预演后有登记 / 操作 / 更正 / 批次提交 → 409 `STALE_TOKEN`，
     均需**重新预演**；
   - 已确认的同一凭据重试 → 200 **幂等返回首次结果**，不重复更正。

现有 `POST /api/operations/:op_id/correct`（立即更正）与批次提交约定继续可用；
工作台“操作记录与更正”页已支持 预演 → 确认 流程并展示差异。

### 隔离与更正的关系

历史记录在隔离期间**始终可查询**；更正仍**只按既有守恒规则**执行（不因来源涉及隔离原瓶而额外禁止）：
更正**成功**后系统按新的来源关系**自动重算隔离影响范围**（影响可能因此扩大或缩小，见下节示例）；
更正**失败**（会导致后续超用）时整次回滚、形成图不变，隔离状态与影响范围也维持原状。
隔离状态变动（设置 / 更新原因 / 解除）会递增台账版本，**尚未使用**的更正确认凭据随之失效
（确认时 409 `STALE_TOKEN`，需重新预演）；已确认凭据的重试仍幂等返回首次结果。

## 质检隔离与影响范围（本轮新增）

质检人员可对**已登记原始瓶**设置隔离状态并填写**非空原因**，或解除隔离；系统按
**当前已提交来源关系**计算影响范围并在工作台 / HTTP API 中展示。

**影响判定规则**

- 仅原始瓶可被隔离（对衍生瓶设置返回 400 `NOT_ORIGINAL`）；原因必填（空 / 纯空白 → 400 `BAD_REASON`）。
- **只要衍生瓶的来源链（任意路径）含隔离原瓶，该衍生瓶即受影响**；每个受影响瓶返回其命中的
  **全部**隔离原瓶及原因（多路径汇入合并）。
- **多个原瓶同时隔离时，解除其中一个不能放行仍受其他原瓶影响的瓶**：解除响应的 `still_affected`
  会列出仍被其他隔离原瓶影响的瓶；直到来源链不再含任何隔离原瓶，瓶才放行。
- **受影响瓶（含隔离原瓶自身）不得用于新分装 / 混样**：
  - 单条提交 `POST /api/operations` → 409 `QUARANTINE_BLOCKED`，`details` 指出每个被拦截来源瓶命中的隔离原瓶；
  - 批次预演逐条报告 `QUARANTINE_BLOCKED`（含命中原瓶与原因），**整批不通过、不发提交凭据**；
    批次提交在事务内重跑同一校验，任一拦截则**整批不落库**；
  - **批内前项若受影响，引用它的后项同样被拦截**（批次图 = 当前台账 + 本批条目叠加判定）。
- 历史操作、谱系、贡献量等查询不受隔离影响，始终可用。
- **凭据失效**：设置 / 更新隔离 / 解除隔离均递增台账版本，预演后未使用的批次提交凭据与更正确认
  凭据自动失效（409 `STALE_TOKEN`，需重新预演）；**已成功提交的相同请求重试仍沿用原幂等结果**
  （批次 / 操作 / 更正的幂等判定均在凭据校验之前）。同原因重复设置隔离为幂等操作，状态与版本不变。

**接口**

| 方法 | 路径 | 说明 |
|---|---|---|
| `PUT` | `/api/bottles/:id/quarantine` | 设置 / 更新原瓶隔离 `{reason（必填）, operator?}`；首次或原因变更 → 201 并返回重算后的 `affected`；同原因重复 → 200 幂等 |
| `DELETE` | `/api/bottles/:id/quarantine` | 解除隔离 → 200（含 `still_affected`）；未隔离 → 409 `NOT_QUARANTINED` |
| `GET` | `/api/quarantine` | 全部隔离原瓶 + 全部受影响瓶（每瓶 `hits`：隔离原瓶、原因、时刻、操作人） |
| `GET` | `/api/quarantine?bottle=xxx` | 单瓶命中查询（瓶是否隔离 + `hits`）；瓶不存在 → 404 |

`GET /api/bottles` 每行新增 `quarantined`（自身是否隔离）与 `quarantine_hits`（命中的隔离原瓶）；
`GET /api/bottles/:id` 新增 `quarantine`（隔离原瓶的原因块）与 `quarantine_hits`。

### 调用示例

```bash
# 1) 隔离原始瓶（原因非空；返回受影响瓶及每瓶命中的隔离原瓶）
curl -s -X PUT http://localhost:3000/api/bottles/W2026-001/quarantine \
  -H 'Content-Type: application/json' -d '{"reason":"浊度异常，待复检","operator":"QC-张"}'
# → 201 {"ok":true,"bottle":"W2026-001","reason":"浊度异常，待复检",
#    "affected":[{"bottle":"ALQ-001","hits":[{"bottle":"W2026-001","reason":"浊度异常，待复检",...}]},
#                {"bottle":"MIX-001", ...}, {"bottle":"W2026-001", ...}], ...}
# 同原因重复 PUT → 200 {"ok":true,"idempotent":true,...}（状态 / 台账版本不变）

# 2) 查询隔离状态与影响范围（工作台“质检隔离”页同数据）
curl -s http://localhost:3000/api/quarantine
# → {"ledger_version":N,"quarantined":[{"bottle":"W2026-001","reason":"...","quarantined_at":"...","operator":"QC-张"}],
#    "affected":[{"bottle":"ALQ-001","hits":[{"bottle":"W2026-001","reason":"...",...}]}, ...]}
curl -s 'http://localhost:3000/api/quarantine?bottle=ALQ-001'   # 单瓶命中

# 3) 受影响瓶用于新分装 / 混样被拦截（单条 409；批次预演逐条报 QUARANTINE_BLOCKED，整批不落库）
curl -s http://localhost:3000/api/operations -H 'Content-Type: application/json' -d '{
  "op_id":"OP-2001","target_bottle":"MIX-201","op_time":"2026-10-05T09:00",
  "sources":[{"bottle":"ALQ-001","volume":10}]}'
# → 409 {"error":{"code":"QUARANTINE_BLOCKED",
#    "message":"来源瓶处于质检隔离 / 受隔离影响，操作已拒绝，数据未变更（命中隔离原瓶：W2026-001）",
#    "details":[{"bottle":"ALQ-001","hits":[{"bottle":"W2026-001","reason":"浊度异常，待复检",...}]}]}}

# 4) 再隔离第二个原瓶：受影响瓶的 hits 同时列出两个原瓶
curl -s -X PUT http://localhost:3000/api/bottles/W2026-002/quarantine \
  -H 'Content-Type: application/json' -d '{"reason":"余氯异常"}'
# MIX-001 / ALQ-001 的 hits = [W2026-001, W2026-002]

# 5) 只解除一个 → 200，still_affected 指出仍被另一原瓶影响、继续被拦截的瓶
curl -s -X DELETE http://localhost:3000/api/bottles/W2026-002/quarantine
# → {"ok":true,"still_affected":[{"bottle":"MIX-001","hits":[{"bottle":"W2026-001",...}]},
#                                {"bottle":"ALQ-001", ...}], ...}

# 6) 解除最后一个隔离原瓶后放行（凭据曾因隔离变动失效的，需重新预演后再提交）
curl -s -X DELETE http://localhost:3000/api/bottles/W2026-001/quarantine
```

更正成功后影响范围自动重算的示例：若 `MIX-001` 原由 `W2026-001 + W2026-002` 混合而成（两者中
`W2026-002` 被隔离），把创建操作更正为仅取 `W2026-001` 并确认成功后，`MIX-001` 及其下游瓶的
来源链不再含 `W2026-002`，下一次 `GET /api/quarantine` 的影响范围即不再包含它们；若更正被守恒
规则拒绝，则隔离状态与影响范围维持原状。

### 调用示例

```bash
# 1) 预演（不写库）：把 OP-0001 的来源改为 W2026-001×250 + W2026-002×100
curl -s http://localhost:3000/api/operations/OP-0001/corrections/preview \
  -H 'Content-Type: application/json' -d '{
    "sources":[{"bottle":"W2026-001","volume":250},{"bottle":"W2026-002","volume":100}]}'
# → {"ok":true,"ledger_version":1,
#    "balance_changes":[{"bottle":"W2026-001","before":800,"after":750},
#                       {"bottle":"W2026-002","before":650,"after":700}],
#    "contribution_changes":[{"bottle":"MIX-001","before":{...},"after":{...}},
#                            {"bottle":"ALQ-001","before":{...},"after":{...}}],
#    "confirm_token":"c1.1.<plan_hash>.<hmac>", ...}

# 2) 确认（携带与预演完全相同的 sources 与 confirm_token，单事务原子应用）
curl -s http://localhost:3000/api/operations/OP-0001/corrections/confirm \
  -H 'Content-Type: application/json' -d '{
    "sources":[{"bottle":"W2026-001","volume":250},{"bottle":"W2026-002","volume":100}],
    "confirm_token":"c1.1.<plan_hash>.<hmac>"}'
# → 201 {"ok":true,"idempotent":false,"balance_changes":[...],"contribution_changes":[...], ...}
#    结果与预演一致；同一凭据重试 → 200 {"ok":true,"idempotent":true,...}（首次结果，不重复更正）
```

## 业务规则与一致性设计

- **唯一操作号 + 幂等**：`op_id` 为主键。同一操作号重复提交**相同内容**时幂等返回
  （HTTP 200，不重复扣量）；内容不一致返回 409 冲突。
- **来源约束**：同一操作内来源瓶不得重复；取用体积必须为正整数；来源瓶必须已存在；
  操作时间不得早于任一来源瓶的创建（采样）时间。
- **任一时刻不超用**：每次写入（新操作 / 更正）后，在事务内按
  （操作时间, 入库顺序）重放全量台账，任一瓶余量为负即整体回滚，
  返回 409 及冲突明细（操作号、瓶、需取量、可用量）。
- **历史更正**：更正取用量后按操作顺序重算所有受影响瓶；若导致后续操作超用，
  **整次更正拒绝并回滚，原数据保持不变**，响应中指出冲突记录。
- **更正两阶段**：更正预演在内存中把“替换来源后”的台账重放一遍并重建形成图，
  给出余量变化与目标瓶及下游瓶的原瓶贡献前后对照（不写库）；确认凭据 =
  HMAC(更正计划哈希 + 台账版本)（域前缀 `c1`，与批次凭据 `v1` 隔离）。确认在单事务中完成
  凭据核对 → 业务校验 → 落库 → 全账重放，结果与预演一致；已确认凭据（落库于
  `correction_confirms`）重试幂等返回首次结果，不重复更正。
- **批次两阶段提交**：预演（dry-run）与提交分离。预演在内存中把批次条目追加到当前台账
  末尾模拟重放（虚拟入库顺序与提交时的真实分配一致），返回逐条可行性与预计余量；
  提交凭据 = HMAC(计划哈希 + 台账版本)，台账每次写成功版本递增，凭据随之失效。
  提交在单事务中完成 凭据核对 → 业务校验 → 落库 → 全账重放，失败即整体回滚。
- **质检隔离**：隔离状态记录于 `quarantines`（原瓶主键、非空原因、时刻、操作人）。影响范围只依赖
  当前已提交来源关系（形成图）：在图上自每个瓶向上递归，来源链任意路径含隔离原瓶即命中，多原瓶
  命中取并集——故解除其一不影响其他原瓶的命中，直到来源链不再含任何隔离原瓶才放行。单条提交在
  事务内按当前台账判定；批次预演 / 提交把本批条目作为虚拟形成图叠加（批内前项受影响时后项引用
  同拦截），提交时事务内重跑，任一拦截整批回滚。设置 / 更新 / 解除隔离均递增台账版本，使未使用的
  批次 `v1` 凭据与更正 `c1` 凭据失效；各类成功请求（操作 / 批次 / 已确认更正）的重试仍走各自的
  幂等短路，返回首次结果，不受凭据过期影响。更正成功后形成图重建即完成影响重算，更正被守恒拒绝时
  回滚则影响不变。
- **并发守恒**：better-sqlite3 为同步驱动，所有写操作包裹在事务中串行执行，
  并发争用同一来源瓶时不会出现超扣；已用并发压测验证（10 并发争用 400 mL，
  恰好 4 笔成功、余量归零不为负；5 并发重复提交同一操作号仅扣量一次；
  5 并发重复提交同一批次仅扣量一次，其余幂等返回）。
- **明确报错**：编号冲突（409）、来源不存在（400）、操作时间早于来源创建时间（400）、
  体积非正整数（400）、超用（409）等均以 JSON `error.code / message / details` 返回，
  页面顶部同步展示。

## API 摘要

| 方法 | 路径 | 说明 |
|---|---|---|
| `POST` | `/api/bottles` | 登记原始瓶 `{bottle_id, sample_time, initial_volume}` |
| `POST` | `/api/operations` | 新建操作 `{op_id, target_bottle, op_time, sources:[{bottle, volume}]}` |
| `POST` | `/api/operations/:op_id/correct` | 更正取用量 `{sources:[{bottle, volume}]}` |
| `POST` | `/api/operations/:op_id/corrections/preview` | **（新增）** 更正预演（不写库）`{sources}` → 余量变化 `balance_changes` + 目标瓶及下游瓶贡献前后对照 `contribution_changes` + `confirm_token`；校验失败给出原因且不签发凭据 |
| `POST` | `/api/operations/:op_id/corrections/confirm` | **（新增）** 更正确认（单事务）`{sources, confirm_token}` → 201 / 同一凭据重试幂等 200 / 冲突 409（`PLAN_CHANGED` / `STALE_TOKEN`） |
| `POST` | `/api/batches/preview` | 批次预演（不写库）`{batch_id, items:[{op_id, target_bottle, op_time, sources}]}` → 逐条可行性 + 预计余量 + 可行条目目标瓶的预计原瓶贡献 `projected_contributions` + `commit_token` |
| `POST` | `/api/batches/commit` | 批次提交（单事务）`{batch_id, items, commit_token}` → 201 / 幂等 200 / 冲突 409 |
| `GET`  | `/api/batches` | 全部已提交批次 |
| `GET`  | `/api/batches/:batch_id` | 批次详情（按录入顺序的条目及来源） |
| `GET` | `/api/bottles` | 全部瓶及当前余量（含 `quarantined` / `quarantine_hits`） |
| `GET` | `/api/bottles/:id` | 单瓶详情：余量、来源谱系、相关操作、原瓶精确贡献量 `contributions`、`quarantine` / `quarantine_hits` |
| `GET` | `/api/bottles/:id/contributions` | 各原始瓶对该瓶的精确贡献量：原瓶编号、约分分数 mL、形成时总体积与合计；瓶不存在 404 |
| `PUT` | `/api/bottles/:id/quarantine` | **（新增）** 设置 / 更新原瓶隔离 `{reason（必填）, operator?}` → 201 / 同原因幂等 200 / 衍生瓶 400 / 瓶不存在 404 |
| `DELETE` | `/api/bottles/:id/quarantine` | **（新增）** 解除隔离 → 200（含 `still_affected`）/ 未隔离 409 `NOT_QUARANTINED` |
| `GET` | `/api/quarantine` | **（新增）** 隔离原瓶清单 + 全部受影响瓶及每瓶命中的隔离原瓶与原因；支持 `?bottle=xxx` 单瓶查询 |
| `GET` | `/api/operations` | 全部操作记录（含来源明细、目标体积与所属批次号 `batch_id`） |

时间格式统一为 `YYYY-MM-DDTHH:MM`（页面用 `datetime-local` 控件），体积单位为毫升（正整数）。

## 项目结构

```
├── Dockerfile            # node:20，构建时安装依赖，启动时确保依赖并初始化数据
├── docker-compose.yml    # 一键启动，映射 3000 端口，named volume 持久化数据
├── package.json
├── server.js             # Express 服务 + SQLite 台账 + 守恒校验 + 谱系查询 + 批次预演/提交 + 原瓶贡献量（精确分数）+ 更正预演/确认 + 质检隔离与影响拦截
└── public/index.html     # 单页工作台（总览 / 登记 / 操作 / 批量预演提交 / 记录与更正（预演+确认）/ 质检隔离 / 贡献量展示）
```
