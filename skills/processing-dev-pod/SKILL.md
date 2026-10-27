---
name: processing-dev-pod
description: 通过 processing-service API 申请、查询和释放带 SSH 的 CPU 或 NVIDIA GPU 开发机；不用于提交训练 Job 或管理集群。
---

# 开发机使用

环境默认生产，用户明确指定时才切换。创建前必须明确用户名、CPU（核或 m）、内存（GiB 或 MiB）、是否需要 GPU 及数量（张）；缺少的配置一次性询问并等待回复，不套用前端默认值或示例值。已说明的配置不重复问，齐全后汇总申请明细并取得用户确认；已有明确确认时不重复确认。随后检查资源，足够才创建。

## 先检查网络

- 从已有配置或测试环境 `ListPods` 取得当前有效的测试 Pod IP，探测 TCP 22，超时 3–5 秒。测试地址为 `https://data-test.current-robotics.work`，仅用于检查，申请仍走生产。
- 能连通就继续；不通再提示连接 VPN，提供[安装和认证文档](https://vcnqhq28cfdm.feishu.cn/wiki/Mhuswc0cXiVc3Rk4E3GcLy6vn1g?fromScene=spaceOverview)。已连接仍不通时检查 IP、路由和端口，不反复要求认证。
- 没有有效测试 IP 时不猜地址、不额外创建测试 Pod，待取得目标 Pod IP 后验证。公网 API 可访问不代表 SSH 可达。

## 申请参数

| 参数 | 填写要求 | 限制 |
|---|---|---|
| `username` | 必填 | 1–256 个小写英文字母，匹配 `^[a-z]{1,256}$`，如 `zoucunan`；中文姓名请用户提供英文或拼音 |
| `spec.cpu` | CPU 核数（核）或毫核（m） | 提交为正整数 m，最小 1m；0.5 核 = 500m，1 核 = 1000m |
| `spec.memory` | 内存大小（GiB 或 MiB） | 提交为正整数 Mi，最小 1Mi；1GiB = 1024MiB，API 对应 1024Mi |
| `spec.gpu` | GPU 数量（张），不需要填 0；明确无需 GPU 才省略 | 非负整数张；大于 0 时提交 `{"model":"5090","count":"1"}`，数量为十进制整数字符串 |
| `spec.sshPublicKey` | 必填 | 用户指定或本机匹配密钥对的单行公钥，去除首尾空白；私钥不上传 |

询问示例：“请提供用户名（仅小写英文）、CPU（核或 m，如 0.5 核）、内存（GiB 或 MiB，如 500MiB）、GPU 数量（张，5090，不需要填 0）；SSH 公钥可以提供内容或本机文件路径。”只问尚未明确的项，公钥可沿用用户已有的匹配密钥对。

只申请前端支持的 5090；不把 GPU 请求改成 CPU。资源单位准确换算；内存仅写数字或 M/G、单位不明确时先询问，不擅自按 MB 或 MiB 处理。不擅自取整或添加前端没有的固定上限。镜像、集群、namespace、存储挂载由服务端配置，创建请求不添加这些字段。

## 接口

生产接口前缀：`https://data.current-robotics.work/api/processing.service.v1.DataProcessingService/`。
全部使用 POST，头为 `Content-Type: application/json`、`Connect-Protocol-Version: 1`，HTTP 超时 75 秒。

| 方法 | 请求 | 结果 |
|---|---|---|
| `GetResources` | `{}` | `connected`、`total`、`allocated`、`available`、`reason` |
| `ListPods` | `{"username":"zoucunan"}`，或 `{}` 查询全部 | `pods` |
| `CreatePod` | 下方示例 | `pod` |
| `GetPod` | `{"podId":"返回的ID"}` | `pod` |
| `DeletePod` | `{"podId":"确认释放的ID"}` | `success`、`phase`、`message` |

```json
{
  "username": "zoucunan",
  "spec": {
    "cpu": "1000m",
    "memory": "1024Mi",
    "sshPublicKey": "ssh-ed25519 BASE64_PUBLIC_KEY user@host"
  }
}
```

上面的 CPU、内存仅演示请求格式，执行时全部换成用户指定的资源，公钥换成真实公钥；GPU 请求在 `spec` 增加上表的 `gpu`。使用 camelCase 字段，不自行生成 `podId` 或添加 `idempotencyKey`。

## 创建并连接

1. 用户确认申请明细后，调用 `GetResources` 检查 `available` 的 CPU、内存和所需 5090 GPU 数量，统一单位后与申请值比较。资源查询失败或无法判断时先停止，不创建。
2. 任一资源不足，直接提示“资源不足”，列出申请量、可用量和缺口，停止创建；不降低配置、不更换 GPU、不提交待调度 Pod。资源信息是预留余量，检查通过也不保证立即调度成功。
3. 资源足够后，提交前简短提醒：“重要代码和数据请保存到对象存储、NAS、CPFS 等远程持久化存储，Pod 本地文件可能随重建或释放而丢失。”不额外要求确认，也不假定这些存储已挂载。
4. 调用 `CreatePod`，保存 `pod.podId`。返回 `pending` 后每 5 秒查询 `GetPod`，最多继续等 2 分钟；仍未就绪则返回 ID、状态和原因。
5. 创建超时或中断后先按用户名 `ListPods`，核对时间和资源，不自动重试创建。历史记录按原用户名查询，不自动转换。
6. `ready` 且 `podIp` 非空后提供 `ssh -i <匹配私钥路径> root@<podIp>`；使用已加载的匹配密钥时可直接 `ssh root@<podIp>`。走 VPN、默认 22 端口，不创建 Service。用户授权验证时执行 `whoami`、`hostname`，未实测不声称登录成功。

交付只列 Pod ID、状态、资源、实际镜像、当前 IP 和 SSH 命令。SSH 超时检查路由及 TCP 22；端口转发不算直连成功，不关闭主机密钥校验。

## 释放：一次确认

- 用户要求释放后，核对具体 `podId`，提示：“释放 `<podId>` 后，Pod 本地未保存到远程持久化存储的重要代码和数据可能丢失。确认已保存并释放吗？”
- 收到一次明确确认后才调用 `DeletePod`。初始释放请求不计为这次确认；未回复、含糊回复或取消均保留 Pod。目标变更后重新确认，不因退出 SSH 或测试结束自动删除。
- `success=true` 且 `phase=released` 才算完成；`releasing` 继续查询，`failed` 展示 `message`。HTTP 200 不代表删除成功。

## 维护

接口变动时核对 `proto/processing/service/service.proto`；输入限制核对 `web/src/features/dev-pods/DevPods.tsx` 的 `CreatePodDialog`、`GPU_MODEL`。维护或验证 Skill 时仅调用查询接口，不创建或删除开发机。
