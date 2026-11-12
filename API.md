# 云育才（yunyucai.cn）学生端接口整理

> 逆向自 `https://yunyucai.cn/stu2/` 学生端前端（kkl-online-school / estudy 体系）。
> 文档中的响应字段均为 2026-11-09 实际调用所得，非推测。
> 仅用于个人学习数据查看，接口归属杭州育才中学 / 开课啦（kaike.la）。

---

## 1. 概览

| 项目 | 值 |
|---|---|
| 网关地址 | `https://yunyucai.cn/api.do?<serviceName>` |
| 请求方法 | `POST` |
| Content-Type | `application/json` |
| 鉴权方式 | 自定义头位 token，放在 **body 的 `token` 字段**（也接受 query 参数） |
| 响应格式 | JSON，业务状态看 `code`，`"1000001"` 为成功 |
| 学生端入口 | `https://yunyucai.cn/stu2/` |
| 教师/管理端 | `https://yunyucai.cn/` |
| 前端静态资源 | `https://static.estudy.cn/kkl/web/online-school/stu2/` |
| 应用标识 `app` | `xhxyc`（育才）；另有 `xhxcs`（未来校）等 |
| 网关 `client_id` | `6` |

### CORS（关键）

服务端会**回显任意 Origin**，连 `Origin: null`（`file://` 页面）都放行：

```
access-control-allow-origin: <请求的 Origin>
access-control-allow-methods: POST,GET,OPTIONS
access-control-allow-headers: Accept,Accept-Language,Content-Language,Content-Type
access-control-allow-credentials: true
```

这意味着**纯前端页面（本地 html / Cloudflare Pages / 任意静态站）可以直接调用，无需自建代理**。

---

## 2. 请求包络

所有接口共用同一个信封结构：

```json
{
  "param": "{\"qry\":{\"startDate\":1790000000000,\"endDate\":1794000000000}}",
  "service_name": "yucai.WebMemberWorkQueryFacade.queryMemberWorkByDate",
  "strategy": "dubbo",
  "client_id": "6",
  "timestamp": 1794146693000,
  "token": "<登录后获得，登录类接口不需要>"
}
```

| 字段 | 类型 | 说明 |
|---|---|---|
| `param` | string | **业务参数，必须序列化成 JSON 字符串**（双层编码，极易踩坑） |
| `service_name` | string | 服务名，同时也要出现在 URL query 上（`?服务名`） |
| `strategy` | string | 固定 `"dubbo"` |
| `client_id` | string | 固定 `"6"` |
| `timestamp` | number | 毫秒时间戳 |
| `token` | string | 登录令牌，登录类接口可省略 |

## 3. 响应包络

```json
{
  "code": "1000001",
  "data": {
    "nCode": 1000001,
    "resultObject": { },
    "resultList": [ ]
  },
  "message": "操作成功",
  "serResult": true,
  "sign": "DCFC07B1F6F98159A34E36ACE2FD709A",
  "success": true
}
```

- 单对象结果 → `data.resultObject`
- 列表结果 → `data.resultList`
- `success` / `serResult` 为 `false` 时，看 `message` 判断原因
- `sign` 是响应签名，客户端不需要校验

---

## 4. 登录流程（两步）

```
① checkAndQueryUserByWeb  手机号 + md5(密码)
        ↓ 返回 resultList[]（含 memberId、loginCode、姓名、班级、学校）
② loginWebByCode          memberId + loginCode
        ↓ 返回 resultObject.token
③ 后续所有接口 body 里带上 token
```

### 4.1 校验账号密码

`aischool.LoginCmdFacade.checkAndQueryUserByWeb`

```json
{
  "service_name": "aischool.LoginCmdFacade.checkAndQueryUserByWeb",
  "param": "{\"cmd\":{\"loginName\":\"1xxxxxxxxxx\",\"password\":\"<md5(明文密码)>\",\"app\":\"xhxyc\",\"clientType\":\"web\",\"sourceType\":\"web\",\"userAgent\":\"Mozilla/5.0\"}}",
  "strategy": "dubbo",
  "client_id": "6",
  "timestamp": 1794146693000
}
```

