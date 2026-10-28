# 平台接口说明

本文列出平台数据与任务接口；Python 示例见 [DataClient 使用](data-client.md)，上传规则见 [数据读取与上传规范](data-transfer.md)，开发机请求与 SSH 使用见 [开发机使用](dev-pod.md)。

## 调用入口

默认生产站点：`https://data.current-robotics.work`。用户明确选择 test 时，按 [测试环境](test-environment.md) 使用测试站点。

- HTTP POST，JSON 请求；设置 `Content-Type: application/json`、`Connect-Protocol-Version: 1`，认证沿用目标环境的正式方式。
- 数据接口前缀：`/api/management.service.v1.DataManagementService/`。
- 任务接口前缀：`/api/processing.service.v1.DataProcessingService/`。
- 下表使用 HTTP JSON 的 camelCase 字段；Python protobuf 使用对应的 snake_case 字段。
- 下文方法均拼接到对应服务前缀；除流式查询外可使用下面的调用函数。认证沿用本次环境的已认证 Session，示例不自动执行。

```python
def rpc(session, site, service, method, body, *, timeout=60):
    response = session.post(
        f"{site}/api/{service}/{method}",
        json=body,
        headers={"Content-Type": "application/json", "Connect-Protocol-Version": "1"},
        timeout=timeout,
    )
    response.raise_for_status()
    return response.json()
```

## 原子能力

| 能力 | 接口 | 输入及用途 |
|---|---|---|
| 搜索数据 | ListData | `filter`、`pageSize`、`pageToken`；处理分页，确认最终数据范围 |
| 统计数量／查询 ID | CountData／SelectData | 复用 filter，返回数量／数据 ID 数组 |
| 流式查询 | StreamDataReferences | filter、returnFields；SDK 优先使用 iter_data |
| 查询数据 | GetData | `dataId`；获取元数据和实际存储位置 |
| 读取 TD | DataClient.open_td | 传元数据对象和选定的 bucket／provider；不是直接传数据 ID |
| 申请结果上传地址 | CreateUploadURL | `dataId`、`objectKey`；可选 `provider` 和 `bucket`，两者同时提供 |
| 上传文件 | HTTP PUT 到上传地址 | 传文件内容；获取 URL 不等于已上传 |
| 上传存储临时凭据 | GetUploadCredentialsV2／V3 | 由 SDK 管理凭据刷新和文件上传 |
| 登记输出数据 | RegisterData | 结果上传并核验后必须登记，获得平台可查询的结果数据 ID |
| 查询资源 | GetResources | `{}`；检查 `connected`、`available`、`observedAt` |
| 申请／管理开发机 | CreatePod、GetPod、ListPods、DeletePod | 参数、返回值及释放确认见开发机使用文档 |
| 提交任务 | CreateJob | `name`、`idempotencyKey`、`spec`，见下方示例 |
| 查询任务列表 | ListJobs | 可选 `status`、分页参数；用于任务检索及提交异常核对 |
| 查询任务状态 | GetJob | `jobId`；返回状态、资源、时间及执行报告 |
| 查询运行日志 | GetJobLogs | `jobId`、`tailLines`；读取标准输出和错误输出 |

## 搜索数据：ListData

**请求**：`{"pageSize":50,"pageToken":"","filter":{"namePrefix":"nobody_"}}`。

**返回**：`data` 为元数据数组，`nextPageToken` 为下一页游标；非空时保持筛选条件继续查询，直到为空。

| filter 字段 | 含义／示例 |
|---|---|
| `namePrefix` | 数据名称前缀 |
| `taskNamePrefix` | 采集任务名称前缀，区分大小写的字面匹配 |
| `dataTypes` | 数组：`DATA_TYPE_EGOSUITE`、`DATA_TYPE_YAM`、`DATA_TYPE_CURRENT_V2`、`DATA_TYPE_PICO_GLOVE`，未分类为 `DATA_TYPE_UNSPECIFIED` |
| `recordingStartedRange` | `{"from":"2026-10-01T00:00:00Z","to":"2026-10-02T00:00:00Z"}`，可省略一端 |
| `recordingDurationRange` | `{"min":"10s","max":"600s"}`，可省略一端，已指定时长必须为正 |

- filter 可省略，多个条件组合筛选。pageSize 默认 50、最多 100；pageToken 原样传回，不自行生成。
- 元数据包含 id、name、storages、fileInfo、recordingInfo、dataType 等；可选字段可能为空。int64 JSON 返回值可能为字符串，默认值字段可能省略。

## 统计数量与查询 ID：CountData／SelectData

| 方法 | 请求 | 返回 |
|---|---|---|
| CountData | `{"filter":{"namePrefix":"nobody_"}}` | `{"totalCount":"123"}` |
| SelectData | `{"filter":{"taskNamePrefix":"inspection"}}` | `{"dataIds":["匹配的数据ID"]}` |

