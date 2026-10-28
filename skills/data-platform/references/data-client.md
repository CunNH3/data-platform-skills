# DataClient 使用

DataClient 继承 ManagementClient，负责查询元数据和远端 TD 读取；结果上传使用签名 URL 或 OSSClient／TOSClient，TD 写出使用 TurboData Writer。

数据范围、对象位置和上传登记规则见 [数据读取与上传规范](data-transfer.md)；本文提供对应代码用法。

## 每次使用前读取最新说明

1. 每次任务需要 SDK，先禁用缓存并重新读取 [current-data-py 的 PyPI 说明](https://pypi.org/project/current-data-py/) 和 [发布元数据](https://pypi.org/pypi/current-data-py/json)，核对最新版本、用法及其链接的官方 API 文档，不沿用上次查询结果。涉及 TD 读写时，同样读取 [turbodata](https://pypi.org/project/turbodata/) 的最新说明。
2. 检查实际运行环境的已安装版本，例如 `importlib.metadata.version("current-data-py")`。已安装版本与最新发布不同，读取 `https://pypi.org/pypi/current-data-py/<版本>/json` 的对应说明；本文示例与 SDK API 不一致时，以本次核对版本的 API 为准。
3. 文档没有说明的方法或字段，从该版本的发布包源码、`inspect.signature` 或 Protobuf `DESCRIPTOR` 核对，不猜测 API。需要安装或升级时，在任务虚拟环境或镜像中固定选定且兼容的版本，不直接升级用户已有环境。
4. PyPI 说明用于核对 SDK 用法，不改变本 Skill 的环境权限和上传登记等平台约定。无法获取说明或确认本次所需 API 时，提示具体问题并暂停依赖该 SDK 的操作。

## 1. 安装与连接

```sh
SDK_VERSION="<本次核对并选定的版本>"
python -m pip install "current-data-py==$SDK_VERSION" requests PyYAML
```

- 使用平台已发布且验证兼容的 SDK 版本，在运行环境中固定实际安装的依赖版本。无需访问数据平台代码仓库。
- 首次导入 SDK 前按目标环境设置 `ENV`，默认 `prod`；明确选择 test 时使用 [测试环境](test-environment.md) 的连接示例。Job 中检查平台注入的 ENV 与已确认目标一致，不覆盖为其他环境。
- 在源码仓库里执行示例时，用已安装的 wheel 并从源码目录外运行，避免同名未构建源码覆盖已安装包。

```python
import os

if os.environ.get("ENV", "prod") != "prod":
    raise RuntimeError("此示例只连接生产环境（ENV=prod）")
os.environ["ENV"] = "prod"

from current_data_py.remote import DataClient, MetadataFilter

with DataClient() as client:
    # 在此作用域内查询、读取及传输文件
    pass
```

- `DataClient()` 的默认地址由已确认目标的 ENV 决定，保持 TLS；上例为默认的生产连接。认证使用平台认可的 channel／拦截器；原始 stub 支持 metadata，iter_data／open_td 没有逐次认证参数。

## 2. 搜索数据：iter_data

```python
from contextlib import closing
from current_data_py.remote import DataClient, MetadataFilter

with DataClient() as client:
    with closing(client.iter_data(
        MetadataFilter(name_prefix="nobody_"),
        return_fields=["name", "file_info.topics", "recording_info.task.name"],
        timeout=30,
    )) as records:
        for metadata in records:
            print(metadata.id, metadata.name)
            for storage in metadata.storages:
                print(storage.provider, storage.bucket, storage.object_key)
```

| 参数 | 含义 |
|---|---|
| filter | MetadataFilter；不提供则搜索全部数据 |
| return_fields | 精简元数据字段路径，始终返回 id 和 storages；不是对 TD topic 的过滤 |
| timeout | 整条流的秒数上限，不是每批次超时 |

- iter_data 基于 StreamDataReferences，不需要手动分页；流式记录可能不含完整元数据，完整内容使用 GetData。
- 提前退出必须关闭迭代器；上例的 closing 同样适用于 break 或只取第一条。其他查询方式按本次核对版本的 API 使用。

## 3. 组合筛选、计数和分页

使用 ParseDict 可以直接复用 HTTP 文档中的 filter，自动转换 protobuf 时间和时长：

```python
from google.protobuf.json_format import ParseDict
from current_data_py.remote import MetadataFilter
from current_data_py.management import service_pb2

query = ParseDict({
    "namePrefix": "nobody_",
    "taskNamePrefix": "inspection",
    "dataTypes": ["DATA_TYPE_CURRENT_V2"],
    "recordingStartedRange": {"from": "2026-10-01T00:00:00Z"},
    "recordingDurationRange": {"min": "10s", "max": "600s"},
}, MetadataFilter())

# 以下代码在已打开的 client 作用域中执行。
count = client.stub.CountData(service_pb2.CountDataRequest(filter=query), timeout=30)
ids = client.stub.SelectData(service_pb2.SelectDataRequest(filter=query), timeout=30)
print(count.total_count, list(ids.data_ids))

page_token = ""
while True:
    page = client.stub.ListData(service_pb2.ListDataRequest(
        filter=query, page_size=50, page_token=page_token,
    ), timeout=30)
    for metadata in page.data:
        print(metadata.id)
    page_token = page.next_page_token
    if not page_token:
        break
```

以上条件是组合示例，不默认对用户任务施加名称、类型或日期过滤。

## 4. 按 ID 获取元数据

```python
from current_data_py.management import service_pb2

metadata = client.stub.GetData(service_pb2.GetDataRequest(data_id=data_id), timeout=30).data
# 如果用户提供的是业务唯一标识，改用 optional_id，不能同时提供两个字段。
```

- data_id 不是对象路径，GetData 返回的 metadata 才能传给 open_td；如所用版本提供便捷方法，按该版本 API 使用。

## 5. 读取 TD 和解码消息

以下在已取得 metadata 和已打开 client 的作用域中执行；先查看 topic，再按实际业务选择。

```python
from contextlib import closing

with client.open_td(metadata) as reader:
    topics = [
        topic
        for group in reader.summary().topics_infos
        for topic in group.topic_metadatas
    ]
    for topic in topics:
        print(topic.name, topic.metadata.get("schema_name"))

    if topics:
        with closing(reader.read_messages(topic_names=[topics[0].name])) as messages:
            for message in messages:
                print(message.topic_name, message.timestamp, type(message.value).__name__)
```

| open_td 参数 | 行为 |
|---|---|
| metadata | 平台元数据对象，含 storages，不是数据 ID 字符串 |
| bucket、provider | 选择实际存储位置；provider 可用 OSS／TOS，相同桶名跨云时明确指定 |
| use_internal_endpoint | 默认 False；True 使用目标云的内网 endpoint，OSS 还需明确 bucket |
| enable_decode | 默认 True：普通消息解析为 protobuf，H.264 视频解码为 RGB PIL Image |
| endpoint | 仅 TOS 支持公网 endpoint 覆盖，不能与内网模式同时使用 |

- 默认 OSS 优先，TOS-only 记录可自动选择唯一桶；多桶或跨云歧义时明确指定，失败不自动跨云回退。
- `read_messages` 可传 topic_names、start_timestamp、end_timestamp，时间戳单位纳秒、两端包含。默认解码只支持正向读取。
- 需要判断编码 I 帧时使用原始数据与解码器；默认 RGB 图片不保留编码帧类型。

## 6. 读取原始消息与内网／TOS 选择

```python
with client.open_td(metadata, enable_decode=False) as reader:
    schemas = {
        topic.name: topic.metadata["schema_name"]
        for group in reader.summary().topics_infos
        for topic in group.topic_metadatas
    }
    for message in reader.read_messages(copy=True, video_decodable=True):
        # message.data 为原始 protobuf 字节；按 schemas 和业务要求处理。
        print(message.topic_name, message.timestamp, len(message.data))
```

- 按时间范围读取视频时保留 video_decodable=True，依赖帧可能超出请求范围，业务输出应按目标时间再次过滤。
- OSS 内网示例：`client.open_td(metadata, provider="OSS", bucket="实际桶名", use_internal_endpoint=True)`，只在网络可达时使用。
- TOS 公网加速示例：`client.open_td(metadata, provider="TOS", bucket="实际桶名", endpoint="https://tos-accelerate.volces.com")`；签名 region 使用平台返回值，不按 endpoint 猜测。

## 7. 写出 TD

生成前先读取本 Skill 自带的 [TD 生成规范](td-format.md)，按规范选择 topic、schema、分组、压缩和 chunk。下面演示 PoseStamped，业务任务使用真实数据和时间戳。

```python
import time
from turbodata import Writer, ChunkConfig, ChunkThresholdMode
from current_data_py.schema import PoseStamped
from current_data_py.schema.message_metadata import topic_metadata

timestamp = time.time_ns()
message = PoseStamped()
message.header.stamp.sec, message.header.stamp.nanosec = divmod(timestamp, 1_000_000_000)
message.header.frame_id = "base"
message.pose.position.y = 1.0
message.pose.orientation.w = 1.0
topic = "/states/head/pose"

with open("result.td", "wb") as output, Writer(output) as writer:
    writer.open_topics(
        [topic], [topic_metadata(type(message))],
        compression=True,
        chunk_config=ChunkConfig(mode=ChunkThresholdMode.SIZE, size=1_000_000),
    )
    writer.write_message(topic, message.SerializeToString(), timestamp)
    writer.close_topic()
```

- 结束 writer、关闭文件后再上传；原始采集的派生数据保留对应时间戳，不能全部换成处理时间。
- 先重新读取本地 TD 核对内容，再上传；不能把 JPEG、ZIP 或 JSON 改扩展名当作 TD。

## 8. 上传输出 TD

通用结果优先使用签名 URL，服务选择默认 OSS 桶；运行过程中不需要用户手动提供桶凭据。

```python
import uuid
import requests
from current_data_py.management import service_pb2

output_id = str(uuid.uuid4())
upload = client.stub.CreateUploadURL(service_pb2.CreateUploadURLRequest(
    data_id=output_id,
    object_key=f"job-results/custom-task/{output_id}/result.td",
), timeout=30)
with open("result.td", "rb") as source:
    response = requests.put(upload.url, data=source, timeout=(10, 600))
    response.raise_for_status()

# 保留 output_id、upload.provider、upload.bucket、upload.object_key，供核验及登记。
```

如果任务已有明确的数据类型和分类，也可用管理临时凭据的文件客户端：

```python
from current_data_py.oss import OSSClient

with OSSClient(client, permission="upload",
               data_type="YAM", data_kind="CONVERTED") as objects:
    key = f"{objects.object_prefix}{output_id}.td"
    objects.upload_file(key, "result.td")
    print(objects.bucket, key)
```

- YAM／CONVERTED 仅为配置组合示例，不默认作为所有结果的分类；V2 上传必须使用服务已配置的类型与分类。
- OSSClient 不传 data_kind 时走历史 V1，未指定 data_type 按 RAW 路由；这与 CreateUploadURL 默认上传位置不是同一规则，不随意互换。
- TOSClient 上传需明确 data_type、data_kind，桶和前缀由服务选定；两个存储客户端的 upload_file 默认禁止覆盖，不会自动登记元数据。

## 9. 登记结果并通过 DataClient 读取

以下衔接第 8 节的签名上传变量，使用真实文件大小、校验和及上传位置：

```python
import hashlib
from pathlib import Path
from current_data_py.management import Metadata, metadata_pb2, service_pb2

path = Path("result.td")
digest = hashlib.sha256()
with path.open("rb") as source:
    for chunk in iter(lambda: source.read(1024 * 1024), b""):
        digest.update(chunk)

output_metadata = Metadata(
    id=output_id,
    name="custom-task-result",
    storages=[metadata_pb2.Storage(
        provider=upload.provider, bucket=upload.bucket, object_key=upload.object_key,
    )],
    file_info=metadata_pb2.FileInfo(
        size_bytes=path.stat().st_size, checksum=digest.hexdigest(),
    ),
)
# 先核验远端 TD，再登记到平台；结果登记是必做步骤。
registered = client.register_data(output_metadata)
verified = client.stub.GetData(
    service_pb2.GetDataRequest(data_id=registered.id), timeout=30,
).data
assert any(s.provider == upload.provider and s.bucket == upload.bucket
           and s.object_key == upload.object_key for s in verified.storages)
scheme = "tos" if upload.provider == metadata_pb2.STORAGE_PROVIDER_TOS else "oss"
print(registered.id, f"{scheme}://{upload.bucket}/{upload.object_key}")
with client.open_td(verified, provider=upload.provider, bucket=upload.bucket) as reader:
    print(reader.summary())
```

- 返回登记后的数据 ID、持久化地址和处理统计；使用 GetData 取得结果元数据，再通过 open_td 读取。
- register_data 不修改已登记数据，重复 ID 先查询核对。

## 10. 异常与能力范围

- RPC 错误按 grpc.RpcError 的 code／details 排查；不要把网络超时判定成权限不足，不能将查询失败当作空结果。
- Job 使用 Processing Service API；文件上传使用存储客户端或签名 URL。SDK 便捷方法按本次核对版本的文档使用，本文示例不代表完整 API 清单。
- ManagementClient 必须保持打开直到读取／传输完成；临时凭据由 SDK 管理，SDK 不会自动重试所有业务 RPC。
