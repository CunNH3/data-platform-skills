---
name: data-platform
description: 使用数据平台查询和处理数据、申请开发机或提交 Job；通过 PyPI 更新 current-data-py，再读取包内 Skill 和接口说明。
---

# 数据平台入口

每次使用先在任务环境通过 PyPI 安装或更新最新版 `current-data-py`，再读取包内索引：

```sh
python -m pip install --upgrade current-data-py
python -m current_data_py.docs --index
```

按索引读取本次所需的 Skill、接口信息和使用示例，再执行任务。读取包内更新记录并提示新增内容，首次只介绍最近一次更新。
