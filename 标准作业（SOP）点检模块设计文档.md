# 标准作业（SOP）点检模块设计文档

## 1. 文档说明

本文面向交接使用，以当前代码实现为准，说明标准作业（SOP）点检模块的职责边界、核心数据表、底表维护、任务生成、移动端保存/提交、待办通知、自动关闭、驱动任务统计、导出和排查注意事项。

当前代码中 SOP 使用 `sop` 作为 taskflow 类型标识，对应移动端地址 `https://mas.minthgroup.com/m/#/audit/sop/`。SOP 与 DDS、5S、设备、工艺点检共用 taskflow 的任务组、审批节点、转派、关闭和通知能力，但业务上不走主管审核流程，移动端提交后直接完成。

## 2. 模块边界和代码入口

| 职责 | 主要代码 | 说明 |
| --- | --- | --- |
| 定时任务入口 | `dap-biz/src/main/java/com/minthgroup/ees/dap/controller/taskflow/TimingTaskFlowController.java` | 触发 SOP 任务生成、人员计数初始化、自动关闭、结束前提醒 |
| 底表接口 | `dap-biz/src/main/java/com/minthgroup/ees/dap/controller/taskflow/BaseSopController.java` | SOP 底表分页、新增、修改、删除、导入、导出、分类查询 |
| 任务接口 | `dap-biz/src/main/java/com/minthgroup/ees/dap/controller/taskflow/TaskSopAuditController.java` | SOP 任务分页、详情列表、保存、提交、删除、导出、历史字段修复 |
| 底表服务 | `dap-biz/src/main/java/com/minthgroup/ees/dap/service/impl/taskflow/BaseSopServiceImpl.java` | 底表版本、启停、分类查询、导入批量保存 |
| 任务服务 | `dap-biz/src/main/java/com/minthgroup/ees/dap/service/impl/taskflow/TaskSopAuditServiceImpl.java` | SOP 任务生成、观察对象分配、保存、提交、扣分统计、非标结果查询、导出 |
| 底表导入监听器 | `dap-biz/src/main/java/com/minthgroup/ees/dap/listener/SopImportListener.java` | Excel 合并单元格解析、按人员类别聚合底表、版本号计算 |
| 驱动任务统计 | `dap-biz/src/main/java/com/minthgroup/ees/dap/handler/driveTask/SopDriveTaskStatistic.java` | 汇总应完成、已完成、准时完成 |

## 3. 核心数据模型

### 3.1 SOP 底表 `c_taskflow_base_sop`

实体：`BaseSopEntity`

| 字段 | 说明 |
| --- | --- |
| `site` / `site_name` | 工厂代码和工厂名称 |
| `category` | 人员类别枚举 `SopCategoryEnum`，代码包括 `C_01` 配送人员、`C_02` 检查人员、`C_03` 生产检测人员、`C_04` 生产作业人员、`C_05` 成品出货 |
| `category_desc` | 人员类别中文描述 |
| `subform` | SOP 稽核项列表，类型为 `List<SopSubformEntity>`，使用 JSON typeHandler 保存 |
| `version` | 版本号 |
| `enable` | 是否启用，`1` 启用，`0` 禁用 |

`SopSubformEntity` 主要字段：

| 字段 | 说明 |
| --- | --- |
| `category` | 稽核分类 |
| `categoryDesc` | 稽核分类描述或编码，取决于维护入口，详见维护注意事项 |
| `item` | 稽核项目 |
| `score` | 标准分值，提交时 `result=NG` 才累加扣分 |
| `needPhotograph` | 是否需要拍照，默认 `N` |
| `referImg` | 参考图片 |
| `image` | 移动端上传的子项图片 |
| `result` | 结果枚举 `ResultEnum`，主要按 `NG` 计算扣分 |
| `ngInfolist` | NG 异常信息 |

### 3.2 SOP 任务主表 `c_yida_sop_audit`

实体：`TaskSopAuditEntity`

该表沿用宜搭标准作业观察表结构，EES 生成的数据通过 `data_source='02'` 区分。