| 参数 | 类型 | 说明 |
|---|---|---|
| `cmd.loginName` | string | 手机号 / 学号 |
| `cmd.password` | string | **密码的 MD5 小写 hex**，不是明文，也不是加盐 |
| `cmd.app` | string | `xhxyc` |
| `cmd.clientType` / `sourceType` | string | `web` |
| `cmd.userAgent` | string | 任意 UA 字符串 |

响应 `data.resultList[0]`：

| 字段 | 示例 | 说明 |
|---|---|---|
| `memberId` | `202507040220586404` | 学员 ID，后续必用 |
| `loginCode` | `d8c57c94-bf7c-4218-a6df-ccb349...` | 一次性登录码，**有时效，需立即换取 token** |
| `firstName` | `孔令泽` | 姓名 |
| `className` | `八年级10班` | 班级 |
| `schoolName` | `杭州育才中学` | 学校 |
| `memType` | `1` | 1=学生，其他为教师/家长 |
| `loginTime` | `1794146693000` | 上次登录时间 |
| `bindWechatStatus` | `0` | 是否绑定微信 |
| `registerSort` | `1751615281000` | 注册时间 |

### 4.2 换取 token

`aischool.LoginCmdFacade.loginWebByCode`

```json
{
  "service_name": "aischool.LoginCmdFacade.loginWebByCode",
  "param": "{\"cmd\":{\"memberId\":\"202507040220586404\",\"loginCode\":\"d8c57c94-...\",\"app\":\"xhxyc\",\"clientType\":\"web\",\"sourceType\":\"web\",\"userAgent\":\"Mozilla/5.0\"}}"
}
```

响应 `data.resultObject`：`token`（字符串）、`memberId`、`memType`、`class`

> token 实测有效期较长，工具里存 sessionStorage；失效后重走 4.1~4.2 即可。

---

## 5. 接口清单

| # | 服务名 | 用途 | 需 token |
|---|---|---|---|
| 1 | `aischool.LoginCmdFacade.checkAndQueryUserByWeb` | 校验账号密码 | 否 |
| 2 | `aischool.LoginCmdFacade.loginWebByCode` | 换取 token | 否 |
| 3 | `aischool.MemberQueryFacade.getMemberInfoById` | 当前登录人资料 | 是 |
| 4 | `yucai.WebMemberWorkQueryFacade.queryMemberWorkByDate` | 按日期区间查作业列表 | 是 |
| 5 | `yucai.WebMemberWorkQueryFacade.listCategoryByStudent` | 学科（科目）字典 | 是 |
| 6 | `yucai.AppMemberWorkFacade.querySubjectWorkDetail` | 作业详情（内容/状态） | 是 |
| 7 | `yucai.AppMemberWorkFacade.queryAnswerSubmitByNoSheet` | 我的作答（图片/得分/批语） | 是 |
| 8 | `yucai.ReviewWorkModuleQueryFacade.queryMemberWorkReview` | 批阅反馈（标注/等级） | 是 |
| 9 | `yucai.AppPublishQueryFacade.queryHomeworkAnswer` | 参考答案 | 是 |

---

## 6. 接口详解

### 6.1 当前登录人资料

`aischool.MemberQueryFacade.getMemberInfoById` — param 传 `{}`，靠 token 识别。

响应 `data.resultObject` 主要字段：

| 字段 | 示例 |
|---|---|
| `memberId` / `id` | `202507040220586404` |
| `firstName` | `孔令泽` |
| `nickName` | 昵称 |
| `loginName` | `KLZ192338`（学号） |
| `schoolName` | `杭州育才中学` |
| `fullClassName` | `八年级 10班` |
| `grade` / `phase` | 年级 / 学段 |
| `memType` | `1` = 学生 |
| `avatarUrl` | 头像 URL |
| `mobileNo` | 手机号（脱敏） |
| `sex` | 性别 |
| `hasViewLoginName` | 是否展示学号 |
| `enrollmentYear` / `joinclassDate` | 入学年份 / 入班时间 |