- 两者使用相同 MetadataFilter。数量反映查询时刻，并发新增数据可能使后续搜索结果变化。
- 仅需要 ID 时使用 SelectData，读取 TD 前通过 GetData 或流式查询取得元数据。

## 流式搜索：StreamDataReferences

**请求**：`{"filter":{"namePrefix":"nobody_"},"returnFields":["name","file_info.topics.name","recording_info.task.name"]}`。

**返回**：多个批次，每批包含 data 数组；始终返回 ID 和 storages，未指定其他字段时返回默认精简元数据。

- returnFields 使用 protobuf 字段路径，不改成 camelCase。使用 SDK 的 iter_data 或 stream_data_references，提前结束时关闭迭代器。
- 这是流式 RPC，不能使用上面的普通 JSON 调用函数读取；SDK 示例见 [DataClient 使用](data-client.md)。

## 查询单条数据：GetData

**请求**：`{"dataId":"11111111-1111-4111-8111-111111111111"}`，或 `{"optionalId":"业务唯一标识"}`，二者只能提供一个。

**返回**：`{"data":元数据对象}`；不存在时返回 not_found。

| storages 字段 | 含义 |
|---|---|
| `provider` | `STORAGE_PROVIDER_OSS`／`STORAGE_PROVIDER_TOS`；历史未指定记录按 OSS 处理 |
| `bucket` | 桶名 |
| `objectKey` | 桶内路径，不含 oss:// 或 tos:// 前缀 |

## TD 读取与写出

- 读取：`DataClient.open_td(metadata, ...)`，支持 bucket、provider、use_internal_endpoint 和 enable_decode。
- 写出：使用 `turbodata.Writer` 和平台 schema 生成真实 `.td`；完整代码见 [DataClient 使用](data-client.md)，不通过修改其他文件扩展名生成 TD。

## 上传结果：CreateUploadURL + PUT

```json
{
  "dataId": "22222222-2222-4222-8222-222222222222",
  "objectKey": "job-results/custom-task/22222222-2222-4222-8222-222222222222/result.td"
}
```

**返回**：url、expiresAt、provider、bucket、objectKey；以返回值为实际上传位置，随后 PUT 文件内容到 url。

| 请求字段 | 要求 |
|---|---|
| `dataId` | 为输出数据生成独立的规范 UUID，不复用输入 ID |
| `objectKey` | 建议明确指定唯一的 .td 相对路径，无开头 /、.. 等非法段 |
| `provider`、`bucket` | 可同时指定已配置的云和桶；同时省略使用默认 OSS 上传位置 |
| `dataType` | 可选，省略 objectKey 时参与默认路径选择；不会自动登记元数据 |

- PUT 不增加未参与签名的认证或自定义头。成功上传文件后，需单独 RegisterData 才能进入平台搜索。

## 上传存储临时凭据

| 方法 | 请求 | 返回／用途 |
|---|---|---|
| GetUploadCredentialsV2 | `{"dataType":"DATA_TYPE_YAM","dataKind":"DATA_KIND_CONVERTED","useInternalEndpoint":false}` | 服务选择 OSS 上传桶及 objectPrefix，返回对应临时凭据 |
| GetUploadCredentialsV3 | 在 V2 上传请求中增加 provider | 服务选择对应云的上传桶与前缀，返回临时凭据 |

- V2／V3 上传须明确 dataType、dataKind，V3 还须明确 provider；不接受 UNSPECIFIED，只支持服务配置的组合。
- SDK 存储客户端管理刷新，通常无需自行处理 AK/SK；凭据只在进程内签名，不打印或写入镜像、任务 YAML。
- useInternalEndpoint 只改变 endpoint，仍需确认运行位置的内网可达；历史 V1 行为见 SDK 文档中的上传说明。

## 登记结果：RegisterData

```json
{
  "data": {
    "id": "22222222-2222-4222-8222-222222222222",
    "name": "custom-task-result",
    "storages": [{
      "provider": "STORAGE_PROVIDER_OSS",
      "bucket": "实际上传返回的桶名",
      "objectKey": "实际上传返回的路径.td"
    }],
    "fileInfo": {"sizeBytes": "1024", "checksum": "实际文件SHA256"}
  }
}
```

**返回**：`{"data":已登记元数据}`。自定义任务的输出必须登记，随后用结果 ID 调用 GetData 核对记录，通过 DataClient.open_td 读取结果。

