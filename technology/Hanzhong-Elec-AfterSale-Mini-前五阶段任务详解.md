## 文档目的

这份笔记用于让后续接手的大模型或开发者快速理解当前项目的前后端联调任务、目标状态、关键文件、接口约束和每一步的验收标准。

项目分为两个部分：

- 小程序前端：`E:\Backstage-Management-System\Hanzhong-Elec-AfterSale-Mini`
- Java 后端：`E:\Backstage-Management-System\RuoYi-Vue-master`

当前的核心问题不是“页面不能显示”，而是“前端虽然能跑，但很多页面仍然在使用本地 mock 数据或本地缓存，尚未真正和后端 App 接口体系打通”。

这份笔记只描述六个总任务中的前五个任务，不包含最后的购物车下单部分。

---

## 任务总览

前五个任务是：

1. 统一基础请求层：`config.js` 和 `utils/request.js`
2. 改造认证接口封装：`api/login.js`
3. 改造登录与个人中心：`pages/profile/login.vue` 和 `pages/profile/index.vue`
4. 打通售后申请与售后订单：`pages/applyAfterSale/index.vue` 和 `pages/afterSaleOrder/index.vue`
5. 打通配件商城与配件详情：`pages/accessoryMall/index.vue` 和 `pages/accessoryDetail/index.vue`

这五部分的目标是完成“从登录到浏览业务数据”的前后端基础闭环。

---

# 第一部分：统一基础请求层

## 目标

让前端所有接口请求都走统一的请求入口，并且：

- 后端地址统一从 `config.js` 读取
- token 自动带上
- 兼容当前项目里两种 `request` 调用方式
- 成功时返回统一结构，避免页面层拿不到 `rows`、`data` 等字段
- 上传请求与普通请求在 token 读取方式、失效跳转页、基础地址来源上保持一致

## 关键文件

- `E:\Backstage-Management-System\Hanzhong-Elec-AfterSale-Mini\config.js`
- `E:\Backstage-Management-System\Hanzhong-Elec-AfterSale-Mini\utils\request.js`
- `E:\Backstage-Management-System\Hanzhong-Elec-AfterSale-Mini\utils\upload.js`
- `E:\Backstage-Management-System\Hanzhong-Elec-AfterSale-Mini\utils\auth.js`
- `E:\Backstage-Management-System\Hanzhong-Elec-AfterSale-Mini\api\system\user.js`

## 必须满足的要求

### 1. `baseUrl` 必须统一

前端不允许同时存在多个后端地址来源。

必须统一为：

- `config.js` 里维护一个 `baseUrl`
- `utils/request.js` 从 `config.js` 读取 `baseUrl`
- `utils/upload.js` 也从 `config.js` 读取 `baseUrl`

不要再出现：

- 一个文件写 `localhost`
- 另一个文件写局域网 IP
- 某些页面里再手写完整地址

### 2. `request` 必须兼容两种调用方式

当前项目里存在两种风格：

位置参数风格：

```js
request('/app/auth/login', 'POST', data)
```

对象参数风格：

```js
request({
  url: '/system/user/profile',
  method: 'get',
  data: {},
  headers: {}
})
```

为了避免后续旧代码全部报错，`utils/request.js` 必须同时兼容这两种调用方式。

### 3. token 自动注入

统一请求层中要自动读取：

```js
uni.getStorageSync('token')
```

或者通过统一 token 工具获取当前 token。

然后在请求头中加入：

```http
Authorization: Bearer xxx
```

登录、注册、发送验证码等接口允许通过 `isToken: false` 跳过。

### 4. 成功时直接返回整个 `res.data`

不要在请求层提前只返回 `res.data.data`。

原因：

- 有的接口返回结构是：`{ code, msg, data }`
- 有的列表接口返回结构是：`{ code, msg, rows, total }`

如果请求层提前只取 `data`，那像 `rows` 和 `total` 就会丢失。

### 5. 错误时保留后端 `msg`

请求失败时，页面层需要能够直接拿到后端返回的 `code` 和 `msg`。

### 6. 上传请求和普通请求要共享同一套登录态规则

上传请求不能再单独使用旧 token key 或旧登录页跳转地址。

必须与普通请求保持一致：

- 读取同一套 token
- 登录失效时跳到同一个登录页
- 使用同一个 `baseUrl`

### 7. 对象模式下支持 `params`

`request({ ... })` 不仅要支持 `url`、`method`、`data`、`headers/header`、`timeout`，还应支持：

```js
request({
  url: '/xxx',
  method: 'GET',
  params: {
    pageNum: 1,
    pageSize: 10
  }
})
```

并自动拼接为 query string。

## 当前后端约束

后端默认端口和上下文路径来自：

- `E:\Backstage-Management-System\RuoYi-Vue-master\ruoyi-admin\src\main\resources\application.yml`

默认值是：

- `port: 8080`
- `context-path: /`

因此前端基础地址通常是：

```text
http://你的主机IP:8080
```

如果只在本机开发者工具中跑，可以暂时使用：

```text
http://localhost:8080
```

如果要真机调试，必须改成局域网 IP。

## 验收标准

完成这一部分后，下面两种写法都必须可用：

```js
request('/app/auth/profile', 'GET')
```

