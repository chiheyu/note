
## 背景

- 小程序前端项目路径：`E:\Backstage-Management-System\Hanzhong-Elec-AfterSale-Mini`
- 后端项目路径：`E:\Backstage-Management-System\RuoYi-Vue-master`
- 当前状态：小程序前端页面可正常运行，但前后端接口尚未真正打通，部分页面仍在使用本地 `mock` 或本地缓存数据。

## 一、基础请求层

1. 统一后端地址配置。
2. 将 `config.js` 中的 `baseUrl` 改为你后端实际访问地址，例如：`http://你的局域网IP:8080`（）。
3. 不再使用线上演示地址 `https://vue.ruoyi.vip/prod-api`。
4. 修改 `utils/request.js`，不要再写死 `http://192.168.43.172:8080`。
5. `utils/request.js` 改为从 `config.js` 读取 `baseUrl`。
6. `utils/request.js` 的 `request` 方法签名改成接收配置对象，例如：`request({ url, method, data, headers })`。
7. 去掉 `X-Real-IP`、`X-Forwarded-For`、`Gateway-Request` 这几个请求头，先只保留常规请求头。
8. 保留 `Authorization: Bearer token` 头。
9. 成功响应时直接返回 `res.data`，不要过早只取某一个字段。

## 二、认证接口改造

1. 修改 `api/login.js` 中的接口路径。
2. 将登录接口从 `/login` 改为 `/app/auth/login`。
3. 将注册接口从 `/register` 改为 `/app/auth/register`。
4. 将获取当前用户信息接口从 `/getInfo` 改为 `/app/auth/profile`。
5. 先不要接退出登录接口，因为当前 App 控制器里没有明确看到 `/app/auth/logout`。

## 三、登录与注册参数对齐

1. 登录请求体改为：

```json
{
  "phone": "手机号",
  "password": "密码"
}
```

2. 注册请求体改为：

```json
{
  "phone": "手机号",
  "password": "密码",
  "confirmPassword": "确认密码",
  "nickName": "昵称",
  "code": "验证码",
  "roleType": "角色值"
}
```

3. 如果注册的是普通用户，`roleType` 使用 `"1"`。
4. 如果注册的是商家，`roleType` 使用 `"2"`。
5. 前端当前的 `role === 'user'`、`role === 'merchant'` 只是页面层逻辑，发给后端时要映射为字符串数字角色值。

## 四、去掉前端 Mock 登录/注册

1. 修改 `pages/profile/login.vue`。
2. 删除或停用 `mockLogin` 逻辑。
3. 删除或停用 `mockRegister` 逻辑。
4. 改为调用 `api/login.js` 中的真实接口。
5. 登录成功后，不再自己生成 `mock_token_xxx`。
6. 登录成功后，从后端响应中取真实 `token`。

## 五、登录成功后的数据落库方式

1. 后端 App 登录返回的不是单独 token 字符串，而是一个对象。
2. 需要从响应里读取：
3. `data.token`
4. `data.roleType`
5. `data.appUser`
6. `data.merchant`
7. 建议本地缓存保存方式：

```js
uni.setStorageSync('token', res.data.token)
uni.setStorageSync('userInfo', {
  ...res.data.appUser,
  roleType: res.data.roleType,
  merchant: res.data.merchant || null,
  role: res.data.roleType === '2' ? 'merchant' : 'user'
})
```

## 六、个人中心页面改造

1. 修改 `pages/profile/index.vue`。
2. 去掉页面内部通过本地 `userList` 做登录校验的逻辑。
3. 去掉本地生成 `mock_token` 的逻辑。
4. 页面展示时，改为优先读取真实登录后的 `userInfo`。
5. 商家判断逻辑建议统一改为：

```js
const isMerchant = userInfo && userInfo.roleType === '2'
```

## 七、售后申请改造

1. 修改 `pages/applyAfterSale/index.vue`。
2. 不再把售后申请写入本地 `afterSaleOrders`。
3. 改为调用后端接口：`POST /app/user/afterSalesOrder`。
4. 当前页面字段和后端字段不完全一致，建议映射为：

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