- 文件大小和校验和必须由实际 TD 计算，示例数字不可直接提交。
- 必填非空 name、至少一条有效 storage，以及 fileInfo 的正数 sizeBytes 和非空 checksum；省略 id 时服务生成 ID。
- 可补真实 topics、dataType、devices、recordingInfo；不要直接复制处理后已失效的输入统计。
- 重复 ID 或非空 optionalId 返回 already_exists，先查询核对再决定重试。此接口不上传文件，也不验证远端对象存在。
- 登记失败保留已上传文件的位置，修复后用同一结果 ID 重试登记；不能把仅上传成功当作完整交付。

## 查询资源：GetResources

**请求**：`{}`。

| 返回字段 | 含义 |
|---|---|
| `clusterId`、`connected`、`reason` | 当前服务连接的集群与连接情况 |
| `total`、`allocated`、`available` | CPU／内存 quantity、按型号统计的 GPU 数量 |
| `nodeCount`、`unavailableNodeCount` | 选定节点及其中不可用节点数量 |
| `observedAt` | 观察时间 |

- 查询失败或资源不足时停止提交；总量检查通过也不保证单节点调度成功。

## CreateJob 请求

下例为现有 CreateJob 协议，运行配置、数据和资源替换为本次确认的值；使用流程见 [任务提交与跟踪](jobs.md)。

```json
{
  "name": "custom-data-task",
  "idempotencyKey": "unique-request-uuid",
  "spec": {
    "image": "<集群可拉取且符合白名单的任务镜像>",
    "parametersYaml": "data_id: <已确认的数据ID>\noutput_prefix: job-results/custom-task/<本次运行ID>/\n",
    "resources": {"cpu": "8", "memory": "16Gi", "gpu": 0},
    "timeout": "43200s"
  }
}
```

**返回**：`{"job":任务对象}`；保存 job.jobId，并检查返回的 status，HTTP 成功不代表业务已完成。

| 参数 | 限制 |
|---|---|
| `name` | 1–128 字符 |
| `idempotencyKey` | 可选但建议提供，本次提交唯一，最多 128 字符 |
| `spec.image` | 已推送、可拉取且符合服务白名单的任务镜像 |
| `spec.parametersYaml` | 合法 YAML 字符串，1 字节至 512 KiB，键由任务脚本定义 |
| `spec.resources` | 正数 CPU／内存 quantity，非负 GPU 数量，同时满足服务上限 |
| `spec.timeout` | 正数 protobuf Duration，建议值 12 小时为 43200s，仍须满足服务上限 |

- CPU、内存为 Kubernetes quantity；GPU 为数量。当前 Job 接口不接受 GPU 型号、集群 ID、资源池、代码地址、SSH 参数或 command／args。
- 超时必须为正且不超过服务限制；当前不支持无限超时。CPU／内存上限和镜像白名单由部署配置决定，以目标环境实际校验为准。
- GetResources 是资源申请量视角，不是实时利用率，也不能保证单个节点能满足该请求；总空闲量不能替代节点调度条件。
- 提交成功后保存 `job.jobId`。同一幂等键只用于同一请求；当前服务不会替你核对复用幂等键时参数是否改变。
- 镜像中的入口程序负责执行业务；服务把 `parametersYaml` 挂为 `/job/params.yaml`，并注入 `JOB_PARAMETERS_FILE` 和 `ENV`。
- 没有单独对外的 Submit API；请求超时先核对已提交任务，再用相同幂等键和相同参数重试。

## 查询任务列表：ListJobs

**请求**：`{"pageSize":50,"pageToken":"","status":"JOB_STATUS_RUNNING"}`；省略 status 查询全部状态。

**返回**：jobs、nextPageToken；默认每页 50，最多 100，游标原样传回。可按返回的 name、spec、创建时间核对本次任务。

## 查询任务状态：GetJob

**请求**：`{"jobId":"实际Job ID"}`。

**返回**：job 包含 jobId、name、kubernetesJobName、spec、status、createdAt、updatedAt、startedAt、finishedAt、report。

- 状态为 SUBMITTING、PENDING、RUNNING、SUCCEEDED、FAILED，JSON 名称带 JOB_STATUS_ 前缀；SUCCEEDED 和 FAILED 是终态。
- report 包含 exitCode、reason、message；输出 TD 地址由任务程序提供，不是 GetJob 内置结果字段。

## 查询日志：GetJobLogs

**请求**：`{"jobId":"实际Job ID","tailLines":500}`。

**返回**：content、podName、state、limitReached。

- state 为 JOB_LOG_STATE_AVAILABLE、WAITING 或 UNAVAILABLE，后两者同样带 JOB_LOG_STATE_ 前缀；不能把未就绪当成无日志且运行成功。
- tailLines 默认 500、最多 2000，内容最多 256 KiB；limitReached 表示达到字节上限，内容可能不完整。

- 任务程序输出 TD 持久化地址和处理统计；日志可能因 Pod 清理／轮转不可用。
- Agent 断开不影响已提交 Job，但后续查询和通知依赖 Agent 会话或另行配置的跟踪机制。
