# 维护规则

- 本仓库维护薄入口，完整能力索引、接口文档和业务 Skill 由 current-data-py 发布；用户使用时不依赖平台源码仓库。
- 平台关键接口、SDK 行为、参数限制或使用流程变化时，在 data-infra 同步更新包内索引、业务 Skill 和示例；入口机制变化时才更新本仓库。
- Skill 内容变更时，在 `skills/data-platform/references/updates.md` 追加记录并同步首页更新说明；使用唯一、递增的更新 ID，已发布 ID 不复用，记录按新到旧排列。
- 修改后验证 frontmatter、相对链接与代码示例；不为验证说明书而创建或释放平台资源。
- 本仓库公开可见，不提交凭据、私钥、kubeconfig、签名链接、业务数据和运行产物。
