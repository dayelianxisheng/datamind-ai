# 给另外两位成员的交接消息

下面内容可以直接复制到小队群，发送仓库链接时不要附带 API Key。

---

DataMind AI 的第一版 Demo 已跑通，仓库：

<https://github.com/dayelianxisheng/datamind-ai>

当前完整链路是：上传 CSV/XLSX → 写入 Docker MySQL → DeepSeek 生成受限 SQL → MySQL 计算 → 页面展示中文结论、结果表和 ECharts 图表。

请先完成以下事项：

1. 把 GitHub 用户名发给仓库负责人，加入 Collaborator。
2. 阅读 `README.md`、`docs/demo-design.md` 和 `docs/team-development.md`。
3. 按 README 在自己电脑启动项目；DeepSeek Key 单独私下配置，不发群、不提交 Git。
4. 使用示例数据提问“哪个地区销售额最高？”，确认华东为 1,280,000 元。
5. 在群里回复：系统版本、启动是否成功、遇到的问题、希望承担 A/B/C 哪个角色。

暂定分工：

- A：GitHub 集成、Vue 前端、交互和图表；
- B：DeepSeek、工具调用、结果解释和效果评测；
- C：FastAPI 数据导入、MySQL、查询安全和数据分析能力。

开发统一从 `main` 拉功能分支，通过 PR 合并，不直接向 `main` 推功能。接口、数据库结构或 ChartSpec 要改时，先提 Issue 让三个人确认契约。

第一次同步会议请完成三件事：确定 A/B/C 对应姓名；共同走一遍数据流；从 P0 任务中每人领取一个 Issue。

---

## 仓库负责人发送前还要做

1. 在 GitHub 邀请两位成员，并保护 `main` 分支。
2. 创建 P0 Issues：
   - `refactor: 拆分后端模块但保持接口不变`
   - `feat: 完善前端加载、错误和空结果状态`
   - `test: 建立固定问答与安全回归清单`
3. 建立 Project 看板并为每个 Issue 指定负责人。
4. 通过私密方式让成员各自配置 Key；不要共享写进仓库的公共 Key。
5. 约定每日合并时间和比赛前完整回归时间。