5. 提交成功后再跳转到售后订单页。

## 八、售后订单列表改造

1. 修改 `pages/afterSaleOrder/index.vue`。
2. 不再从本地 `afterSaleOrders` 读取列表。
3. 普通用户列表改为调用：`GET /app/user/afterSalesOrder/list`。
4. 商家待处理列表改为调用：`GET /app/merchant/order/pendingList`。
5. 商家全部订单列表改为调用：`GET /app/merchant/order/list`。
6. 用户取消售后单改为调用：`PUT /app/user/afterSalesOrder/cancel/{orderId}`。
7. 商家接单改为调用：`PUT /app/merchant/order/take/{orderId}`。
8. 商家更新状态改为调用：`PUT /app/merchant/order/status`。

## 九、售后状态值同步后端

1. 前端当前状态值逻辑与后端不一致，需要统一。
2. 后端售后状态值如下：
3. `0`：待接单
4. `1`：已接单
5. `2`：维修中
6. `3`：已完成
7. `4`：已取消
8. 修改 `pages/afterSaleOrder/index.vue` 中的状态文案、筛选逻辑、按钮显示逻辑。
9. 删除当前页面里 `-1`、`-2` 这种本地状态约定。

## 十、配件商城浏览接口改造

1. 修改 `pages/accessoryMall/index.vue`。
2. 不再优先读取本地 `goodsList`。
3. 改为调用后端公共接口：`GET /app/common/accessory/list`。
4. 列表接口返回的是分页结构，前端应从 `rows` 取数组数据。
5. 搜索、分类筛选可以继续在前端对接口返回结果做二次处理。

## 十一、配件详情接口改造

1. 修改 `pages/accessoryDetail/index.vue`。
2. 不再使用页面内写死的商品详情映射。
3. 改为调用后端接口：`GET /app/common/accessory/{accessoryId}`。

## 十二、购物车与下单

1. 购物车数据当前可以暂时继续保留本地存储。
2. 修改 `pages/cart/index.vue` 的“结算”逻辑。
3. 不再只做本地“支付成功”提示。
4. 改为遍历购物车商品，调用后端配件下单接口：`POST /app/user/accessoryOrder`。
5. 请求体字段需要按后端结构组织：

```json
{
  "accessoryId": 1,
  "quantity": 2,
  "receiverName": "收货人",
  "receiverPhone": "手机号",
  "receiverAddress": "完整地址",
  "orderRemark": "备注"
}
```

6. 收货地址可以先从本地默认地址中读取后拼接提交。

## 十三、商家商品管理先暂缓

1. 当前前端把商品新增、编辑、删除开放给 `merchant`。
2. 但后端现有配件新增、编辑、删除接口在 `AppAdminController`，不是 `AppMerchantController`。
3. 这意味着“商家管理商品”这块暂时不是简单改前端就能打通。
4. 在后端权限和业务归属没明确前，建议先保留本地逻辑，或者先隐藏商家商品增删改入口。

## 十四、地址管理先暂缓

1. 当前 `pages/address/index.vue` 使用本地 `addressList`。
2. 我没有在 App 控制器中查到地址管理接口。
3. 所以地址页暂时建议继续保留本地实现。
4. 等后端补充地址接口后再联调。

## 十五、建议的实际执行顺序

1. 先改 `config.js` 和 `utils/request.js`。
2. 再改 `api/login.js`。
3. 再改 `pages/profile/login.vue` 和 `pages/profile/index.vue`。
4. 然后打通 `applyAfterSale` 和 `afterSaleOrder`。
5. 再打通 `accessoryMall` 和 `accessoryDetail`。
6. 最后再考虑 `cart` 下单逻辑。

## 十六、当前最小可验证闭环

1. 登录
2. 获取当前用户信息
3. 提交售后申请
4. 查看售后订单列表

如果这 4 步能打通，就说明小程序与后端的基础联调已经成立。

