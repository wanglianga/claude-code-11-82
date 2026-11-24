# 区域影像中心受控分析平台

基于 **Vue 3 + Spring Boot 3 + PostgreSQL** 的受控影像分析沙箱。平台只处理**结构化影像特征与随访**，
不存储、不输出任何原始影像或患者级身份记录；通过批准编译、联合推断审查、累计披露预算与交付冻结/复现，
防止由多次合法聚合反推出小群体。

## 一、核心能力与对应实现

| 题目要求 | 实现位置 |
| --- | --- |
| 批准编译器生成可复现队列与字段投影 | `ApprovalService`、`CohortService`（确定性 JSON + specHash/cohortHash） |
| 查询解析器规范化筛选/分组/时间窗/输出，并与批准范围相交 | `QueryParser`（注释等展示文本不入哈希） |
| 结果关系账（队列/维度/过滤/抑制/舍入/水位） | `QueryAttempt`、`Deliverable.manifestJson` |
| 推断审查器：以所有已披露输出 + 公开总数为约束计算受抑制单元可行整数区间 | `InferenceEngine`（界传播 + DFS 可行性枚举）、`DisclosureReviewService` |
| 披露预算按研究空间 + 有效期累计窄化程度，不随查询文本/版本号重置 | `BudgetService`、`BudgetLedger`（空间×季度账本） |
| 交付器冻结结果/算法/审批/审查证明；复现器按当时审批与水线重建 | `DeliverableService`（sha256 清单 + 逐单元复现比对） |
| 最小发布单元 5；每空间每季度 10 风险点；唯一计数硬阻断；耗尽后仅零窄化结果 | `ResearchSpace`、`DisclosureReviewService` 裁决链 |
| 固定表组整组联合核验 / 交互会话实时扣共同预算（同一规则两条路线） | `reviewTableSet` / `review` + `InteractiveSession` |
| 撤回/到期拒新读，旧结果仅审计留存；算法升级只新增版本不改写旧结论 | `ApprovalService.requireReadable`、`AnalysisVersionService`、`DataSeeder`(feat-v1/v2) |

### 风险点如何计算

- 候选约束分两层：**基线层** = 历史约束 + 本次抑制承诺（抑制是平台安全保证，不计点）；
  **增量层** = 精确披露等式、舍入区间、分区恒等。
- 对每个受保护单元（真实计数 < 5），比较加入增量约束前后的可行整数集合，
  **每少一个可能值计 1 点**；可行集被约束为唯一值时，无论预算余额一律硬阻断。
- 阻断不扣点、不留约束；同语义查询重复提交幂等零扣点；预算耗尽后只放行增量为 0 的结果。

## 二、目录结构

```
82/
├── backend/          Spring Boot 3.5 / Java 21 后端（JPA + JWT + PostgreSQL）
│   ├── src/main/java/com/sandbox/imaging/
│   │   ├── domain/       JPA 实体（15 张表）
│   │   ├── inference/    线性整数约束与可行性枚举引擎
│   │   ├── service/      批准编译、队列、查询解析/执行、披露审查、交付复现、预算
│   │   ├── web/          REST 控制器与视图投影
│   │   └── bootstrap/    虚构演示数据种子
│   └── e2e_verify.py     端到端场景验证脚本（35 项断言）
├── frontend/         Vue 3 + Vite + Pinia + Vue Router（10 个审计/操作视图）
└── docker-compose.yml
```

## 三、一键启动（Docker Compose）

```bash
docker compose up --build
# 前端：http://localhost:8088
# 后端 API：http://localhost:8080/api
# PostgreSQL：localhost:5432（库/用户/密码 ccdev）
```

> 容器内 nginx 负责托管前端静态资源并把 `/api/` 反向代理到后端。

## 四、本地开发启动（使用预装运行时）

前置：PostgreSQL 运行在 `127.0.0.1:5432`（库/用户/密码 `ccdev`）。

### 后端（JDK 21 编译目标，本机 JDK 25 可运行）

```bash
cd backend
/opt/maven/bin/mvn spring-boot:run          # 默认读取 application.yml 的本地数据源
# 首次启动自动建表并写入虚构数据（DataSeeder）
```

### 前端（Node 22）

```bash
cd frontend
npm install
npm run dev        # http://localhost:5173 ，vite 已配置 /api 代理到 8080
```

## 五、测试账号

| 角色 | 用户名 | 密码 | 权限 |
| --- | --- | --- | --- |
| 管理员 | `admin` | `admin123` | 批准编译/激活/撤回、预算上限治理、水线、队列 |
| 分析师 | `analyst` | `analyst123` | 队列复现、查询干跑/提交、固定表组、交付冻结与复现 |

## 六、虚构演示数据

21 名脱敏患者（医院 A 14 名、医院 B 7 名），无原始影像、无真实身份：

- **APV-S1**（2026 上半年队列，水线 WL-2026Q3）：A=14、B=1，公开总数 15 —— 相减推断阻断演示。
- **APV-S2**（2026 下半年队列，水线 WL-2026Q4）：A=8、B=7 —— 无小单元安全披露演示。
- **APV-EXPIRED**：有效期 2025 年 —— 到期拒新读演示。
- 特征算法 `feat-v1` / `feat-v2` 两版 —— 算法升级只生成新分析版本，旧冻结结论与披露成本保留。

## 七、验证方式（均已实际执行）

### 1. 后端单元测试（推断引擎）

```bash
cd backend && /opt/maven/bin/mvn test
# InferenceEngineTest：5 项，含 14/1 唯一推出、8/7 可披露、舍入区间、矛盾系统无解
```

### 2. 端到端场景验证（35 项断言）

后端启动后：

```bash
python3 backend/e2e_verify.py
```

覆盖：

1. **14/1 联合阻断**：总数表放行后，医院表干跑即识别 `hospital=B` 可行集被唯一推为 `{1}`，
   正式提交 BLOCKED 且不扣预算；固定表组（总数+医院）整组阻断；改用舍入到 5 后 B 被抑制、可行集恢复多值并放行。
2. **8/7 保留可用结果**：两单元均 ≥5，医院表 A=8/B=7 放行；同语义查询重复提交零扣点；
   冻结交付物后按当时审批与水线逐单元复现成功（outputMatch / cohortHashMatch / sha256 均一致）。
3. **撤回与到期**：撤回 APV-S2 后新查询返回 403，旧交付物仍可审计读取与复现（标记 approvalReadableNow=false）；
   到期批准新建版本被 403 拒绝。
4. **算法升级**：feat-v2 生成独立新版本并重新参加预算审查，旧交付物载荷哈希不变、未被改写。
5. **预算耗尽规则**：管理员把季度上限压到已用量后，窄化查询（1 风险点）被预算阻断，
   零窄化查询仍 ALLOWED 且账本不增长。

### 3. 前端

```bash
cd frontend && npm run build     # 生产构建通过（vite build）
```

登录后打开「场景验证导览」，可分别一键运行三个规定场景并查看联合推断可行集、裁决与审计留痕；
其余视图提供批准/版本、队列复现、交互查询、固定表组、交付复现、预算约束图与审计事件。

## 八、安全边界说明

- 对外输出单元对被抑制数据不携带真实计数（`displayCount=null`），真实值仅存于内部台账与审计。
- 变量按 `批准@水线` 作用域隔离，不同队列的同名维度不会跨队列错误串扰。
- 可行性枚举设解数/节点上限，触顶时 fail-closed 阻断，避免在超大解空间下给出不完整的安全结论。