### 6.2 作业列表（核心）

`yucai.WebMemberWorkQueryFacade.queryMemberWorkByDate`

```json
{ "qry": { "startDate": 1790000000000, "endDate": 1794000000000 } }
```

- `startDate` / `endDate`：**毫秒时间戳**，闭区间
- 实测可一次查 180 天（249 条），无明显上限

响应 `data.resultObject.webMemberWorkList[]`，单个作业项字段：

| 字段 | 类型 | 示例 | 说明 |
|---|---|---|---|
| `id` | string | `45609` | 发布 ID（publishId），详情类接口都用它 |
| `name` | string | `背诵课文` | 作业名称 |
| `categoryId` | number | `9` | 学科 ID |
| `categoryName` | string | `英语` | 学科名 |
| `homeworkType` | number | `4` | 1=学科作业 4=练习（背诵/跟读） |
| `startDate` | number | `1794153600000` | 布置时间 |
| `endDate` | number | `1794239400000` | 截止时间 |
| `completeStatus` | number | `0` | 1=已完成，0=未完成 |
| `reviewStatus` | number | `0` | 1=已批阅 |
| `reviewReadStatus` | number | `0` | 1=批阅结果已读 |
| `subId` | string | `1794153600000` | 子作业 ID，查批阅时用 |
| `answerStatus` | number | `1` | **不可靠**，见 §8 |
| `exemptStatus` | number | `0` | 是否免做 |
| `exemptCorrectStatus` | number | `0` | 是否免订正 |
| `signCurrentDay` / `signTotalDays` | number | `1` / `1` | 打卡类作业的签到进度 |
| `repulseStatus` | number | `0` | 是否被驳回 |
| `status` | number | `1` | 作业状态 |

> 响应里还有 `workSubmitTimeCO`（提交时间统计），本工具未使用。

### 6.3 学科字典

`yucai.WebMemberWorkQueryFacade.listCategoryByStudent` — param `{}`。

响应 `data.resultList[0]`：

| 字段 | 说明 |
|---|---|
| `subCategoryInfoCOList` | 科目数组 |
| `subjectCategoryId` / `subjectCategoryName` | 当前分组（顶层，通常为"全部"） |
| `categoryType` | 分组类型 |
| `isSelected` | 是否选中 |

`subCategoryInfoCOList[]` 单项：

```json
{ "subjectCategoryId": 7, "subjectCategoryName": "数学", "relationSubjectId": 2, "isSelected": true }
```

育才八年级实测 12 个科目（`subjectCategoryId` 可直接用于筛选）：

| ID | 科目 | ID | 科目 |
|---|---|---|---|
| 7 | 数学 | 35 | 物理 |
| 8 | 语文 | 36 | 化学 |
| 9 | 英语 | 40 | 生化 |
| 10 | 科学 | 43 | 地理 |
| 34 | 文综 | 44 | 历史 |
| 45 | 生物 | 46 | 政治 |

> 作业列表项里的 `categoryId` 与这里的 `subjectCategoryId` 是同一套编号（实测英语=9、数学=7、文综=34 均对应），可直接用于客户端筛选。

### 6.4 作业详情

`yucai.AppMemberWorkFacade.querySubjectWorkDetail` — `{ "publishId": "44496" }`

响应 `data.resultObject`：

