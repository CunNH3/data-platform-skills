---
name: data-platform
description: 使用公司的数据平台搜索和读取数据、上传并登记数据、申请和释放 SSH 开发机、提交处理 Job 并查询状态和日志。用户需要在平台上处理机器人数据、使用 DataClient 或生成 TD 时使用；不用于集群管理或平台部署。
---

# 数据平台使用

## 每次使用先读取最新说明

- 每次调用本 Skill，执行平台操作前先重新请求 `https://api.github.com/repos/CunNH3/data-platform-skills/commits/main`，禁用缓存并取得本次 main 的提交 SHA，不复用上次调用的结果。
- 随后读取 `https://raw.githubusercontent.com/CunNH3/data-platform-skills/<SHA>/skills/data-platform/SKILL.md`，以该版本的索引为准；本次所需的 references 文档也从同一 SHA 的相对路径读取，不使用安装包里的旧正文替代。
- 一次调用固定使用上述 SHA，刷新一次后直接按最新索引执行；下一次调用重新检查 main。无需访问平台源码仓库，也不要求用户重新安装全部文档。
- 无法查询 main 或读取本次必需文档时，先提示具体失败原因并暂停平台操作，不能将本地缓存当作最新说明继续执行。

## 能力导航

| 用户要做什么 | 读取的说明 |
|---|---|
| 按特征搜索数据、处理并上传 TD：开发机运行或本地构建后提交 Job | [两个完整使用案例](references/use-cases.md) |
| 申请、查询、SSH 连接或释放开发机 | [开发机使用](references/dev-pod.md) |
| 提交处理任务、查询状态和日志 | [任务提交与跟踪](references/jobs.md) |
| 搜索数据、按 ID 查询、读取 TD | [DataClient 使用](references/data-client.md) |
| 上传数据、登记元数据、核验结果 | [数据读取与上传规范](references/data-transfer.md) |
| 生成或转换 TD | [TD 文件生成规范](references/td-format.md)，写入代码见 DataClient 使用 |
| 查具体 API 请求、参数和返回值 | [平台接口说明](references/api.md) |

## 共同约定

- 本 Skill 仅面向生产环境（prod），数据查询、读取、上传登记、开发机和 Job 均使用生产环境。用户请求其他环境时说明不支持并停止操作，不自动转到生产环境执行。
- 复用用户已提供的配置和本机认证，只合并询问缺失项；参数确认和资源检查按对应说明执行，不因 Skill 被调用就创建资源。
- 数据读取使用 DataClient；处理任务的数据结果统一为 TD，上传核验后必须登记到数据平台，返回结果数据 ID 和存储地址。
