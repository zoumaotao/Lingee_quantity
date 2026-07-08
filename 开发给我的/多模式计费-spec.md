# 多模式计费系统架构设计规范 (Spec)

## 1. 概述

本规范旨在定义一套面向 Agent 场景、支持多种服务（LLM、WebSearch、OCR、图像生成、WPS等）的高扩展性、可配置化多维计费引擎架构。核心理念是**计量与计费解耦**，通过统一数据结构和表达式引擎实现计费规则的动态配置。

## 2. 核心架构原则

1. **计费单元统一：** 全局使用唯一的计费单元 `Credit`（点数）。
2. **配置化驱动：** 计费规则抽象为表达式公式，存储于数据库，由规则引擎动态解析，避免硬编码。
3. **单表+扩展列架构：** 计量数据和计费流水均采用单表存储，核心字段物理化，多维变动参数使用 JSON 字段承载。

## 3. 存储层设计

### 3.1 统一计量事件表 (t_meter_tenant_user_other_metering)

记录所有网关或服务上报的原始物理消耗事实，不涉及费率换算。除大模型之外, 所有的计量事件统一放在这里

- **表名设计理念：** 单表结构，容纳所有业务线的计量事件。
- **核心字段划分：**

| 列名            | 类型        | 允许为空 | 描述                                                         |
| --------------- | ----------- | -------- | ------------------------------------------------------------ |
| id              | bigint      | 否       | 主键                                                         |
| tenant_id       | bigint      | 否       | 租户ID                                                       |
| user_id         | bigint      | 否       | 用户iD                                                       |
| metering_type   | varchar(30) | 否       | 计费类型: 图片生成, 视频生成, ocr等                          |
| event_id        | varchar(64) | 否       | 外部ID, 幂等用                                               |
| create_time     | timestamp   | 否       | 事件时间                                                     |
| value           | BigDecimal  | 否       | 优惠值, 流水表将使用这个值进行扣减                           |
| original_value  | BigDecimal  | 否       | 原值                                                         |
| formula_version | varchar(10) | 否       | 计量公式版本号                                               |
| request_id      | varchar(64) | 是       | 请求唯一ID. 有可能和 event_id使用相同值                      |
| agent_id        | varchar(64) | 是       | 调用的agent                                                  |
| skill_id        | varchar(64) | 是       | 调用的skill                                                  |
| event_data      | json        | 否       | 计量内容, json格式. 同一个formula_version的数据提供者要保持一致 |
| metric_volume   | int         | 否       | 数量. 比如一次性生成4张图片, 这里数量就是4. 根据计费类型不同而不同 |



### 3.2 计费流水表

现有表 t_meter_tenant_user_statement



### 3.3 数据统计优化方案 [暂不实现]

针对将多维数据存入 JSON 导致统计困难（如：按租户统计视频总时长）的问题，采取以下递进方案：

1. **首选方案（MySQL 虚拟列）：** 为高频统计的 JSON 字段创建 `Virtual/Generated Column`，并建立索引，实现关系型数据库级别的高效聚合查询。
2. **终极方案（OLAP 分离）：** 当数据量达到千万级需要复杂看板时，通过 Binlog 将流水表同步至 ClickHouse 等列式数据库进行聚合分析。



### 3.4 计费公式

新启一个表: t_meter_sys_metering_formula

| 列名             | 类型          | 允许为空 | 描述                                                         |
| ---------------- | ------------- | -------- | ------------------------------------------------------------ |
| id               | bigint        | 否       | 主键                                                         |
| metering_type    | varchar(30)   | 否       | 计费类型: 图片生成, 视频生成, ocr等                          |
| metering_version | varchar(30)   | 否       | 计量版本.比如 glm5.2                                         |
| formula          | varchar(1000) | 否       | 实际的计算公式                                               |
| global_ratio     | BigDecimal    | 否       | 系数常量. 比如只按次数计的计费, 可以直接定义 global_ratio, 计算时使用: global_ratio * metric_volume |
| formula_version  | varcahr(10)   | 否       | 版本. 参考 t_meter_sys_ratio 的 ratio_version 列. 具有唯一性 |
| create_time      | timestamp     | 否       | 事件时间                                                     |
| update_time      | timestamp     | 否       | 更新时间                                                     |
| delete_time      | timestamp     | 是       | 删除时间, 用于唯一性校验                                     |



## 4. 计算层设计 (计费引擎)

### 4.1 技术选型：QLExpress

- **选型理由：** 偏向纯数据公式计算，具备良好的性能，且在阿里系有大规模落地经验，支持丰富的自定义操作符和业务方法绑定。
- *(注：前序讨论曾推荐 Aviator，根据最新决策调整为 QLExpress，但配置化公式解析的核心思想一致。)*

### 4.2 公式配置模型

将计费规则抽象为文本字符串，存储于 `t_meter_sys_metering_formula` 表中。

- **模型 A 公式示例：**

  `base * duration_factor * duration * resolution_factor * resolution`

- **模型 B 公式示例 (含阶梯逻辑)：**

  `base * duration_factor * Math.ceil(duration / 10.0) * resolution_factor * Math.ceil(resolution / 960.0)`

### 4.3 性能优化规范 (关键防坑)

1. **防精度丢失：** 引擎计算必须全程使用 `BigDecimal` 或开启对应的高精度浮点数支持选项，严禁直接使用 `double` 运算。
2. **AST 编译缓存 (绝对强制)：** 严禁在每次请求时重新编译公式字符串。系统启动或配置变更时，必须将公式预编译为 QLExpress 的执行计划对象，并缓存于内存 (如 `ConcurrentHashMap`) 中。运行时仅传递 `Context` (JSON 反序列化后的 Map) 进行求值。

## 5. 数据源接入模式

根据业务场景和服务特性，系统支持两种主要的数据接入与计算模式：

### 5.1 拉取式 (Pull) / 定时对账模式 [暂不实现]

适用于按周期计费（如存储容量）、或者第三方服务仅提供定期账单查询的场景。

- **机制：** 系统通过定时任务 (Job) 定期扫描内部数据库状态或调用第三方 API 抓取用量。
- **流程：** Job 生成统一的计量事件 -> 写入 `unified_metering_event` -> 触发后续计算流程。

### 5.2 实时计算式 (Push) / 流式处理模式

提供一个实时计算的 /backend 接口, 用于提供给外部调用, 直接生成计费. 支持单条和多条