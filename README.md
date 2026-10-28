# 数据平台 Skills

供 AI Agent 使用的数据平台说明书：搜索和读取数据、处理并上传 TD、申请开发机、提交 Job 及查询日志。默认使用生产环境，所有用户均可明确选择测试环境。使用平台仍需具备相应权限和网络访问条件。

| Skill | 用途 |
|---|---|
| [data-platform](skills/data-platform/SKILL.md) | 数据平台完整使用流程，包含开发机直接运行代码、本地构建镜像提交 Job 两个案例 |

## 更新说明

- 最近更新：测试阶段取消管理员限制，所有用户均可明确选择测试环境；默认仍使用生产环境。
- 首次使用介绍最近一次更新，之后只提示尚未介绍的新记录；没有新增时简短说明。

完整历史见 [更新记录](skills/data-platform/references/updates.md)。

## 安装

将下面这句话发送给你的 AI Agent：

```text
请安装 https://github.com/CunNH3/data-platform-skills/tree/main/skills/data-platform 这个 Skill
```

手动安装时复制整个 `skills/data-platform` 文件夹，保留 `SKILL.md` 和其引用的全部文件；安装位置遵循所用 Agent 的要求。

## 发布与更新

1. 修改对应 Skill 的正文和引用文档，检查链接、请求参数及示例，再提交并推送到仓库；用户可从 `main` 获取最新版。
2. 需要固定版本时，在 GitHub 的 **Releases → Draft a new release** 创建如 `v0.1.0` 的 tag，选择发布提交、填写变更说明，再点击 **Publish release**。[GitHub 发布说明](https://docs.github.com/en/repositories/releasing-projects-on-github/managing-releases-in-a-repository)
3. 安装入口的本地文件不会自动覆盖；入口要求每次使用先读取 `main` 的最新索引和更新记录，提示变化后按需读取同一提交的说明。已安装旧入口的用户需更新一次，之后文档变化无需反复重新安装。

公开内容只包含使用说明和示例；账号凭据、SSH 私钥、kubeconfig、签名链接、业务数据和运行产物不进入仓库。