```js
request({
  url: '/system/user/profile',
  method: 'get',
  params: {
    pageNum: 1,
    pageSize: 10
  }
})
```

并且请求头中的 `Authorization` 能自动带上。

## 当前完成情况（2026-03-23）

任务一已经完成，当前基础请求层已经满足“统一地址来源、统一 token 注入、兼容两种 request 调用方式、成功返回整个 res.data”这四个核心目标。

### 本部分实际修改的文件

#### 1. `E:\Backstage-Management-System\Hanzhong-Elec-AfterSale-Mini\utils\request.js`

这是任务一的主修改文件，已完成以下改动：

- 将 `baseUrl` 的来源统一为：
  - `import config from '@/config'`
  - `const baseUrl = config.baseUrl`
- 保留并统一了 `Authorization: Bearer token` 自动注入逻辑。
- 增加了 `isToken: false` 开关支持：
  - 当某个接口显式声明 `isToken: false` 时，请求层不会自动携带 token。
- 将 `request` 方法改为同时兼容两种调用方式：

```js
request('/app/auth/login', 'POST', data)
```

```js
request({
  url: '/system/user/profile',
  method: 'get',
  data: {},
  headers: {}
})
```

- 成功响应时统一执行：

```js
resolve(res.data)
```

而不是提前裁剪成 `res.data.data`。

- 错误响应时统一保留：
  - `code`
  - `msg`

方便页面层直接提示后端原始错误信息。

- 补充了：

```js
export default request
```

这样旧代码中使用：

```js
import request from '@/utils/request'
```

的文件也不会立即失效。

#### 2. `E:\Backstage-Management-System\Hanzhong-Elec-AfterSale-Mini\utils\upload.js`

这是任务一的配套修改文件，目的是让上传行为和统一请求层保持一致。

已完成以下改动：

- 继续从：

```js
import config from '@/config'
const baseUrl = config.baseUrl
```

读取上传基础地址。

- 合并 `config.headers` 和 `config.header`，避免调用方有的传 `headers`、有的传 `header` 导致行为不一致。
- 增加 `isToken: false` 的识别逻辑：
  - 如果上传接口显式设置 `isToken: false`，则不会自动携带 token。
- 自动删除 `isToken`，避免把这个前端控制参数直接带到后端。

#### 3. `E:\Backstage-Management-System\Hanzhong-Elec-AfterSale-Mini\utils\auth.js`

这是任务一在复核阶段实际补改的文件。

已完成以下改动：

- 将主 token key 统一为：

```js
const TokenKey = 'token'
```

- 兼容读取历史遗留的：

```js
const LegacyTokenKey = 'App-Token'
```

- `getToken()` 现在会优先读 `token`，不存在时回退读 `App-Token`。
- `setToken()` 写入新 token 时会自动清理旧的 `App-Token`。
- `removeToken()` 会同时清理 `token` 和 `App-Token`。

### 本部分关联但未在初次实现中主动改动的文件

#### 4. `E:\Backstage-Management-System\Hanzhong-Elec-AfterSale-Mini\config.js`

这个文件在任务一中作为唯一的 `baseUrl` 配置源被使用。

当前值是：

```js
baseUrl: 'http://localhost:8080'
```

说明：

- 这个值适合本机开发者工具联调。
- 如果后续需要真机调试，应改为：
  - `http://你的局域网IP:8080`

本次“任务一完成”的含义不是强制把它改成某一个固定值，而是“所有请求都统一从这里读取”，不再出现多个地址来源。

#### 5. `E:\Backstage-Management-System\Hanzhong-Elec-AfterSale-Mini\api\system\user.js`

这个文件没有被直接修改，但它是任务一的验证对象。

它当前仍然使用：

```js
import request from '@/utils/request'
return request({ url, method, data })
```

任务一完成的一个重要标志，就是：

- 即使这个文件不改，当前 `utils/request.js` 也已经可以兼容它的调用方式。

### 任务一当前达成的技术结果

完成任务一后，项目现在已经具备以下能力：

1. 所有普通请求统一通过 `config.js` 读取 `baseUrl`。
2. 上传请求也统一通过 `config.js` 读取 `baseUrl`。
3. `request` 兼容“位置参数写法”和“对象写法”。
4. token 可以自动注入。
5. 登录、注册、发验证码这类接口可以通过 `isToken: false` 跳过 token 注入。
6. 成功响应统一返回整个 `res.data`，后续列表接口可以读取 `rows`、`total`。
7. 老代码里 `import request from '@/utils/request'` 不会因为缺少默认导出而立即报错。
8. 上传请求与普通请求已经统一到同一套 token 读取方式。
9. 对象参数模式现在支持 `params` 自动拼接。

### 任务一仍需使用者自行确认的环境项

虽然任务一的代码层实现已经完成，但仍有一个环境项需要根据联调方式自行确认：

- 如果你只在本机开发者工具里联调：
  - `config.js` 保持 `http://localhost:8080` 即可。
- 如果你要在手机真机上调试：
  - 需要把 `config.js` 的 `baseUrl` 改为电脑的局域网 IP。

这个环境项不影响“任务一代码结构已经完成”的结论。

### 二次复核补充（总规划师）

#### 本次补充说明（2026-03-23）

本次补充专门处理了任务一在二次复核中提出的 3 个遗留问题，具体如下：