| 字段 | 示例 | 说明 |
|---|---|---|
| `homeworkName` | `特殊三角形综合复习` | |
| `homeworkId` | `25593` | 作业模板 ID（≠ publishId） |
| `questionContent` | `语文练习纸` 或 `""` | 作业内容文字，常为空 |
| `subjectName` / `subjectId` | `数学` / `7` | |
| `homeworkType` | `1` | 1=学科 4=练习 |
| `answerStatus` | `2` | 答案状态（同样不可靠） |
| `sheetExistStatus` | `1` | 是否走答题卡（1=有） |
| `needCorrectStatus` | `1` | 是否需要订正 |
| `shouldCorrectionTime` | `1791388799000` | 订正截止时间 |
| `endDate` | `1791300600000` | 截止时间 |
| `status` | `3` | 作业状态 |

> **`homeworkType=4`（背诵/跟读）调本接口会直接返回 `1100001 学生作业数据获取错误`**，属正常现象。

### 6.5 我的作答

`yucai.AppMemberWorkFacade.queryAnswerSubmitByNoSheet` — `{ "qry": { "id": "44496" } }`

（注意参数名是 `qry.id`，不是 `publishId`）

响应 `data.resultObject` 关键字段：

| 字段 | 示例 | 说明 |
|---|---|---|
| `score` | `70` | 得分（数值，无单位） |
| `title` | `已完成` | 提交状态描述 |
| `description` | `已订正` | 老师批语/订正状态 |
| `mark` | `B` | 等级评价 |
| `reviewStatus` | `1` | 1=已批阅 |
| `correctionStatus` | `2` | 订正状态 |
| `questionImage` | `[{ fullUrl, workImageId, type }]` | **我的作答原图（对象数组）** |
| `answerImages` | `["https://...jpg"]` | 作答图（字符串数组，与 questionImage 重叠） |
| `correctImages` | `["https://...jpg"]` | 订正后的图片 |
| `publishAnswerStatus` | `1` | 是否已发答案 |
| `homeworkId` / `name` / `subjectName` / `className` / `studentName` | | 元信息 |
| `studentRepulseCO` | object | 被驳回时的信息 |
| `correctionRefuseMessage` | string | 订正被拒原因 |
| `excellentStudentList` | array | 优秀作业名单 |

> ⚠️ `questionImage` 是**对象数组**（取 `fullUrl`），`answerImages` 是**字符串数组**——同一个响应里两种格式混用，解析时必须分别处理。

### 6.6 批阅反馈

`yucai.ReviewWorkModuleQueryFacade.queryMemberWorkReview`

```json
{ "qry": { "publishId": "44496", "subId": "0", "readOnly": true } }
```

响应 `data.resultObject`：

| 字段 | 示例 | 说明 |
|---|---|---|
| `reviewStatus` | `1` | 1=已批阅，0=未批阅 |
| `mark` | `B` | 等级 |
| `markPointList` | `[]` | 批注点（坐标 + 文案） |
| `imageCorrectList` | `["https://...jpg"]` | **批阅标注图（字符串 URL 数组）** |
| `rewardPoint` | `0` | 奖励积分 |
| `rewardDetailList` | `[]` | 积分明细 |
| `excellentStatus` | `0` | 是否评优 |
| `repulseStatus` | `0` | 是否驳回 |

### 6.7 参考答案

`yucai.AppPublishQueryFacade.queryHomeworkAnswer` — `{ "publishId": "44496" }`

响应 `data.resultObject` —— **重点：答案有 5 种载体，必须全部检查**：

| 字段 | 类型 | 说明 |
|---|---|---|
| `answerImageList` | `string[]` | 答案图片 URL 数组（**字符串，不是对象**） |
| `answerUrlList` | `string[]` | 外部链接答案（图片或网页） |
| `answerAudios` | `string[]` | 音频答案（听力/朗读） |
| `answerOssKey` | `string` | **单个 OSS 文件绝对 URL**，99% 是 PDF（老师上传的整份答案原件） |
| `answerContent` | `string` | **纯文本答案**，含 `\n` 换行 |

其他字段：`name`、`subjectName`、`homeworkType`、`startDate`、`endDate`、`reportStatus`、`questionEditMode`、`singleQuestionEditMode`。

示例（三种不同形态，均真实存在）：

