---
name: data-platform
description: 使用数据平台查询和读取数据、上传登记 TD、申请 SSH 开发机、提交 Job 并查询日志；执行前获取 current-data-py 中的最新兼容说明书。
---

# 数据平台入口

## 每次使用先检查更新

1. 禁用缓存查询 `https://api.github.com/repos/CunNH3/data-platform-skills/commits/main`，取得本次提交 SHA；限流时可用 `git ls-remote https://github.com/CunNH3/data-platform-skills.git refs/heads/main`。读取 `https://raw.githubusercontent.com/CunNH3/data-platform-skills/<SHA>/skills/data-platform/SKILL.md` 作为最新入口，每次调用只刷新一次。
2. 查询 `https://pypi.org/pypi/current-data-py/json`，核对最新正式版本与 Python 兼容要求。在任务独立环境安装或升级选定版本，不覆盖用户其他项目的依赖；本次任务固定该版本。
3. 在该环境执行 `python -m current_data_py.docs --index`，先读取索引和它指向的业务 Skill、更新记录，再按能力读取所需 API 和参考文档。
4. 后续操作遵循包内说明，包括目标环境、资源配置确认、上传登记及开发机释放确认。

- SDK 中的索引负责“能力 → 指南 → 服务文档及 API／SDK 方法”映射；本入口不重复维护业务参数和执行流程。
- 包内文档可用 `python -m current_data_py.docs --skill`、`--api management`、`--api processing` 或 `--read <包内相对路径>` 读取。
- 在开发机和任务镜像中使用相同 SDK 版本；不要将另一个版本的文档与当前代码混用。
- 无法检查版本或读取必要文档时说明失败原因，不声称已获取最新版本。已安装版本没有 `current_data_py.docs` 时，说明需要发布或安装支持包内说明书的 SDK，先停止平台操作。

## 更新提示

读取 SDK 包内的更新记录，提示用户上次已介绍之后的新内容；首次只介绍最近一次更新。入口自身的变更见 [入口更新记录](references/updates.md)。