| 字段 | 说明 |
| --- | --- |
| `sop_audit_id` | 任务主键 |
| `uuid` | 任务组 UUID，同一次生成的多条观察任务共用 |
| `dateField_m4jlurzm` | 系统推送时间，代码按 UTC 写入 |
| `employeeField_m4jlurzn` / `textField_m4nmvt2z` | 观察人姓名 / 工号 |
| `textField_m4nmvt3b` / `textField_m4ul4ekc` | 工厂名称 / 工厂代码 |
| `department` / `textField_m56e3vjq` | 观察人部门编号 / 名称 |
| `employeeField_m4jmgkmv` / `textField_m4nmvt30` | 作业人员姓名 / 工号 |
| `observe_department` / `textField_m4jmgkn0` | 作业人员所在部门或课室编号 / 名称 |
| `shift` / `selectField_m4v5p5cs` | 班次代码 / 描述 |
| `dateField_m4tlkqds` | 班次日期，也是统计日期 |
| `tableField_m56cxezc` | 移动端填写的 SOP 稽核项列表 |
| `numberField_m528nh72` | 累计扣分分值 |
| `instance_status` | 状态：`CREATED`、`SAVE`、`COMPLETED`、`CLOSED` 等 |
| `data_source` | 数据来源，`01` 宜搭，`02` EES |
| `time_out_status` | 超时状态，统计准时完成时使用 |

### 3.3 通用 taskflow 表

| 表/实体 | 作用 |
| --- | --- |
| `TaskGroupEntity` | 待办任务组。SOP 使用 `checkType=sop`，点检待办 `type=0` |
| `ApprovedEntity` | 流程节点。生成时写创建人节点和观察人节点，提交时追加 OK 节点 |
| `BaseUserEntity` | SOP 观察人配置。任务生成按 `type=sop` 和 `frequency` 读取人员 |
| `BaseExcludeDateConfigEntity` | 人员排除日期配置。任务生成通过 `isExecute(site, cycleDate, userId, sop)` 判断是否执行 |
| `UserCountEntity` | 被观察人计数表 `c_taskflow_base_user_count`。用于选择本月被观察次数较少的作业人员 |

## 4. 枚举和配置

| 枚举/配置 | 值 | 说明 |
| --- | --- | --- |
| `TypeEnum.SOP` | `sop` | SOP 业务类型 |
| `TypeNoticeUrlEnum.SOP` | `https://mas.minthgroup.com/m/#/audit/sop/` | 移动端任务地址 |
| `DriveTaskTypeEnum.T_SOP` | `sop` | 驱动任务统计类型 |
| `JobLevelEnum.L20` | `20` / 课长 | 新版 SOP 生成入口支持的岗位层级 |
| `JobLevelEnum.L30` | `30` / 经理 | 新版 SOP 生成入口支持的岗位层级 |
| `JobLevelEnum.L40` | `40` / 工厂总 | 新版 SOP 生成入口支持的岗位层级 |
| `FrequencyEnum.F01` | 一周一次 | L30 经理配置读取 |
| `FrequencyEnum.F02` | 一班一次 | 老版 `createTaskV1` 按课室读取 |
| `FrequencyEnum.F03` | 一日一次 | L20 课长配置读取 |
| `FrequencyEnum.F04` | 两周一次 | `UserCountService` 计数分配中参与计算 |

## 5. 底表维护流程

### 5.1 新增

入口：`POST /base/sop`

流程：

1. Controller 设置 `version=1`。
2. 按 `base.category.code` 从 `SopCategoryEnum` 回填 `categoryDesc`。
3. 遍历 `subform`，按子项 `category` 从 `CheckCategoryEnum` 回填 `categoryDesc`。
4. 直接调用 `service.save(base)` 保存。

新增接口当前没有做同工厂、同人员类别的启用版本去重。

### 5.2 修改

入口：`PUT /base/sop`

流程：

