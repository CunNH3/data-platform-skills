# 数据平台 Skills

[data-platform](skills/data-platform/SKILL.md) 是数据平台的 Agent 入口：通过 PyPI 安装或更新最新版 `current-data-py`，再读取包内的 Skill、接口信息和使用示例。

## 安装

将下面这句话发送给你的 AI Agent：

```text
请安装 https://github.com/CunNH3/data-platform-skills/tree/main/skills/data-platform 这个 Skill
```

手动安装时复制整个 `skills/data-platform` 文件夹，安装位置遵循所用 Agent 的要求。使用平台仍需相应认证和网络访问条件。

## 维护与发布

1. 在 data-infra 维护 SDK、Proto、能力索引和业务 Skill，构建并验证 SDK 包内说明书。
2. 按 SDK 发布流程更新版本并发布到 PyPI，验证发布包包含 `current_data_py.docs` 和完整文档。
3. 再发布 GitHub 入口改动，确保新入口指向的能力已随 SDK 提供；入口机制不变时无需为每个业务接口更新本仓库。

公开仓库不提交凭据、SSH 私钥、kubeconfig、签名链接、业务数据或运行产物。
