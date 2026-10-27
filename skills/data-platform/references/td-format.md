# TD 文件生成规范

本文件保存仓库 convert-robot-data Skill 的规范正文要点，随本 Skill 一起分发。执行任务直接读取本文，不需要访问仓库或安装其他 TD Skill。

## 1. Topic 和消息

- 数据按 topic 和带时间戳的消息保存，每个 topic 对应一个确定的 Protobuf 类型；不同 topic 可以复用消息类型。
- topic 固定为 `/{category}/{body_part}/{data_name}`，以 `/` 开头，恰好三段非空路径，使用小写英文 snake_case。
- topic 中不加入帧率、分辨率、单位、像素编码或硬件型号，这些信息放入消息或元数据。

| category | 使用范围 |
|---|---|
| sensors | 外部接口直接读取的传感器数据，如图像、IMU、单独编码器读数 |
| commands | 发给机器人的控制目标或动作请求；记录命令不代表已执行 |
| states | 非直接传感器输出的机器人状态，包括聚合状态、计算或融合结果 |

- body_part 使用完整名称，如 head、whole_body、upper_body、lower_body、left_arm、right_arm、left_wrist、right_wrist；左右以机器人自身为准。
- data_name 按实际含义区分，如 color、color_left、color_right、infrared、imu、joints、pose、pose_rpy；只创建真实存在的数据流。
- 相机左右眼与机器人左右部位分别命名，例如 `/sensors/left_wrist/color_right`；不能为同一条流生成多个无差别别名。

## 2. 消息类型

| 数据含义 | 使用的 schema |
|---|---|
| 视频图像 | foxglove.CompressedVideo |
| 音频 | foxglove.CompressedAudio |
| IMU | sensor_msgs.Imu |
| 聚合关节状态、夹爪开度 | sensor_msgs.JointState |
| 四元数位姿 pose | geometry_msgs.PoseStamped |
| 欧拉角位姿 pose_rpy | custom_msgs.PoseRPY |
| 关节或夹爪电流 | custom_msgs.JointCurrent |

- 先检查已安装 SDK 的 `SCHEMA_TO_CLASS` 和对应消息的 `DESCRIPTOR`，确认实际字段、嵌套结构及版本；类型名称相似不代表字段相同。
- pose 使用 PoseStamped：位置和四元数在 pose.position、pose.orientation 下；header.stamp 与 TD 消息时间戳一致，header.frame_id 使用 base。
- base 是平台显示约定，不代表坐标已标定到同一物理参考系，也不会执行坐标变换。PoseRPY 保留自身字段，不套 PoseStamped 包装。
- 仅重命名 topic 不等于转换消息；必须解码源格式并写入目标 schema。SDK 缺少所需类型时由平台补充并发布，不能自行猜字段。

## 3. 时间戳与数据语义

- TD 写入时间戳使用纳秒；明确源数据的单位、时钟和时间基准，再转换到任务使用的时间轴。
- 派生数据保留对应源时间，不用处理时间替换采集时间；字段包含秒／纳秒时间戳时保持与 TD 时间一致。
- 单位、坐标、关节顺序、像素编码和左右眼映射必须有明确依据；缺失或不明确的内容不能靠名称猜测。
- 抽帧、重采样等操作保留转换规则和输入／输出数量，解释预期的数据减少；不静默丢记录或伪造缺失通道。

## 4. 视频要求

- 头部和腕部相机短边为 720 像素，保持比例缩放，不拉伸、不裁剪。
- H.264 编码参数为 `-preset medium -crf 18 -pix_fmt yuv420p -bf 0 -g 50`。
- 复用已有编码帧前检查是否符合要求，不符合时转码；不能将 JPEG 或任意图像字节标成 CompressedVideo。
- 按实际 SDK 的视频帧／packet 协议写入，保证参数集和解码依赖完整，输出样本必须能连续解码。

## 5. 分组、分块和压缩

| 配置 | 规范 |
|---|---|
| 视频 TopicGroup | 每组一个视频 topic，按 100 条消息阈值分块 |
| 非视频 TopicGroup | 启用 TD 压缩，按 1 MB 大小阈值分块 |
| 数据分组 | 视频、Pose、关节角、IMU、一次性标定／元数据和其他消息分别分组 |
| Pose 类型 | PoseStamped 与 PoseRPY 分组存储 |

- 只生成实际存在的分组。当前写入示例用 `size=1_000_000` 表示 1 MB，视频用 `count=100`，记录实际 SDK 的阈值和关键帧边界行为。
- 写入接口是 `writer.open_topics(..., compression=..., video=..., chunk_config=...)`；每组写完关闭 group，整个 writer 关闭后才上传文件。

## 6. 输出校验和登记

1. 重新读取生成的 TD，核对 topic、实际 schema、分组和压缩配置，确认能解码代表性样本。
2. 核对消息数、时间范围、时间戳映射及业务处理规则；含视频时检查各路分辨率、编码和可解码性。
3. 上传 TD 后核验远端数据，使用同一输出数据 ID 调用 RegisterData，登记实际存储位置、文件大小、校验和和可用的内容统计。
4. 用 GetData 核对登记记录，通过 DataClient 读取结果；返回结果数据 ID、持久化地址和处理统计。

- 输入采集设备、topic 数量、帧率、时长等可能在处理后改变，元数据按输出实际内容生成。
- 有未解决的必需数据流、解码失败或登记失败时说明具体问题，不能仅凭 Job 退出码为 0 宣称完整成功。

## 7. SDK 字段检查

```python
from current_data_py.schema import SCHEMA_TO_CLASS

schema = SCHEMA_TO_CLASS["geometry_msgs.PoseStamped"]
for field in schema.DESCRIPTOR.fields:
    print(field.name, field.type, field.message_type.full_name if field.message_type else "")
```

实际 TD 读写和上传登记代码见同目录的 DataClient 使用文档；本文负责数据规范，字段以任务实际安装的 SDK 为准。
