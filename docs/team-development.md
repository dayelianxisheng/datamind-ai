# 三人协作与开发边界

## 共同目标

现阶段先保证比赛 Demo 稳定，再并行增加能力。任何功能都要保持这条链路可运行：

```text
上传文件 → MySQL → 自然语言问题 → 受限 SQL → 真实结果 → 结论与图表
```

每位成员都应能独立启动项目、讲清数据流，并完整演示一次，避免只有一个人了解全系统。

## 三人职责

成员姓名确定后，把 A、B、C 替换为真实姓名。

| 角色 | 主责 | 默认负责目录 | 首批任务 |
| --- | --- | --- | --- |
| A：仓库负责人 / 前端 | GitHub 管理、版本集成、页面交互、图表和演示体验 | `frontend/**`、`README.md`、`.github/**` | 拆分前端组件；补加载、空状态和错误状态；维护演示脚本 |
| B：AI 分析 | DeepSeek 提示词、工具调用、结果解释、问题理解与效果评测 | 计划中的 `backend/app/analysis.py`、`backend/evals/**` | 抽离模型链路；建立 15—20 条固定问答评测；处理无效模型响应 |
| C：数据与后端 | 文件导入、MySQL、查询安全、数据画像和后端测试 | 计划中的 `backend/app/datasets.py`、`query.py`、`schemas.py`、`backend/tests/**`、`datasets/**` | 抽离数据与查询模块；补上传和危险 SQL 检查；增加基础数据画像 |

当前后端集中在 `backend/app/main.py`。为避免 B、C 同时改一个文件，第一项后端 PR 由 C 做纯模块拆分：保持接口和行为不变，将数据导入、SQL 查询、Pydantic 模型分别移到上述文件；合并后 B 再抽离 AI 链路。拆分期间其他人不改 `main.py`。

### 跨边界修改

- 可以改别人负责的模块，但 PR 必须由该模块负责人 Review。
- API 路径、请求体、响应字段、ChartSpec 或数据库结构属于共享契约，先提 Issue 说明变更，再编码。
- 重构 PR 不混入新功能；功能 PR 不顺手大改无关格式。
- `.env`、真实数据、个人密钥、数据库卷和构建产物永不提交。

## Git 仓库管理

### 分支

`main` 始终保持可启动，只通过 Pull Request 合并。分支从最新 `main` 创建：

```bash
git switch main
git pull --ff-only
git switch -c feat/short-name
```

命名统一使用：

- `feat/frontend-history`
- `feat/agent-retry`
- `fix/xlsx-empty-row`
- `docs/demo-script`
- `refactor/backend-modules`

提交信息用“类型 + 简短动作”，例如：

```text
feat: add dataset profile card
fix: reject duplicate xlsx headers
docs: clarify local startup
```

### Pull Request

每个 PR 只解决一个问题，尽量控制在可快速 Review 的范围。合并前必须：

1. 填写改动内容和验证方式；
2. 本地跑通受影响链路；
3. 至少一位队友 Review；
4. 解决冲突并确认没有提交密钥；
5. 使用 Squash merge，删除已合并分支。

仓库负责人在 GitHub 设置中完成：

- 邀请另外两位成员为 Collaborator；
- 保护 `main`，要求 PR 和至少 1 个审批；
- 禁止 force push 和删除 `main`；
- 建立 Issues 或 Project 看板：Backlog、In progress、Review、Done；
- 为每个 Issue 指定唯一负责人和验收条件。

## 接口契约

前后端当前只依赖四个接口：

| 方法 | 路径 | 负责人 |
| --- | --- | --- |
| GET | `/api/health` | C |
| POST | `/api/datasets` | C |
| POST | `/api/datasets/sample` | C |
| POST | `/api/analyses` | B；其中 SQL 执行由 C 负责 |

A 只依赖接口响应，不读取数据库。B 只通过 C 提供的安全查询函数访问当前数据集，不拼接真实物理表名。C 不修改 AI 的系统提示词或结论文案。

接口变更必须同时更新 README、FastAPI schema 和前端类型；PR 描述给出修改前后的 JSON 示例。

## 开发路线

### P0：稳定 Demo

- 完成后端模块拆分，保持现有行为；
- 统一错误响应并补最小端到端检查；
- 固定 5 个演示问题和预期结果；
- 完善加载、失败、空结果和模型断网状态；
- 完成三人各自电脑的启动验收。

完成标准：三台电脑都能从全新 clone 启动；地区排名、月度趋势、缺字段和危险 SQL 场景结果稳定。

### P1：比赛展示能力

A：

- 对话式结果历史；
- 图表切换、数据下载和适配答辩投屏；
- 分析步骤只展示后端返回的真实状态。

B：

- 查询失败时限次修正；
- 数据字段语义理解与问题澄清；
- 固定评测集，记录正确率、延迟和失败原因。

C：

- 数据画像：数值分布、缺失值、唯一值和异常值；
- 数据集列表、删除与过期清理；
- 更严格的 SQL 解析与只读数据库权限。

### P2：赛后扩展

根据比赛反馈再选择：多表关联、报告导出、预测分析、用户登录、团队空间、异步任务和部署。当前不提前搭建多 Agent 或微服务。

## 联调节奏

- 每人一次只领取一个主任务，Issue 写明输入、输出和验收条件。
- 每天合并一次小 PR，避免积累大分支。
- 共享契约改动时，先合并后端兼容版本，再合并前端切换，最后删除旧字段。
- 每次准备演示前，从全新数据库卷跑一遍五步演示流程。
- 比赛前指定一名演示人、一名设备与网络保障人、一名问答补充人，并准备本地示例数据和录屏备份。

## 新成员验收清单

每位成员完成后在群里回复结果：

- [ ] 已加入 GitHub 仓库并开启双因素认证；
- [ ] 已阅读 README、Demo 设计和本文件；
- [ ] 已在自己的电脑启动 MySQL、后端和前端；
- [ ] `/api/health` 中数据库连接正常；
- [ ] 已跑通示例数据的地区排名；
- [ ] 已确认 `.env` 不在 `git status` 中；
- [ ] 已领取一个 Issue 并创建自己的功能分支。