1. `E:\Backstage-Management-System\Hanzhong-Elec-AfterSale-Mini\utils\auth.js`
   - 已将主 token key 统一为 `token`。
   - 已兼容读取历史遗留的 `App-Token`。
   - 写入新 token 时会自动清理旧的 `App-Token`。

2. `E:\Backstage-Management-System\Hanzhong-Elec-AfterSale-Mini\utils\upload.js`
   - 上传鉴权现在通过 `getToken()` 读取统一后的 token。
   - 上传接口的 401 登录失效跳转页，已从旧的 `/pages/login/login` 改为当前实际使用的 `/pages/profile/login`。

3. `E:\Backstage-Management-System\Hanzhong-Elec-AfterSale-Mini\utils\request.js`
   - 已补充对象模式下的 `params` 自动拼接支持。
   - 现在可以直接使用：

```js
request({
  url: '/xxx',
  method: 'GET',
  params: {
    pageNum: 1,
    pageSize: 10
  }
})
```

#### 当前结论

- 任务一在二次复核中提出的“上传鉴权 token key 不统一”、“上传 401 跳转页错误”、“request 缺少 params 支持”这三个问题都已完成修复。
- 当前任务一剩余的唯一注意事项仍然只是环境项：`config.js` 中的 `baseUrl` 目前为 `http://localhost:8080`，适合本机开发者工具联调；如果后续要做真机调试，仍需手动改成电脑的局域网 IP。

### 二次复核补充（总规划师）

#### 本次补充说明（2026-03-23）

- 当前 `login.vue` 注册提交的实际字段为 `phone`、`password`、`confirmPassword`、`nickName`、`code`、`roleType`、`merchantName`；后端 `AppRegisterBody` 还支持的 `contactName`、`address`、`serviceScope`，当时页面层还没有提供输入项。
- 当前 `saveAuthState()` 与 `mergeUserInfo()` 只把 `roleType === '2'` 识别为商家，而后端对新注册商家返回的很可能是 `roleType === '0'`（待审核商家）；因此“新商家注册后个人中心立刻显示商家视图”这一点当时还未真正完成。
- `pages/profile/index.vue` 中的 `getInfo()` 只会刷新 `AppUser`，不会刷新 `merchant` 对象；当时代码通过本地缓存保留 `merchant`，足够支撑基础展示，但商家资料一旦在后端审核或更新，前端本地 `merchant` 仍可能滞后。
- 个人中心里的 `statData`、客服弹窗、地址簿仍然主要是本地展示逻辑；这不影响“真实登录态已打通”的结论，但这些内容不能算“已经完成后端联调”。

#### 当前结论

- 任务三在二次复核中识别出的核心问题有 3 类：商家扩展字段缺失、待审核商家角色语义未处理、商家资料刷新不完整。
- 当时这些问题虽然不阻塞“真实登录态已打通”的主结论，但确实属于第三部分需要继续收口的边界项。

### 三次复核补充（总规划师）

#### 本次补充说明（2026-03-23）

- 已改造 `E:\Backstage-Management-System\Hanzhong-Elec-AfterSale-Mini\pages\profile\login.vue`：登录表单统一改为 `phone`、`password`；注册表单不再依赖本地 `userList`、`mockLogin`、`mockRegister`，并统一组装 `phone`、`password`、`confirmPassword`、`nickName`、`code`、`roleType`、`merchantName` 等真实注册字段。
- 已改造 `E:\Backstage-Management-System\Hanzhong-Elec-AfterSale-Mini\pages\profile\login.vue`：新增 `saveAuthState()` 统一写入 `token` 与 `userInfo`，补齐 `nickName`、`nickname`、`roleType`、`role`、`merchant`、`avatar`；当注册接口未直接返回 `token` 时，会自动切回登录 tab 并回填手机号、密码，避免流程中断。
- 已改造 `E:\Backstage-Management-System\Hanzhong-Elec-AfterSale-Mini\pages\profile\index.vue`：页面内嵌的本地登录/注册表单已去掉，未登录态统一显示“去登录 / 注册”入口；`onShow` 时会先校验本地 `token + userInfo`，再调用 `getInfo()` 刷新真实 `AppUser` 资料，并通过 `mergeUserInfo()`、`updateUserInfo()` 统一同步昵称、头像、角色和 `merchant` 缓存。
- 已新增 `E:\Backstage-Management-System\Hanzhong-Elec-AfterSale-Mini\api\merchant.js`：封装 `GET /app/merchant/info`，用于正式商家进入个人中心时刷新最新商家资料，避免本地缓存中的 `merchant` 长期滞后。
- 已补齐任务三二次复核中指出的边界问题：商家注册页补充了 `contactName`、`address`、`serviceScope` 输入项；登录态语义统一走 `getRoleMeta()`，单独识别 `roleType === '0'` 的待审核商家状态，不再把它误判成已审核商家。
- 当前任务三已经达成的技术结果是：登录页不再使用 `mockLogin`；注册页不再使用 `mockRegister`；注册页支持验证码发送；登录成功后可以进入个人中心主页；个人中心主页可以识别真实登录态，不再依赖本地 `userList`；正式商家进入个人中心时可以额外刷新最新商家资料。
- 当前仍需单独说明的边界项是：个人中心中的 `statData` 统计卡片、客服弹窗、地址簿入口仍主要是本地展示或本地缓存逻辑，这些内容不影响“真实登录态已打通”的结论，但当前不能记为“已完成后端联调”。