1. Controller 回填人员类别和稽核分类描述。
2. `BaseSopServiceImpl#updateSop` 查询旧记录。
3. 将旧记录 `enable` 置为 `0`。
4. 清空新对象的 `id/create/update` 相关字段。
5. 新记录 `version=旧版本+1`，`enable=1`，重新插入。

修改不是原地覆盖，已创建任务中的 `tablefieldM56cxezc` 不会被新底表影响。

### 5.3 删除

入口：`DELETE /base/sop`

流程：

1. 删除指定 id。
2. 如果删除的是启用版本，则按相同 `site + category` 查询版本号最大的旧记录。
3. 找到旧记录时，将旧记录 `enable` 改回 `1`。

### 5.4 导入导出

导入入口：`POST /base/sop/import?site=...`

导入逻辑：

1. 使用 `EasyExcel` 读取 `ImportSopDto`，并读取合并单元格信息。
2. 行内 `item`、人员类别、稽核分类都为空时跳过。
3. 对 Excel 第 1、2 列的合并单元格做向下填充。
4. 按人员类别 `categoryDesc` 聚合成一条 `BaseSopEntity`，每行变成一个 `SopSubformEntity`。
5. 按 `site + categoryDesc` 查询最新版本，新版本号为最新版本 `+1`。
6. `saveConfigBatch` 会先停用同 `site + categoryDesc` 下当前启用记录，再保存新版本并置 `enable=1`。

导出入口：`POST /base/sop/export`

导出会将底表每个 `subform` 拆成一行，输出人员类别、稽核分类、稽核项目、分值、是否拍照、启用状态。

## 6. 任务生成流程

SOP 当前有两个定时入口：

| 入口 | 行为 |
| --- | --- |
| `GET /timing/taskflow/sop` | 老入口，遍历站点课室，调用 `createTaskV1`，按 `FrequencyEnum.F02` 读取人员 |
| `GET /timing/taskflow/sop/{jobLevel}` | 新入口，按岗位层级调用 `createTask` |

两个入口都会构造 Redis 锁 key，使用 `SafeTaskSubmitter.submitOnce` 异步执行，并通过 `RedisUtils.getLock(key, bbuSites, 900)` 防止同一批次重复执行。支持站点限定为 `2871`、`2081`、`2071`、`2881`、`2951`、`3271`。

### 6.1 老入口 `createTaskV1`

主流程：

1. 按站点查询四级部门列表：`syncEmpMapper.getDeptid4ListBySite(site)`。
2. 对每个课室调用 `dealByDeptId`。
3. `dealByDeptId` 分页读取 `type=sop + classroom + frequency=F02` 的基础人员配置。
4. 将基础人员工号转为主岗员工信息。
5. 查询当天考勤，过滤未上班和请假人员。
6. 通过排除日期配置过滤不执行人员。
7. 按人员当天班次逐个调用 `createTaskByJob`。
8. 同一观察人、同一班次、同一天已存在任务时不重复创建。
9. `buildInstance` 从 `UserCountService#getMinCountUserByShift` 选择被观察作业人员并创建任务。

### 6.2 新入口 `createTask(site, cycleDate, jobLevel, ...)`

`L20` 课长：

1. 读取 `type=sop + frequency=F03` 的基础人员。
2. 转为员工信息后查询当天考勤。
3. 过滤未上班、请假、排除日期人员。
4. 调用 `createTaskByJobV1(cycleDate, syncEmpDto, ...)`。
5. 该方法同一天同一观察人已存在任务时不再生成，并要求当天能查到班次。

`L30` 经理：

1. 读取 `type=sop + frequency=F01` 的基础人员。
2. 按工号去重，保留 `approveLvl` 比较后的员工记录。
3. 过滤排除日期、无考勤、无排班、请假人员。
4. 调用 `createTaskByJobV1(JobLevelEnum.L30, ..., weeks=1)`。
5. 去重范围是本周周一到周日内同一观察人已存在任务则不再生成。

`L40` 工厂总：

