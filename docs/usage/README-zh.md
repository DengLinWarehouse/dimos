# 概念

本页面介绍 DimOS 的基本概念。

## 目录

- [模块](/docs/usage/modules.md)：DimOS 的基本部署单元，模块以并行方式运行，本质上是 Python 类。
- [数据流](/docs/usage/sensor_streams/README.md)：模块之间的通信方式，基于发布/订阅（Pub/Sub）系统。
- [蓝图](/docs/usage/blueprints.md)：将多个模块组合在一起并定义它们之间连接关系的方式。
- [RPC](/docs/usage/blueprints.md#calling-the-methods-of-other-modules)：一个模块如何调用另一个模块的方法（参数会被序列化为类 JSON 的二进制数据）。
- [技能](/docs/usage/blueprints.md#defining-skills)：一种 RPC 函数，但可以被 AI 智能体调用（作为 AI 的工具）。
- 智能体（Agents）：具有目标、可访问数据流、并能调用技能作为工具的 AI。