#### 当前结论

- 任务三三次复核中已经把“登录注册仍残留 mock 逻辑”、“待审核商家角色语义未处理”、“商家资料刷新不完整”、“商家扩展字段缺失”这几类问题全部收口。
- 当前任务三剩余需要继续说明的，只是个人中心里那些本地展示型模块，它们不影响“真实登录态已经打通”的结论。
- 当前第三部分已经同时保留了 `二次复核补充（总规划师）` 与 `三次复核补充（总规划师）` 两层复核记录，结构与第一部分一致。

---
# 第四部分：打通售后申请与售后订单

## 目标

让“提交售后申请 -> 查看售后订单列表 -> 用户取消订单 / 商家接单与推进状态”走真实后端接口，而不是本地缓存。

## 关键文件

- `E:\Backstage-Management-System\Hanzhong-Elec-AfterSale-Mini\pages\applyAfterSale\index.vue`
- `E:\Backstage-Management-System\Hanzhong-Elec-AfterSale-Mini\pages\afterSaleOrder\index.vue`
- `E:\Backstage-Management-System\Hanzhong-Elec-AfterSale-Mini\api\afterSale.js`

后端参考：

- `E:\Backstage-Management-System\RuoYi-Vue-master\ruoyi-admin\src\main\java\com\ruoyi\web\controller\app\AppUserController.java`
- `E:\Backstage-Management-System\RuoYi-Vue-master\ruoyi-admin\src\main\java\com\ruoyi\web\controller\app\AppMerchantController.java`
- `E:\Backstage-Management-System\RuoYi-Vue-master\ruoyi-system\src\main\java\com\ruoyi\app\service\impl\AppAfterSalesOrderServiceImpl.java`
- `E:\Backstage-Management-System\RuoYi-Vue-master\ruoyi-system\src\main\java\com\ruoyi\app\domain\AppConstants.java`
- `E:\Backstage-Management-System\RuoYi-Vue-master\ruoyi-system\src\main\java\com\ruoyi\app\domain\AppAfterSalesOrder.java`
- `E:\Backstage-Management-System\RuoYi-Vue-master\ruoyi-system\src\main\java\com\ruoyi\app\domain\bo\AppOrderActionBody.java`

## 售后申请页必须满足的要求

### 1. 删除本地 `afterSaleOrders` 写入逻辑

不要再执行：

```js
const afterSaleOrders = wx.getStorageSync('afterSaleOrders') || []
afterSaleOrders.unshift(afterSaleOrder)
wx.setStorageSync('afterSaleOrders', afterSaleOrders)
```

### 2. 改成调用真实接口

使用：

```text
POST /app/user/afterSalesOrder
```

### 3. 提交参数需要映射到后端模型

前端当前表单字段和后端 `AppAfterSalesOrder` 不完全一致，建议映射为：

```js
{
  productType: form.productName + ' / ' + form.productModel,
  faultDesc: form.faultDesc,
  faultImages: form.imgList.join(','),
  contactName: form.name,
  contactPhone: form.phone,
  address: form.region + ' ' + form.detail
}
```

注意：

- 后端并没有 `productName` 和 `productModel` 两个独立字段
- 它只有 `productType`

### 4. 成功后再跳订单列表页

不要再因为本地写入成功就算完成。

只有接口返回成功后，才提示“提交成功”。

## 售后订单页必须满足的要求

### 1. 删除本地 `mockData`

不要再在页面里生成假的订单数据。

### 2. 删除本地 `afterSaleOrders` 读取逻辑

订单列表应从后端接口读取：

普通用户：

```text
GET /app/user/afterSalesOrder/list
```

商家待处理：

```text
GET /app/merchant/order/pendingList
```

商家全部订单：

```text
GET /app/merchant/order/list
```

### 3. 状态值必须和后端一致

后端状态值：

- `0` 待接单
- `1` 已接单
- `2` 维修中
- `3` 已完成
- `4` 已取消

不要再使用本地状态：

- `-1`
- `-2`

### 4. 页面按钮逻辑要按后端状态流转改造

根据后端 `AppAfterSalesOrderServiceImpl` 的状态校验规则：

- `0 -> 1`：商家接单
- `0 -> 4`：用户取消
- `1 -> 2`：进入维修中
- `2 -> 3`：完成订单

因此商家页不应该在待接单时出现“拒绝”按钮，至少当前后端实现并没有给商家一个明确的“拒绝待接单订单”接口。

### 5. 用户取消接口

```text
PUT /app/user/afterSalesOrder/cancel/{orderId}
```

### 6. 商家接单接口

```text
PUT /app/merchant/order/take/{orderId}
```

### 7. 商家更新状态接口

```text
PUT /app/merchant/order/status
```

请求体类似：

```json
{
  "orderId": 123,
  "status": "2",
  "serviceRemark": "开始维修"
}
```

## 验收标准

完成第四部分后，应满足：

1. 用户能提交真实售后申请
2. 用户能看到自己的真实售后订单列表
3. 用户能取消待接单订单
4. 商家能查看待接单订单并接单
5. 商家能把已接单订单推进到“维修中”和“已完成”

## 当前完成情况（2026-03-23，任务四）