```json
// A. 图片答案
{ "answerImageList": ["https://public-oss.kaike.la/wlx_app/public/1a1b05c891a3796/1793960216684_nullimage_1793960137875_compress.jpg"], "answerUrlList": [], "answerAudios": [] }

// B. PDF 答案
{ "answerOssKey": "https://public-oss.estudy.cn/school-resource/school-paper/1a0c7c6c44e5200/初二中秋国庆假期育才答案汇总_10.6.pdf", "answerImageList": [], "answerContent": "" }

// C. 文本答案
{ "answerContent": "17.但最后远去的依然是我，停留在原地的，还是父亲。\n\n18.1“我\"和父亲赶火车，发车的时间不等人;...", "answerImageList": [] }
```

答案资源域名（均为公开、`Access-Control-Allow-Origin: *`，可直接 `<img>` / `<iframe>` / 新窗口打开，无需鉴权）：

- `public-oss.kaike.la`
- `public-oss.estudy.cn`（PDF 主要走这个，单文件约 1.3 MB）

---

## 7. 枚举字典

### homeworkType
| 值 | 含义 | 备注 |
|---|---|---|
| 1 | 学科作业 | 有题目纸 / 答题卡 |
| 4 | 练习（背诵、跟读、听力） | 答卷是录音，平台不放书面答案 |

> 工具里 `TYPE_MAP` 还预留了 2=考试 / 3=阅读 / 5=实践，但学生端实测只出现 1 和 4，其余未验证。

### completeStatus
| 值 | 含义 |
|---|---|
| 0 | 未完成 |
| 1 | 已完成 |

### reviewStatus
| 值 | 含义 |
|---|---|
| 0 | 未批阅 |
| 1 | 已批阅 |

### reviewReadStatus
| 值 | 含义 |
|---|---|
| 0 | 批阅结果待查看 |
| 1 | 已查看 |

### memType
| 值 | 含义 |
|---|---|
| 1 | 学生 |
| 其他 | 教师 / 家长（未逐一验证） |

---

## 8. 关键坑（血泪总结）

1. **`param` 是字符串不是对象** —— 网关做了双层 JSON 编码。传对象会报参数错误。

2. **`answerStatus` 不能当"有没有答案"的依据** —— 234 条作业里 106 条 `answerStatus=4`，绝大多数实际没有答案资源。唯一可靠做法是逐条调 `queryHomeworkAnswer`，看 5 个载体字段是否非空。

3. **同一个响应里数组元素类型不统一** ——
   - `answerImageList` / `answerUrlList` / `answerAudios` / `correctImages` / `imageCorrectList`：**字符串数组**
   - `questionImage`：**对象数组**（`{ fullUrl, workImageId, type }`）

   写成 `arr.map(x => x.fullUrl)` 通用处理会在字符串上取到 `undefined`，导致答案/作答图全部"消失"。

4. **答案不只在 `answerImageList` 里** —— 27/34 条有答案的作业，答案在 `answerOssKey`（PDF）；5 条在 `answerContent`（文本）。只看图片字段会漏掉 94%。

5. **`homeworkType=4` 的两个接口都会报错** —— `querySubjectWorkDetail` 返回"学生作业数据获取错误"，`queryHomeworkAnswer` 返回"作业信息查询失败"。这是设计如此（录音作业不开放），不是 token 或参数问题。

6. **`loginCode` 是一次性的** —— 拿到后要立刻换 token，不要缓存复用。

7. **批阅接口的 `subId`** —— 传列表项里的 `subId`，非答题卡类作业常为 `"0"`。

---

## 9. 错误码

| code | message | 说明 |
|---|---|---|
| `1000001` | 操作成功 | 成功 |
| `6200000` | 用户名或密码错误 | 密码错误（或账号不存在） |
| `1001010` | 缺少必要参数token或token无效，[开课啦网关]请检查请求参数！ | token 缺失/过期 |
| `1001009` | [开课啦网关]接口服务权限不存在 | 服务名写错，或无权限调用该服务 |
| `1100001` | 作业信息查询失败 | publishId 不存在，或该作业类型不开放（type=4） |

