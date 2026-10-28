# 任务提交与跟踪

本页说明现有 CreateJob、任务查询和日志接口的使用。processor 的任务定义与执行方式后续单独设计。

## 用户需要提供什么

| 信息 | 要确认的内容 |
|---|---|
| 运行环境 | 默认生产 prod；普通用户只使用生产，管理员 test 入口见 [管理员测试环境](admin.md) |
| 输入数据 | 数据 ID、查询条件或可访问的数据地址；查询得到多条数据时确认处理范围 |
| 处理要求 | 用户脚本／项目路径，或要求 Agent 编写的业务逻辑及成功标准 |
| 运行资源 | 未指定时推荐 8 核 CPU、16Gi 内存、0 GPU、超时 12 小时，展示后让用户确认；按任务规模和平台限制调整建议 |
| 数据结果 | 确认需要保存的数据内容；统一输出 TD 文件，保存到平台配置的对象存储并登记到数据平台 |
| 任务运行配置 | 使用已有任务的运行配置；现有 CreateJob 的必填字段见接口说明 |

- 合并询问缺失项，不重复询问已经确定的参数。用户提供代码仓库时，确定本次使用的版本。
- 业务含义不明确时先确认，例如“关键帧”可能指编码 I 帧，也可能指定时抽帧。
- 凭据复用本机已有认证或平台临时凭据，不要求用户把 AK/SK、密码写入任务参数。

## 镜像编译

本机需安装并启动 Docker，并具备基础镜像的拉取权限和任务镜像仓库的推送权限。先使用下面的常用基础镜像构建任务镜像。

构建目录准备 `main.py`、`requirements.txt` 和 `Dockerfile`；使用 SDK 时先按 [SDK 版本核对](data-client.md#每次使用前读取最新说明) 读取 PyPI 最新用法和 API，再将选定的兼容版本与业务依赖写入 `requirements.txt`。

```dockerfile
FROM default-acr-registry.cn-beijing.cr.aliyuncs.com/data_infra/base_image:build-20261016.1@sha256:d0a80f1008332d5b383ac91e92f963937d3dea8444b8651bd75dc4ab25b1be8c

USER root
WORKDIR /app
COPY requirements.txt ./
RUN python -m pip install --no-cache-dir -r requirements.txt
COPY main.py ./
RUN chown -R runner:runner /app
USER runner
ENTRYPOINT ["python", "-u", "/app/main.py"]
CMD []
```

- 多文件项目按实际使用的模块追加 `COPY`；数据、凭据和运行结果不打入镜像。
- 基础镜像包含 Python 和常用数据处理依赖，任务所需的 `current-data-py` 等 SDK 仍需按版本安装；示例设置任务入口并清空基础镜像的默认 CMD。

在构建目录执行，先将 `TASK_IMAGE` 替换为有推送权限、集群可拉取且符合服务白名单的任务镜像地址，使用本次唯一版本：

```sh
TASK_IMAGE="<任务镜像仓库>:<本次唯一版本>"
docker info
docker build --platform linux/amd64 -t "$TASK_IMAGE" .
# 完成小样本验证后再推送。
docker push "$TASK_IMAGE"
docker image inspect "$TASK_IMAGE" --format '{{json .RepoDigests}}'
```

记录镜像地址和 digest，将任务镜像填入 `CreateJob.spec.image`。任务参数通过 `parametersYaml` 传入，平台挂载为 `JOB_PARAMETERS_FILE` 指向的文件。

## Agent 执行流程

1. **读取接口说明。** 先读 [接口说明](api.md)，检查已授权目标环境的 API 是否可达；不通时核对地址、路由及代理，需要 VPN 时再提示用户。
2. **检查输入数据。** 按 [DataClient 使用](data-client.md) 搜索、查询并读取样本，确认数据内容和处理范围。
3. **准备任务程序。** 按本地 [TD 生成规范](td-format.md) 编写或检查脚本，读取 YAML 参数、输出进度，将 TD 结果上传并调用 RegisterData 登记到数据平台。
4. **核对运行配置。** 使用已准备好的任务配置，核对输入参数和入口行为；需要本地构建时按 [镜像编译](#镜像编译) 准备，不能假设平台支持直接执行代码包。
5. **检查运行条件。** 调用 GetResources，检查可用资源和 Job 接口限制。资源不足或参数不支持时直接说明，不能擅自降低配置。
6. **确认并提交。** 展示环境、数据范围、镜像、资源、超时和输出位置；确认尚未确认的配置，并满足当前环境的资源创建授权要求后调用 CreateJob。
7. **跟踪运行。** 保存 Job ID，通过 GetJob、GetJobLogs 跟踪到结束；失败时结合日志判断原因，修改业务逻辑或资源需求前与用户确认。
8. **核验并交付。** 检查输出 TD 及平台登记记录，确认内容符合要求；返回结果数据 ID、持久化地址、Job 状态和处理统计，后续通过 DataClient 读取结果。