任务四已完成，当前售后申请页和售后订单页已经切换到真实后端接口，不再依赖本地 `afterSaleOrders`、`mockData` 或页面内的假订单逻辑。

### 本部分实际修改的文件

#### 1. `E:\Backstage-Management-System\Hanzhong-Elec-AfterSale-Mini\api\afterSale.js`

这是任务四的接口封装文件，已完成以下改动：

- 封装了 `uploadAfterSaleImage(filePath)`，上传故障图片到：

```text
POST /common/upload
```

- 封装了 `createAfterSaleOrder(data)`，提交售后申请到：

```text
POST /app/user/afterSalesOrder
```

- 封装了用户订单列表接口：

```text
GET /app/user/afterSalesOrder/list
```

- 封装了用户取消接口：

```text
PUT /app/user/afterSalesOrder/cancel/{orderId}
```

- 封装了商家待接单列表：

```text
GET /app/merchant/order/pendingList
```

- 封装了商家全部订单列表：

```text
GET /app/merchant/order/list
```

- 封装了商家接单接口：

```text
PUT /app/merchant/order/take/{orderId}
```

- 封装了商家更新订单状态接口：

```text
PUT /app/merchant/order/status
```

#### 2. `E:\Backstage-Management-System\Hanzhong-Elec-AfterSale-Mini\pages\applyAfterSale\index.vue`

这是任务四的用户提交页，已完成以下改动：

- 删除本地 `afterSaleOrders` 读写逻辑，不再通过本地缓存伪造申请成功。
- 页面表单仍保留原来的前端输入结构：
  - `productName`
  - `productModel`
  - `faultDesc`
  - `imgList`
  - `name`
  - `phone`
  - `region`
  - `detail`
- 提交前会把这些字段映射成后端实际需要的结构：

```js
{
  productType,
  faultDesc,
  faultImages,
  contactName,
  contactPhone,
  address
}
```

- 故障图片改为先通过 `/common/upload` 上传，拿到后端返回的 `url` 后再拼进 `faultImages`。
- 提交成功后再提示成功，并跳转：

```text
/pages/afterSaleOrder/index?type=all&role=user
```

- 增加了登录校验、默认地址回填、图片删除、图片上传失败兜底等流程控制。

#### 3. `E:\Backstage-Management-System\Hanzhong-Elec-AfterSale-Mini\pages\afterSaleOrder\index.vue`

这是任务四的订单列表页，已完成以下改动：

- 删除本地 `mockData` 与本地 `afterSaleOrders` 读取逻辑。
- 普通用户模式下读取：

```text
GET /app/user/afterSalesOrder/list
```

- 商家待接单模式下读取：

```text
GET /app/merchant/order/pendingList
```

- 商家全部订单模式下读取：

```text
GET /app/merchant/order/list
```

- 页面已统一按照后端字段展示：
  - `orderNo`
  - `productType`
  - `faultDesc`
  - `contactName`
  - `contactPhone`
  - `address`
  - `serviceRemark`
  - `faultImages`
  - `createTime`
  - `userName`
  - `merchantName`

- 状态值已统一为后端真实状态：
  - `0` 待接单
  - `1` 已接单
  - `2` 维修中
  - `3` 已完成
  - `4` 已取消

- 商家页面不再出现本地“拒绝”按钮，只保留真实后端存在的状态流转按钮。

### 本部分当前结论

从“售后申请页 + 售后订单页 + 售后接口封装”这个范围看，任务四已经完成。

当前已经收口完成的能力：

1. 用户提交售后申请走真实接口。
2. 用户查看自己的售后订单走真实接口。
3. 商家查看待接单订单和全部订单走真实接口。
4. 商家接单、开始维修、完成订单走真实接口。
5. 故障图片上传与参数映射已经接通。

当前仍需通过联调确认的内容：

- 图片上传在实际设备上的稳定性。
- 用户账号和商家账号在真实数据下的权限表现。
- 较大订单列表和分页场景下的页面表现。

### 二次复核补充（总规划师）

- 当前前端“用户取消”按钮比后端真实能力更收敛：后端 `cancelOrder()` 实际允许用户在 `0`、`1`、`2` 状态下取消，只禁止 `3`、`4`；而现在页面 `canCancel()` 只在 `0` 时展示按钮。
- 当前商家页面也比后端真实能力更收敛：后端 `/app/merchant/order/status` 在已接单或维修中阶段实际上可以把订单更新为 `4` 已取消，但当前页面只暴露了 `1 -> 2` 和 `2 -> 3` 两个按钮，没有提供“商家取消 / 终止”入口。
- `api/afterSale.js` 当前已封装 flat query 参数，常用的 `pageNum`、`pageSize`、`status`、`orderNo`、`productType`、`userName`、`merchantName` 都可以直接传；但如果后续要对接后端 Mapper 中的 `params.beginTime`、`params.endTime` 时间范围筛选，现有 `buildQuery()` 还不支持嵌套参数，需要额外扩展。
- 当前售后图片上传依赖 `/common/upload` 返回的 `url` 字段，参数链路已经通；但它仍然依赖第一部分提到的 `upload.js`，因此上传鉴权策略一旦收紧，需要先确保任务一里的上传鉴权统一性仍然成立。

### 二次复核补充已完成（2026-03-23）