判定成功请用 `code === "1000001"`（字符串比较，别用 `success`，部分失败场景 `serResult` 仍是 `true`）。

---

## 10. 可直接用的最小客户端（Node / 浏览器通用）

```js
const BASE = 'https://yunyucai.cn';
const CLIENT_ID = '6';
let TOKEN = '';

async function api(service, param, token) {
  const env = {
    param: JSON.stringify(param || {}),
    service_name: service,
    strategy: 'dubbo',
    client_id: CLIENT_ID,
    timestamp: Date.now(),
  };
  if (token) env.token = token;
  const res = await fetch(`${BASE}/api.do?${encodeURIComponent(service)}`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(env),
  });
  const json = await res.json();
  if (String(json.code) !== '1000001') throw new Error(json.message || '请求失败');
  const d = json.data || {};
  return d.resultObject !== undefined ? d.resultObject : (d.resultList || d);
}

async function login(account, password) {
  const cmd = { app: 'xhxyc', clientType: 'web', sourceType: 'web', userAgent: navigator.userAgent };
  const users = await api('aischool.LoginCmdFacade.checkAndQueryUserByWeb', {
    cmd: { ...cmd, loginName: account, password: md5(password) },
  });
  const u = users[0];
  const r = await api('aischool.LoginCmdFacade.loginWebByCode', {
    cmd: { ...cmd, memberId: u.memberId, loginCode: u.loginCode },
  });
  TOKEN = r.token;
  return u;
}

// 用法
// await login('手机号', '密码');
// const list = (await api('yucai.WebMemberWorkQueryFacade.queryMemberWorkByDate',
//   { qry: { startDate: Date.now() - 30 * 864e5, endDate: Date.now() } }, TOKEN)).webMemberWorkList;
// const ans = await api('yucai.AppPublishQueryFacade.queryHomeworkAnswer', { publishId: list[0].id }, TOKEN);
```

`md5()` 未内置，任意实现皆可（如 `crypto.createHash('md5')` 或前端 spark-md5）。

### curl 示例

```bash
curl -X POST "https://yunyucai.cn/api.do?aischool.LoginCmdFacade.checkAndQueryUserByWeb" \
  -H "Content-Type: application/json" \
  -d '{"service_name":"aischool.LoginCmdFacade.checkAndQueryUserByWeb","strategy":"dubbo","client_id":"6","timestamp":1794146693000,"param":"{\"cmd\":{\"loginName\":\"1xxxxxxxxxx\",\"password\":\"<md5>\",\"app\":\"xhxyc\",\"clientType\":\"web\",\"sourceType\":\"web\",\"userAgent\":\"Mozilla/5.0\"}}"}'
```

---

## 11. 附：未被使用但已探到的服务名

同一网关下还可调用（学生端前端里出现，本次未深入）：

| 服务名 | 推测用途 |
|---|---|
| `aischool.MemberInfoFacade.modifyPassword` | 修改密码 |
| `aischool.AiSchoolAuthFacade.listUserAuth` | 权限列表 |
| `aischool.LoginCmdFacade.loginWebByMemberId` | 免密登录（需额外凭据） |
| `learnprod.WeekVodQueryFacade.queryCatalogue` | 课程目录 |
| `learnprod.StudyConfigQueryFacade.queryConfig` | 学习端配置 |
| `marketactivity.CoinUserPointsQueryFacade.queryPointsRuleByKeys` | 积分规则 |
| `marketactivity.CoinProductQueryFacade.queryMemberAccountInfo` | 积分账户 |
| `estudy.CommonFacade.getSingleObtainPropertiesKeyValue` | 通用配置读取 |

---

*整理时间：2026-11-09 · 数据来源：yunyucai.cn 学生端实测*
