# 测试环境

用户明确选择 test 时使用本页，未指定时默认使用生产环境。测试阶段不限制管理员身份，平台原有认证和接口权限保持不变。

## 接口与操作

- 测试站点：`https://data-test.current-robotics.work`。接口方法、请求格式和参数沿用 [平台接口说明](api.md)，认证使用测试环境已有的正式方式。
- 数据查询、读取、上传登记、开发机申请与释放、Job 提交及日志查询全部使用 test；生产请求失败不自动改到 test，测试请求失败也不自动改到生产。
- 开发机资源确认、资源检查和释放确认仍按 [开发机使用](dev-pod.md) 执行；Job 按 [任务提交与跟踪](jobs.md) 执行。
- SSH 检查使用目标测试环境中已有的有效 Pod IP；没有 IP 时待目标开发机创建后验证，不额外创建探测 Pod。

## DataClient

在独立进程中、首次导入 SDK 之前设置 `ENV=test`，不要复用生产客户端：

```python
import os

os.environ["ENV"] = "test"
from current_data_py.remote import DataClient

with DataClient() as client:
    pass
```

Job 中核对平台注入的 `ENV=test`；脚本不要覆盖为其他环境。其余 SDK 读写与上传登记用法见 [DataClient 使用](data-client.md)。