1. 按岗位名 `工厂高级经理` 查询人员。
2. 将岗位名改为 `工厂总`，固定 `deptid4=10100232`。
3. 过滤排除日期。
4. 调用 `createTaskByJobV1(JobLevelEnum.L40, ..., weeks=2)`。
5. 计划完成时间按 `pushTime + 14 天` 的当天最大时间计算。

### 6.3 任务实例构建

主要方法：

| 方法 | 用途 |
| --- | --- |
| `buildInstance(String shift, LocalDateTime cycleLocalDateTime, SyncEmpDto syncEmpDto, ...)` | 老入口和 L20 使用，按班次创建 |
| `buildInstance(SyncEmpDto syncEmpDto, LocalDateTime startOfDay, boolean isNotice, int weeks)` | L30/L40 使用，默认白班 `S1202`，每次最多创建 3 条 |

创建内容：

1. 生成一个 `uuid`，同一批观察任务共用。
2. 通过 `UserCountService` 选择被观察对象，优先选择本月计数低的人员。
3. 创建 `TaskSopAuditEntity`，状态为 `CREATED`，`dataSource=02`。
4. 填入观察人、作业人员、工厂、部门、课室、班次、班次日期等字段。
5. 更新被观察人的 `UserCountEntity.count`。
6. 为每条任务写入 `ApprovedEntity` 创建人节点和观察人节点。
7. 创建一条 `TaskGroupEntity`，`checkType=sop`，任务名取 `Check_Name_sop`。
8. 发送钉钉待办和 EES 个人待办。

计划完成时间：

| 场景 | 计划完成时间 |
| --- | --- |
| 老入口 / L20 | `pushTime + 24 小时` |
| L30 | `pushTime + 7 天` 当天 `23:59:59.999` |
| L40 | `pushTime + 14 天` 当天 `23:59:59.999` |

## 7. 移动端查询、保存和提交

### 7.1 查询任务

入口：`GET /taskflow/sop/list/{uuid}`

返回同一 `uuid` 下 `data_source=02` 的任务列表，按 `sopAuditId` 倒序。返回时会：

1. 从 `TaskGroupEntity` 补充任务名称和发起人描述。
2. 将 `tablefieldM56cxezc` 赋给 VO 的 `formData`。
3. 对观察人部门名称做站点语言渲染。

### 7.2 保存

入口：`POST /taskflow/sop/save`

请求体：`List<TaskSopAuditDto>`

流程：

1. 校验 `sopAuditId` 必填、任务存在。
2. 如果作业人员发生更换，则调用 `userCountService.userCount(month, sop, newUserId)` 增加新作业人员计数。
3. 将 DTO 拷贝到实体。
4. 如果实体 `tablefieldM56cxezc` 为空，则使用 DTO 的 `formData`。
5. 遍历 `formData`，只统计 `result=NG` 且 `score` 非空的分值，写入 `numberfieldM528nh72`。
6. 状态置为 `SAVE` 并更新。

### 7.3 提交

入口：`POST /taskflow/sop/submit`

流程与保存基本一致，但提交会：

1. 拒绝已经 `CLOSED` 的任务。
2. 重新计算累计扣分。
3. 将任务状态直接置为 `COMPLETED`。
4. 追加一条 `ApprovedEntity`，`userName=OK`、`type=1`、`progress=3`。
5. 调用 `taskGroupService.updateStatus(uuid, "1")` 完成任务组。
6. 调用 `sendNoticeService.sendDemandProgressByUuid(uuid)` 刷新 EES 待办进度。

当前 SOP 没有独立主管审核接口，`TaskSopAuditController` 也没有 `/audit`。NG 项只影响累计扣分和非标查询，不会像设备点检那样生成异常处理任务。

## 8. 转派、关闭和提醒

### 8.1 转派

通用转派入口在 `TaskGroupServiceImpl#reassign`。当 `checkType=sop` 时进入 `reassignSop`：

1. 按 `uuid` 查询 SOP 任务，找不到则报“点检任务已被撤销”。
2. 仅处理任务组 `type=0` 的点检待办，调用通用 `reassignCheck`。
3. 使用 `TypeNoticeUrlEnum.SOP` 构造移动端地址并发送钉钉通知。

