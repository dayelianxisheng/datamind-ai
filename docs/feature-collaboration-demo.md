# 新功能三人协作演练

这份文档用一个真实但范围可控的功能，完整演示三个人怎样共同开发、push、Review 和合并代码。第一次协作按本文顺序执行，熟悉以后再增加并行度。

## 演练功能：一键生成数据洞察

用户上传数据后，点击“一键生成洞察”，页面显示：

- 数据行数、列数和缺失单元格数量；
- 数值字段的最小值、最大值和平均值；
- DeepSeek 根据这些统计量生成两到三条简短洞察；
- 数据不足或模型失败时显示真实错误，不编造结果。

完成标准：

1. 统计值由后端实际计算；
2. 原始完整数据不发送给 DeepSeek，只发送字段信息和统计结果；
3. 前端能展示加载、成功、空结果和失败状态；
4. 原来的上传和自然语言分析功能仍然可用。

## 第一步：先定契约，不先写代码

三个人先共同确认两个接口。

### 数据画像接口

```http
GET /api/datasets/{dataset_id}/profile
```

示例响应：

```json
{
  "dataset_id": "abc123",
  "row_count": 24,
  "column_count": 4,
  "missing_cells": 0,
  "numeric_columns": [
    {
      "name": "sales",
      "min": 80000,
      "max": 260000,
      "mean": 151250
    }
  ]
}
```

### AI 洞察接口

```http
POST /api/datasets/{dataset_id}/insights
```

示例响应：

```json
{
  "dataset_id": "abc123",
  "insights": [
    "数据共 24 行、4 列，没有缺失单元格。",
    "sales 字段均值为 151250，范围为 80000 到 260000。"
  ],
  "profile": {
    "row_count": 24,
    "column_count": 4,
    "missing_cells": 0
  },
  "mode": "deepseek"
}
```

契约确认后不要边开发边随意改字段。如果必须修改，先在总 Issue 中说明，由受影响的成员确认。

## 第二步：拆成三个 Issue

### Issue A：前端洞察卡片

负责人：成员 A。

范围：

- 新增“一键生成洞察”按钮；
- 请求洞察接口；
- 展示统计摘要和洞察列表；
- 处理加载、错误和空结果状态。

验收：使用示例数据可以看到真实统计结果；模型不可用时页面不崩溃。

### Issue B：AI 洞察生成

负责人：成员 B。

范围：

- 接收 C 生成的 profile；
- 只把统计信息发送给 DeepSeek；
- 限制洞察数量和文本长度；
- 校验模型返回格式；
- 建立固定输入和预期事实的评测样例。

验收：洞察只能引用 profile 中存在的事实；无效模型响应会返回明确错误。

### Issue C：数据画像接口

负责人：成员 C。

范围：

- 根据 dataset_id 读取当前数据集；
- 计算行列数、缺失值和数值字段统计量；
- 实现 profile 接口；
- 检查不存在的数据集、非数值字段和空值场景。

验收：sales.csv 的统计值可以用独立 SQL 或计算器复核。

## 第三步：明确合并顺序

这个功能存在依赖关系，第一次演练采用顺序合并：

```text
C：数据画像接口
        ↓ 合并到 main
B：AI 洞察生成
        ↓ 合并到 main
A：前端洞察卡片
        ↓ 合并到 main
完整功能验收
```

A 和 B 可以提前设计界面、提示词和评测数据，但依赖代码要从最新 `main` 开始，避免三个人长期修改同一套临时代码。

## 第四步：成员 C 开发后端画像

### 创建分支

```bash
git switch main
git pull --ff-only
git switch -c feat/dataset-profile-api
```

完成代码后检查：

```bash
git status
git diff
```

只暂存相关文件：

```bash
git add backend
git commit -m "feat: add dataset profile API"
git push -u origin feat/dataset-profile-api
```

C 在 GitHub 创建 PR，在描述中粘贴接口响应示例和验证结果。A 检查响应是否满足页面需要，B 检查统计信息是否足够生成洞察。通过后由仓库负责人使用 **Squash and merge**。

## 第五步：成员 B 接入 AI

C 的 PR 合并后，B 从最新主分支创建新分支：

```bash
git switch main
git pull --ff-only
git switch -c feat/profile-insights
```

完成后：

```bash
git add backend
git commit -m "feat: generate insights from dataset profile"
git push -u origin feat/profile-insights
```

B 创建 PR。C 检查 B 是否只使用安全的画像函数、是否避免发送完整数据；A 检查响应格式是否符合已确认的契约。通过后合并。

## 第六步：成员 A 完成前端

B 的 PR 合并后，A 从最新主分支开始：

```bash
git switch main
git pull --ff-only
git switch -c feat/profile-insight-card
```

完成后：

```bash
git add frontend
git commit -m "feat: add dataset insight card"
git push -u origin feat/profile-insight-card
```

A 创建 PR。B 检查洞察文案是否正确展示，C 检查接口调用和错误处理。通过后合并。

## 第七步：三人联合验收

三个人都同步最新代码：

```bash
git switch main
git pull --ff-only
```

共同检查：

- [ ] 全新启动 MySQL、后端和前端；
- [ ] 加载示例数据；
- [ ] 点击“一键生成洞察”；
- [ ] 页面数值与数据库结果一致；
- [ ] DeepSeek 洞察没有超出统计结果；
- [ ] 断开模型后能看到明确错误；
- [ ] 原有地区排名和月度趋势仍然正常；
- [ ] `git status` 显示工作区干净。

## 如果开发期间 main 更新了

假设 A 正在开发时，B 的 PR 已经合并。A 在自己的功能分支执行：

```bash
git fetch origin
git switch feat/profile-insight-card
git merge origin/main
```

没有冲突就继续开发并 push。出现冲突时，先找修改同一文件的成员确认最终内容，再执行：

```bash
git add 冲突文件
git commit -m "merge: resolve main conflicts"
git push
```

不要直接覆盖不理解的队友代码，也不要对共享的 `main` 使用 force push。

## 群里怎么同步

开始任务：

```text
我领取 Issue：数据画像接口
分支：feat/dataset-profile-api
预计修改：backend 数据读取与 profile API
不会修改：前端和 AI 提示词
```

准备 Review：

```text
PR 已提交：<PR 链接>
请 A 检查接口字段，请 B 检查统计信息是否满足洞察生成。
本地已验证：示例数据、空值、数据集不存在。
```

合并完成：

```text
数据画像 PR 已合并，请 B 执行 git switch main 和 git pull --ff-only，
然后创建 feat/profile-insights 分支。
```

## 练习结束后应理解的规则

- 分支属于任务，不属于成员；
- push 只上传自己的分支，PR 才申请进入 `main`；
- 有依赖的任务按顺序合并，无依赖的任务才并行；
- 共享契约先确认，代码后实现；
- PR 作者负责修改，模块负责人负责 Review，仓库负责人负责合并；
- 合并后所有成员先更新 `main`，再开始下一个任务。