- 已修改 `E:\Backstage-Management-System\Hanzhong-Elec-AfterSale-Mini\pages\afterSaleOrder\index.vue`：用户取消按钮范围从仅 `0` 状态扩展为 `0 / 1 / 2`，与后端 `cancelOrder()` 的真实能力对齐。
- 已修改 `E:\Backstage-Management-System\Hanzhong-Elec-AfterSale-Mini\pages\afterSaleOrder\index.vue`：新增商家“终止订单”入口，商家在 `1 已接单`、`2 维修中` 两个阶段都可调用 `/app/merchant/order/status` 将订单更新为 `4 已取消`，并附带 `serviceRemark`。
- 已修改 `E:\Backstage-Management-System\Hanzhong-Elec-AfterSale-Mini\api\afterSale.js`：`buildQuery()` 现在支持嵌套对象与数组的序列化，后续可传 `params: { beginTime, endTime }` 这类结构并生成 `params[beginTime]`、`params[endTime]` 查询参数。
- 已同步重写 `E:\Backstage-Management-System\Hanzhong-Elec-AfterSale-Mini\pages\applyAfterSale\index.vue` 与 `E:\Backstage-Management-System\Hanzhong-Elec-AfterSale-Mini\pages\afterSaleOrder\index.vue` 的当前 UTF-8 页面内容，修复此前文件中的乱码与模板损坏，保持任务四页面可以继续联调。
- 当前这轮二次复核已完成代码修正，但仍未在 HBuilderX、微信开发者工具或真机下实测；建议下一步重点验证：用户在 `已接单 / 维修中` 状态下取消、商家在 `已接单 / 维修中` 状态下终止订单，以及带 `params.beginTime / params.endTime` 的订单查询。

### 三次复核补充（总规划师）

- 仍需补充一个列表边界：当前订单页固定请求 `pageNum = 1`、`pageSize = 100`，页面还没有做分页切换、上拉加载或“加载更多”；如果某个账号的订单数超过 100 条，当前页面只会显示第一页数据，这属于后续可继续完善的列表分页项。

### 三次复核补充已完成（2026-03-23）

- 已修改 `E:\Backstage-Management-System\Hanzhong-Elec-AfterSale-Mini\pages\afterSaleOrder\index.vue`：新增分页状态 `pageNum`、`pageSize`、`total`、`hasMore`、`isLoadingMore`，并将列表请求从固定第一页改为可持续加载的分页模式。
- 已补充订单页 `onReachBottom()` 与页面底部“点击加载更多”交互，当前用户订单、商家待接单、商家全部订单都会按后端分页结果继续请求后续页，不再固定停留在前 100 条或第一页数据。
- 当前分页实现仍保持任务四原有的前端筛选方式：用户侧 `pending / finished`、商家侧 `pending / finished` 仍是基于已加载订单列表做前端过滤，但底层数据源现在可以继续翻页累积。
- 本轮未改动任务四的接口路径与状态流转逻辑；`api/afterSale.js` 继续负责售后接口封装，订单页在分页刷新后会保留既有的用户取消、商家接单、开始维修、完成订单、终止订单逻辑。
- 尚未在 HBuilderX、微信开发者工具或真机下实测分页边界；后续建议重点验证：订单超过一页时的上拉加载、操作后重新刷新是否回到第一页，以及 `total` 与前端累计条数是否一致。

### 当前结论

任务四的完整实现、二次复核修正和三次复核修正都已经完成。

当前第四部分剩余的主要注意事项只有两类：

1. 真实设备联调验证仍未完成。
2. 分页、图片上传和状态流转需要分别在用户账号与商家账号下跑一遍完整闭环。

---
# 第五部分：打通配件商城与配件详情

## 目标

让配件商城和配件详情不再依赖本地假数据，而是展示真实后端接口返回的配件数据。

## 关键文件

- `E:\Backstage-Management-System\Hanzhong-Elec-AfterSale-Mini\pages\accessoryMall\index.vue`
- `E:\Backstage-Management-System\Hanzhong-Elec-AfterSale-Mini\pages\accessoryDetail\index.vue`
- 建议新增接口封装：`E:\Backstage-Management-System\Hanzhong-Elec-AfterSale-Mini\api\accessory.js`

后端参考：

- `E:\Backstage-Management-System\RuoYi-Vue-master\ruoyi-admin\src\main\java\com\ruoyi\web\controller\app\AppCommonController.java`
- `E:\Backstage-Management-System\RuoYi-Vue-master\ruoyi-system\src\main\java\com\ruoyi\app\domain\AppAccessory.java`
- `E:\Backstage-Management-System\RuoYi-Vue-master\ruoyi-system\src\main\java\com\ruoyi\app\service\impl\AppAccessoryServiceImpl.java`

## 配件商城必须满足的要求

### 1. 删除本地 `goodsList` 主数据源逻辑

当前页面如果仍在做：

- `getStorageSync('goodsList')`
- `setStorageSync('goodsList')`
- `initGoodsList`

说明它还没真正接到后端。

### 2. 改成调用真实接口

使用：

```text
GET /app/common/accessory/list
```

### 3. 正确读取列表返回结构

这个接口返回的是分页结构：

```json
{
  "code": 200,
  "msg": "查询成功",
  "rows": [...],
  "total": 10
}
```