SOP 当前没有审核待办转派分支。

### 8.2 自动关闭

入口：`GET /timing/taskflow/auto/close`

通用自动关闭会调用 `autoAuditNoticeService.autoClosed(TypeEnum.SOP)`。真正关闭任务组时，`TaskGroupServiceImpl#closeSop` 会把同 `uuid` 下所有 SOP 任务 `instanceStatus` 置为 `CLOSED`。

注意：`closeSop` 没有限定原状态为 `CREATED` 或 `SAVE`，与设备、工艺点检的关闭实现不同。交接排查时要关注是否可能关闭已保存或已完成数据。

### 8.3 结束前提醒

入口：`GET /timing/taskflow/notifyBeforeShutdown`

通用提醒中会调用 `autoAuditNoticeService.notifyBeforeShutdown(site, TypeEnum.SOP)`。

## 9. 统计、导出和非标结果

### 9.1 驱动任务统计

统计类：`SopDriveTaskStatistic`

当前统计不是直接使用 `TaskSopAuditMapper` 的 `listTaskCount/listTaskCompletedCount/listOnTimeTaskCompletedCount` 三个聚合 SQL，而是调用 `listTask(queryCycleDate)` 拉取任务后按观察人聚合：

| 指标 | 计算方式 |
| --- | --- |
| 应完成数 | 同一观察人任务总数 |
| 已完成数 | `instance_status=COMPLETED` 的数量 |
| 准时完成数 | 直接等于已完成数，代码注释为“不存在超时完成的情况” |

统计维度会尝试通过 `getUser(site, userId)` 回填部门和课室，缺失时设置为 `TEMP0001`。

### 9.2 任务导出

入口：`POST /taskflow/sop/export`

导出两个 sheet：

1. “点检”：任务主数据。
2. “明细”：从 `tablefieldM56cxezc` 拆出稽核项明细。

导出时会把 `instanceStatus` 转中文描述，并将 `datefieldM4jlurzm` 按 `local.time-zone` 加小时偏移。

### 9.3 非标扣分查询

服务方法：`TaskSopAuditServiceImpl#listByUserId(cycleDate, userId)`

该方法按班次日期和作业人员工号查询当天任务，条件包括：

1. `datefieldM4tlkqds` 在当天范围内。
2. `textfieldM4nmvt30 = userId`。
3. `numberfieldM528nh72 > 0`。

返回时把扣分转成负数，并只输出 `result=NG` 的子项，`standardName` 固定为“标准作业”，`approvedResult` 固定为 `OK`。

## 10. 常用接口清单

| 接口 | 方法 | 说明 |
| --- | --- | --- |
| `/base/sop/page` | POST | SOP 底表分页 |
| `/base/sop/category` | GET | 查询启用人员类别 |
| `/base/sop/relList` | GET | 查询底表关联子表单 |
| `/base/sop` | POST | 新增底表 |
| `/base/sop` | PUT | 修改底表，新建版本 |
| `/base/sop` | DELETE | 删除底表，必要时恢复旧版本 |
| `/base/sop/import` | POST | 导入底表 |
| `/base/sop/import/template` | GET | 下载导入模板 |
| `/base/sop/export` | POST | 导出底表 |
| `/timing/taskflow/sop` | GET | 老版 SOP 任务生成 |
| `/timing/taskflow/sop/{jobLevel}` | GET | 按岗位层级生成 SOP 任务 |
| `/taskflow/sop/page` | POST | SOP 任务分页 |
| `/taskflow/sop/list/{uuid}` | GET | 移动端查询同组任务 |
| `/taskflow/sop/save` | POST | 移动端保存 |
| `/taskflow/sop/submit` | POST | 移动端提交并完成 |
| `/taskflow/sop/getTaskFlowBaseSopInfo` | GET | 按工厂和人员类别查询启用底表 |
| `/taskflow/sop/export` | POST | 导出任务和明细 |
| `/taskflow/sop/updateClassroom` | GET | 历史数据回填作业人课室编号 |
| `/taskflow/sop/updateSiteName` | GET | 历史数据修正站点名称 |

