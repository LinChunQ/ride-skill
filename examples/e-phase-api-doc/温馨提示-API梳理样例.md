# 温馨提示 API 梳理样例

> 来源：`温馨提示-API对接文档.md`
>
> 本文只做接口职责和页面使用边界梳理，不修改前端业务代码。

## 一、结论

当前文档共包含 4 个接口：

| 端 | 接口 | 作用 | 是否属于移动端学员温馨提示 |
| --- | --- | --- | --- |
| 学员端 | `POST /app/greeting-tip/detail` | 查询指定培训班的温馨提示详情 | 是，唯一需要学员端页面调用的接口 |
| 班主任端 | `POST /app/greeting-tip/manage/class/list` | 查询可维护的培训班列表 | 否 |
| 班主任端 | `POST /app/greeting-tip/manage/detail` | 查询某培训班的温馨提示维护详情 | 否 |
| 班主任端 | `POST /app/greeting-tip/manage/save` | 保存培训班专属提示语和二维码 | 否 |

统一约定：所有接口使用 `POST`、JSON 请求体和 Bearer Token，响应结构为 `{ code, msg, data }`。前端项目的 `commonReq` 会自动携带 Token 并解包 `data`。

## 二、学员端接口

### 1. 查询温馨提示详情

```http
POST /app/greeting-tip/detail
```

#### 用途

学员进入某个培训班的“温馨提示”页面时，按当前选中的 `classId` 查询页面展示所需的完整数据。

#### 请求参数

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `classId` | `Long` | 是 | 当前学员选中的培训班 ID |

示例：

```json
{
  "classId": 1001
}
```

#### 响应核心字段

| 字段 | 类型 | 页面用途 |
| --- | --- | --- |
| `classId` | `Long` | 校验当前详情对应的培训班 |
| `className` | `String` | 页面标题或培训班名称 |
| `trainingCenterName` | `String` | 培训中心/实施单位文案 |
| `welcomeMessage` | `String` | 页面欢迎语，优先直接展示后端拼装结果 |
| `reportDate` | `String` | 报到日期，格式 `yyyy-MM-dd` |
| `reportTime` | `String` | 报到时间文案 |
| `reportPlace` | `String` | 报到地点 |
| `reportNoticeText` | `String` | 报到提示文案，优先直接展示后端拼装结果 |
| `headmasterName` | `String` | 班主任姓名 |
| `headmasterPhone` | `String` | 班主任联系电话 |
| `qrUrl` | `String \| null` | 温馨提示二维码，无二维码时为空 |
| `qrFileName` | `String \| null` | 二维码文件名，通常只在需要保存/追踪文件时使用 |
| `qrUuid` | `String \| null` | 上传记录 UUID，普通展示场景可不使用 |
| `finalTipContent` | `String` | 最终展示的提示语，优先使用该字段 |
| `globalConfig` | `Object \| null` | `greeting_tip` 完整配置，需要读取原始配置时使用 |

#### 推荐页面使用方式

学员端页面应按以下顺序取值：

1. 从培训班 store 读取当前培训班 ID。
2. 调用 `/app/greeting-tip/detail`，请求体只传 `{ classId }`。
3. 页面直接使用 `welcomeMessage`、`reportNoticeText`、`reportDate`、`reportTime`、`reportPlace`、`headmasterName`、`headmasterPhone`、`qrUrl`、`finalTipContent`。
4. 不要在页面内重新拼接欢迎语、报到提示或默认提示语。
5. `globalConfig.configContent` 只作为需要展示额外配置项时的补充来源，不应覆盖后端已计算好的展示字段。

#### 学员页面对应关系

当前项目中，学员端“温馨提示”入口位于班主任告警页相关快捷入口规划中；如果后续落地学员页面，应由学员角色页面调用本接口，使用培训班 store 中的当前 `classId`。当学员有多个培训班时，先在“我的”页切换培训班，再进入温馨提示页面查询对应详情。

## 三、班主任端接口

以下 3 个接口用于班主任维护培训班温馨提示，属于管理端能力，学员端不调用。

### 2. 查询可维护培训班列表

```http
POST /app/greeting-tip/manage/class/list
```

用途：班主任温馨提示维护列表。

请求体：

```json
{
  "keyWords": "新员工"
}
```

关键响应字段：`classId`、`className`、`reportDate`、`headmasterName`、`trainingStatus`、`trainingStatusName`、`reportedStudentNum`、`unReportedStudentNum`、`hasQrCode`、`hasTipContent`。

接口只返回培训班状态为 `1`（筹备）和 `2`（实施）的记录。

### 3. 查询温馨提示维护详情

```http
POST /app/greeting-tip/manage/detail
```

用途：班主任进入某培训班维护页面时查询当前提示语和二维码。

请求体：

```json
{
  "classId": 1001
}
```

关键响应字段：`classId`、`className`、`reportDate`、`reportPlace`、`headmasterName`、`headmasterPhone`、`trainingStatus`、`trainingStatusName`、`currentTipContent`、`qrFileName`、`qrUrl`、`qrUuid`、`globalConfig`。

调用方必须是该培训班班主任。

### 4. 保存温馨提示

```http
POST /app/greeting-tip/manage/save
```

用途：班主任保存培训班专属提示语和二维码关联。

请求体：

```json
{
  "classId": 1001,
  "tipContent": "请提前加入班级群，报到时携带身份证原件。",
  "qrFileName": "profile/upload/2026/07/10/qr-1001.png",
  "qrUrl": "/profile/upload/2026/07/10/qr-1001.png"
}
```

约束：

- `classId`、`tipContent` 必填。
- `tipContent` 最大 4000 字符。
- `qrFileName` 和 `qrUrl` 要么同时传，要么都不传。
- 只有当前培训班班主任可保存。
- 只有培训班状态 `1`（筹备）或 `2`（实施）允许保存。
- 当前接口没有显式删除二维码能力；不传二维码字段表示保留原二维码。