因此前端应从 `res.rows` 取数组，而不是从 `res.data` 取。

### 4. 后端字段要映射成页面需要的字段

后端 `AppAccessory` 的字段包括：

- `accessoryId`
- `categoryName`
- `accessoryName`
- `accessoryDesc`
- `coverImage`
- `price`
- `stock`
- `status`

前端页面映射建议：

```js
{
  id: accessoryId,
  name: accessoryName,
  spec: categoryName,
  desc: accessoryDesc,
  image: coverImage,
  price,
  stock
}
```

### 5. 商家编辑商品功能先不要强行打通

当前后端配件新增、编辑、删除接口在 `AppAdminController`，不是 `AppMerchantController`，更推荐先隐藏，避免误导。

## 配件详情必须满足的要求

### 1. 删除页面内写死的 `goodsData`

### 2. 改成调用详情接口

使用：

```text
GET /app/common/accessory/{accessoryId}
```

### 3. 正确做字段映射

详情接口返回的是单个 `AppAccessory`，前端应映射到当前页面的 `currentGoods`。

### 4. 加入购物车逻辑可以先保留本地

因为购物车本身在当前任务范围内还不是重点，加入购物车先继续本地存储是可以接受的。

但详情页的商品数据本身必须来自后端，而不是本地写死对象。

## 验收标准

完成第五部分后，应满足：

1. 配件商城显示真实后端配件列表
2. 搜索和分类筛选基于真实列表做前端过滤
3. 配件详情页展示真实后端详情
4. 不再依赖 `initGoodsList` 或 `goodsData` 这类本地假数据

## 当前完成情况（2026-03-23，任务五）

任务五已完成，当前配件商城列表、搜索筛选、详情查看都已切换到真实后端接口；购物车仍按任务要求保留本地存储。

### 本部分实际修改的文件

#### 1. `E:\Backstage-Management-System\Hanzhong-Elec-AfterSale-Mini\api\accessory.js`

- 已新增 `getAccessoryList()`、`getAccessoryDetail()`。
- 已新增 `normalizeAccessory()`、`resolveAccessoryImage()`。
- `coverImage` 会基于 `config.js` 中的 `baseUrl` 补全为页面可直接展示的图片地址。

#### 2. `E:\Backstage-Management-System\Hanzhong-Elec-AfterSale-Mini\pages\accessoryMall\index.vue`

- 已删除本地 `initGoodsList`、本地商家编辑弹窗以及未启用的本地商品维护逻辑。
- 已改为通过 `loadGoodsList()` 调用真实列表接口，并从 `res.rows` 读取数据。
- 已新增 `fetchAllAccessories()`，通过 `pageNum`、`pageSize` 循环拉取所有分页数据。
- 分类列表改为基于真实配件数据动态生成，搜索和分类筛选都改为基于真实 `goodsList` 的前端过滤。
- 加入购物车仍保留本地 `cartList` 存储。

#### 3. `E:\Backstage-Management-System\Hanzhong-Elec-AfterSale-Mini\pages\accessoryDetail\index.vue`

- 已删除页面内本地写死的 `goodsData` 兜底和本地商家编辑/删除逻辑。
- 已改为通过 `options.id/options.accessoryId` 调用 `loadGoodsDetail()`，从 `/app/common/accessory/{accessoryId}` 读取真实详情。
- 用户端仍可加入购物车，商家端当前仅保留“返回商城”入口，避免误导为已打通商品维护能力。

### 本部分当前结论

从“配件商城页 + 配件详情页 + 配件接口封装”这个范围看，任务五主链路已经完成。

当前已经收口完成的能力：

1. 商城列表展示真实后端配件数据。
2. 搜索和分类筛选基于真实列表做前端过滤。
3. 详情页展示真实后端配件详情。
4. 商家本地商品编辑假逻辑已经清理，不再误导用户。
5. 购物车继续保留本地存储，符合当前任务范围。

当前仍需通过联调确认的内容：

- 分页拉取在真实数据量较大时的表现。
- 用户账号与商家账号下的页面展示差异。
- 图片地址在开发者工具、真机和不同网络环境下的加载稳定性。

### 二次复核补充（总规划师）

- 当前 `pages/accessoryMall/index.vue` 中仍保留 `initGoodsList`、`initGoodsData()`、`saveGoods()`、`deleteGoods()` 等旧本地逻辑，但页面主流程已经切到 `loadGoodsList()` + `getAccessoryList()`；由于 `merchantEditEnabled = false`，这些本地商家编辑入口不会在当前业务流程中启用。
- 当前 `pages/accessoryDetail/index.vue` 中仍保留旧的 `getGoodsDetail()` 本地兜底方法，以及本地 `allGoods` 编辑/删除逻辑，但页面主流程已经切到 `loadGoodsDetail()` + `getAccessoryDetail()`；同样由于 `merchantEditEnabled = false`，这些本地编辑逻辑不会影响当前真实详情展示。
- 当前前端真正映射并消费的后端字段主要是 `accessoryId`、`categoryName`、`accessoryName`、`accessoryDesc`、`coverImage`、`price`、`stock`、`status`；而页面模板里还保留了 `originalPrice` 这样的本地 UI 字段，但当前后端 `AppAccessory` 并没有这个参数，所以原价展示位通常不会从真实接口带出。
- `api/accessory.js` 当前支持 flat query 参数；如果后续要接入后端 Mapper 中的 `params.beginPrice`、`params.endPrice` 价格区间筛选，现有 `buildQuery()` 还不支持嵌套参数，需要单独补。
- `getAccessoryList()` 当前没有把 `categoryName`、`accessoryName` 直接走后端检索参数，页面仍然是“先拉真实列表，再在前端搜索和分类过滤”；这符合任务五当前目标，但不是后端分页检索模式。
- 二次复核结论：任务五的“浏览列表、详情查看、搜索筛选、商家编辑入口隐藏”已经完成；后续如需继续清理代码，可把这些未启用的旧本地方法作为代码清理项单独处理，但它们不构成当前任务五未完成。