## 11. 关键排查路径

### 11.1 定时触发后没有生成任务

按顺序检查：

1. 调用的入口是否正确：老入口 `/timing/taskflow/sop`，新入口 `/timing/taskflow/sop/{jobLevel}`。
2. `bbuSites` 是否包含允许站点：`2871/2081/2071/2881/2951/3271`。
3. Redis 锁 key 是否已有任务执行中。
4. `c_taskflow_base_user` 是否有 `type=sop` 且频次匹配的观察人配置。
5. 观察人工号是否能通过 `SyncEmpMapper` 查到员工信息。
6. 当天是否有考勤或班次，是否请假。
7. `c_taskflow_base_exclude_date_config` 是否配置了不执行。
8. 同观察人、同日期、同班次或同周是否已存在任务。
9. `c_taskflow_base_user_count` 是否有可被观察人员，且班次过滤后非空。

### 11.2 任务生成但被观察人为空或数量少

重点检查：

1. `c_taskflow_base_user_count` 当月是否已初始化。
2. 被观察人员所在课室是否与观察人的课室匹配。
3. `UserCountService#getMinCountUserByShift` 是否因班次不一致过滤掉人员。
4. L30/L40 使用的 `buildInstance` 每次最多创建 3 条任务。
5. L30 按 `manager` 选择被观察人，需确认 `creatUserCount` 后是否执行过经理归属回填。

### 11.3 移动端提交后扣分不正确

重点检查：

1. 前端提交的 `formData` 是否传入，或是否写在 `tablefieldM56cxezc`。
2. 子项 `result` 是否为枚举 `NG`。
3. 子项 `score` 是否为可解析整数。当前代码使用 `Integer.parseInt`。
4. 作业人员是否被更换，更换会增加新作业人员的计数。

### 11.4 已提交任务仍显示待办

重点检查：

1. `c_yida_sop_audit.instance_status` 是否已为 `COMPLETED`。
2. 同一 `uuid` 下是否存在多条任务，提交接口会按入参列表逐条处理。
3. `TaskGroupEntity` 是否已被 `taskGroupService.updateStatus(uuid, "1")` 更新。
4. `sendNoticeService.sendDemandProgressByUuid(uuid)` 是否异常。

## 12. 维护注意事项

1. SOP 任务生成不直接依赖 SOP 底表；底表主要供移动端按人员类别查询稽核项。任务生成依赖观察人配置、考勤/班次、排除日期和被观察人计数表。
2. SOP 提交即完成，没有主管审核和异常处理任务生成逻辑。
3. `c_yida_sop_audit` 是实际任务表，EES 任务用 `data_source=02` 过滤，不要只按表名判断为宜搭数据。
4. 底表导入时，`SopImportListener` 中 `subItem.setCategory(item.getCategoryDesc1())`、`subItem.setCategoryDesc(CheckCategoryEnum.getCodeByName(...))`，与新增/修改接口对子项 `category/categoryDesc` 的语义方向不完全一致。
5. `BaseSopController#save` 没有对同站点同人员类别启用底表做去重，导入和修改会停用旧版本。
6. `BaseSopServiceImpl#removeBase` 删除启用版本后按 `site + category` 恢复版本号最大的旧记录，而导入版本查询按 `site + categoryDesc`。
7. `TaskGroupServiceImpl#closeSop` 会关闭同 `uuid` 下所有 SOP 任务，没有状态条件。
8. `TaskSopAuditMapper#listNoSend` SQL 中引用了 `c_yida_dds` 和 `dds_id/dds_uuid`，看起来是从 DDS 复制遗留，当前 SOP 主流程未使用该方法。
9. 老入口 `createTaskV1` 和新入口 `createTask` 同时存在，排查重复或缺失任务时要先确认实际调度调用的是哪一个。
10. `pushTime` 使用 UTC，页面和导出会按 `local.time-zone` 做偏移显示。