### 二次复核补充已完成（2026-03-23）

- 已进一步清理 `E:\Backstage-Management-System\Hanzhong-Elec-AfterSale-Mini\pages\accessoryMall\index.vue`，删除未启用的本地兜底数据与本地商家编辑逻辑，包括 `initGoodsList`、`initGoodsData()`、`openEditPopup()`、`closeEditPopup()`、`validateForm()`、`saveGoods()`、`deleteGoods()`、`switchTab()` 及对应的编辑弹窗模板。
- 已进一步清理 `E:\Backstage-Management-System\Hanzhong-Elec-AfterSale-Mini\pages\accessoryDetail\index.vue`，删除旧的 `getGoodsDetail()` 本地详情兜底、商家编辑弹窗、本地 `allGoods` 编辑/删除逻辑；详情页当前只保留真实详情加载、图片兜底、返回商城、加入购物车。
- 配件商城当前主流程只保留 `getAccessoryList()` + `loadGoodsList()` + 前端搜索/分类过滤；配件详情当前主流程只保留 `getAccessoryDetail()` + `loadGoodsDetail()`。
- 二次复核补充中提到的“旧本地方法仍保留但未启用”问题，现已实际清理完成，不再只是通过开关隐藏。

### 三次复核补充（总规划师）

- 仍需补充一个很关键的分页边界：当前 `pages/accessoryMall/index.vue` 调用 `getAccessoryList()` 时没有显式传 `pageNum`、`pageSize`，而后端 `TableSupport` 的默认分页值是 `pageNum = 1`、`pageSize = 10`；因此当配件数据超过 10 条时，当前商城页实际上只会展示第一页数据，后续仍需补分页或滚动加载。
- 当前任务五的脚本和模板主逻辑已经清理到位，但样式层仍残留少量未使用选择器，例如 `pages/accessoryMall/index.vue` 中的 `.edit-popup`、`.goods-actions`、`.delete-btn`，以及 `pages/accessoryDetail/index.vue` 中的 `.edit-modal`、`.delete-btn`、`.original-price` 等；这些不会影响当前功能联调，但仍属于后续可继续收尾的样式清理项。

### 三次复核补充已完成（2026-03-23）

- 已修改 `E:\Backstage-Management-System\Hanzhong-Elec-AfterSale-Mini\pages\accessoryMall\index.vue`：新增 `PAGE_SIZE = 50`，并在 `loadGoodsList()` 中通过 `fetchAllAccessories()` 循环传入 `pageNum`、`pageSize` 拉取所有分页数据，避免后端默认 `pageSize = 10` 时只显示第一页。
- 配件商城页仍然保持“先拉真实列表，再做前端搜索和分类过滤”的模式，但当前过滤基于已完整拉取的真实列表执行，不再受默认第一页 10 条数据限制。
- 已继续清理 `E:\Backstage-Management-System\Hanzhong-Elec-AfterSale-Mini\pages\accessoryMall\index.vue` 的残余无用样式，删除了 `.add-btn`、`.original-price`、`.goods-actions`、`.edit-btn`、`.delete-btn`、`.empty-add-btn`、`.edit-popup`、`.popup-*`、`.form-*`、`.cancel-btn`、`.save-btn` 等已不再使用的选择器。
- 已继续清理 `E:\Backstage-Management-System\Hanzhong-Elec-AfterSale-Mini\pages\accessoryDetail\index.vue` 的残余无用样式，删除了 `.price-wrap`、`.original-price`、`.edit-btn`、`.merchant-tip`、`.delete-btn`、`.edit-modal`、`.modal-*`、`.form-*`、`.cancel-btn`、`.save-btn` 等已不再使用的选择器。
- 当前第三次复核补充中提出的两个问题都已处理：分页边界已补齐，样式层残留的无用选择器已完成收尾清理。

### 当前结论

任务五的完整实现、二次复核修正和三次复核修正都已经完成。

当前第五部分剩余的主要注意事项只有两类：

1. 真实设备联调验证仍未完成。
2. 配件列表分页拉取、搜索筛选和详情页展示需要分别在用户账号与商家账号下跑一遍完整闭环。

---
## 当前推荐执行顺序

如果新模型继续接手，推荐按下面顺序执行：

1. 先确认请求层已经统一
2. 再确认认证接口和登录注册完全打通
3. 再处理个人中心主页真实登录态
4. 再处理售后申请与售后订单
5. 最后处理配件商城与配件详情

只要前五部分完成，就已经形成了一个能让用户登录、提交售后、查询售后、浏览配件的可用小程序基础版本。