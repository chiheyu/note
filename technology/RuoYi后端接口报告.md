# RuoYi 后端接口报告

# 一、报告概述

本文档为 `RuoYi-Vue-master` 后端项目接口扫描报告，格式参考同目录下的《后端接口报告.md》模板生成。报告依据后端源码中的 Controller 路由注解、权限注解、Spring Security 配置、应用配置和 App 业务实体字段整理，用于前后端联调、接口验收和后续维护。

|项目|内容|
|---|---|
|项目名称|RuoYi / 若依管理系统，含汉中售后 App 业务扩展|
|报告版本|V1.0|
|报告生成时间|2026-04-10 16:40|
|源码目录|`E:\Backstage-Management-System\RuoYi-Vue-master`|
|接口统计口径|扫描 `*Controller.java` 中的 `@GetMapping`、`@PostMapping`、`@PutMapping`、`@DeleteMapping`、方法级 `@RequestMapping`|
|接口总数|193 个 URL 映射；包含同一方法多 URL 的别名映射|
|测试状态|本报告执行静态源码扫描，未启动服务做真实 HTTP 联调|

# 二、接口基本信息

|序号|接口模块|接口数量（个）|接口类型|部署环境|接口基础 URL|备注|
|---|---|---:|---|---|---|---|
|1|App移动端/售后业务|52|RESTful/HTTP|开发环境|http://localhost:8080/app|移动端注册登录、用户售后单、商家接单、配件与评价等业务接口|
|2|系统认证与首页|5|RESTful/HTTP|开发环境|http://localhost:8080|后台登录、用户信息、路由、注册和首页接口|
|3|系统管理|81|RESTful/HTTP|开发环境|http://localhost:8080/system|用户、角色、菜单、部门、岗位、字典、参数、通知、个人资料等后台管理接口|
|4|系统监控|32|RESTful/HTTP|开发环境|http://localhost:8080/monitor|缓存、服务状态、登录日志、操作日志、在线用户、定时任务与任务日志接口|
|5|系统工具|18|RESTful/HTTP|开发环境|http://localhost:8080/tool、http://localhost:8080/test|代码生成与 Swagger 示例测试接口|
|6|公共能力|5|RESTful/HTTP|开发环境|http://localhost:8080/common、http://localhost:8080/captchaImage|验证码、文件上传下载、公开 App 查询接口|

补充说明：后端端口来自 `ruoyi-admin/src/main/resources/application.yml`，当前 `server.port=8080`、`server.servlet.context-path=/`；Swagger 配置中 `pathMapping=/dev-api`，通常用于前端代理或文档展示前缀。

鉴权说明：`token.header=Authorization`，`expireTime=30` 分钟。Spring Security 明确放行 `/login`、`/register`、`/captchaImage`、`/app/auth/sendCode`、`/app/auth/register`、`/app/auth/login`、`/app/common/**`、静态资源、Swagger 和 Druid 路径；其他请求默认需要登录 token。带 `@PreAuthorize` 的接口还需满足权限码或角色校验。

环境依赖：MySQL 数据源使用 `localhost:3306/ruoyi-data`，Redis 使用 `localhost:6379`。数据库账号、密码、JWT secret 等敏感配置在报告中不展开，提交或共享前建议改为环境变量/外部配置。

# 三、接口详细说明（按模块划分）

## 3.1 统一请求与响应约定

|项目|说明|
|---|---|
|请求头|登录后接口统一携带 `Authorization: <token>`；JSON 请求使用 `Content-Type: application/json`；上传接口使用 `multipart/form-data`|
|分页接口|继承 `BaseController` 的列表接口通常支持 `pageNum`、`pageSize` 等分页参数，并返回 `TableDataInfo`|
|普通响应|多数接口返回 `AjaxResult`，常见结构为 `code`、`msg`、`data`|
|列表响应|分页列表返回 `TableDataInfo`，常见结构为 `code`、`msg`、`rows`、`total`|
|导出/下载响应|导出、模板下载、代码下载、文件下载接口直接写入 `HttpServletResponse`，响应体为文件流|
|异常响应|未登录通常返回 401；无权限返回 403 或业务封装错误；参数校验、业务异常由全局异常处理和 `AjaxResult.error` 返回|

## 3.2 App 业务实体与请求体字段概要

|类名|字段概要|
|---|---|
|AppAccessory|Long accessoryId, String categoryName, String accessoryName, String accessoryDesc, String coverImage, BigDecimal price, Long stock, Long salesCount, Long merchantId, String status, Boolean collected|
|AppAccessoryCollection|Long collectionId, Long appUserId, Long accessoryId, String userName, String accessoryName, String coverImage, BigDecimal price|
|AppAccessoryOrder|Long accessoryOrderId, String orderNo, Long accessoryId, Long appUserId, Long merchantId, Long quantity, BigDecimal price, BigDecimal totalAmount, String status, String receiverName, String receiverPhone, String receiverAddress, String orderRemark, String accessoryName, String coverImage, String userName, String merchantName, Boolean reviewed|
|AppAccessoryOrderSubmitBody|Long accessoryId, Long quantity, String receiverName, String receiverPhone, String receiverAddress, String orderRemark|
|AppAfterSalesOrder|Long orderId, String orderNo, Long appUserId, Long merchantId, String productType, String faultDesc, String faultImages, String status, String serviceRemark, String contactName, String contactPhone, String address, Date acceptTime, Date finishTime, String userName, String merchantName|
|AppLoginBody|String phone, String password|
|AppLoginVo|String token, String roleType, AppUser appUser, AppMerchant merchant|
|AppMerchant|Long merchantId, Long appUserId, Long sysUserId, String merchantName, String contactName, String contactPhone, String address, String serviceScope, String merchantDesc, String cityName, String auditStatus, String userName|
|AppMerchantAuditBody|Long merchantId, String auditStatus, String auditRemark|
|AppMerchantReview|Long reviewId, Long merchantId, Long accessoryOrderId, Long appUserId, Integer rating, String reviewContent, String userName, String avatar, String accessoryName|
|AppMerchantReviewSubmitBody|Long accessoryOrderId, Integer rating, String reviewContent|
|AppOrderActionBody|Long orderId, String status, String serviceRemark|
|AppRegisterBody|String phone, String password, String confirmPassword, String nickName, String code, String roleType, String merchantName, String contactName, String address, String serviceScope|
|AppUser|Long appUserId, Long sysUserId, String phone, String nickName, String roleType, Long merchantId, String status, String lastSmsCode, Date lastSmsExpireTime, String userName, String merchantName|

## 3.3 接口清单

### App移动端/售后业务

|序号|Controller|请求方式|接口 URL|功能说明|请求/参数来源|响应类型|鉴权/权限|
|---:|---|---|---|---|---|---|---|
|1|AppAdminController|POST|/app/admin/accessory|新增/创建数据|@RequestBody AppAccessory accessory|AjaxResult|权限码：app:accessory:add|
|2|AppAdminController|PUT|/app/admin/accessory|修改状态或业务数据|@RequestBody AppAccessory accessory|AjaxResult|权限码：app:accessory:edit|
|3|AppAdminController|DELETE|/app/admin/accessory/{accessoryId}|删除/清理/取消数据|@PathVariable Long accessoryId|AjaxResult|权限码：app:accessory:remove|
|4|AppAdminController|GET|/app/admin/accessory/{accessoryId}|查询详情|@PathVariable Long accessoryId|AjaxResult|权限码：app:accessory:query|
|5|AppAdminController|GET|/app/admin/accessory/list|查询列表/树/选项数据|AppAccessory accessory|TableDataInfo|权限码：app:accessory:list|
|6|AppAdminController|GET|/app/admin/accessoryOrder/{accessoryOrderId}|查询详情|@PathVariable Long accessoryOrderId|AjaxResult|权限码：app:accessoryOrder:query|
|7|AppAdminController|GET|/app/admin/accessoryOrder/list|查询列表/树/选项数据|AppAccessoryOrder order|TableDataInfo|权限码：app:accessoryOrder:list|
|8|AppAdminController|GET|/app/admin/afterSalesOrder/{orderId}|查询详情|@PathVariable Long orderId|AjaxResult|权限码：app:afterSalesOrder:query|
|9|AppAdminController|GET|/app/admin/afterSalesOrder/list|查询列表/树/选项数据|AppAfterSalesOrder order|TableDataInfo|权限码：app:afterSalesOrder:list|
|10|AppAdminController|PUT|/app/admin/afterSalesOrder/status|修改状态或业务数据|@RequestBody AppOrderActionBody actionBody|AjaxResult|权限码：app:afterSalesOrder:edit|
|11|AppAdminController|PUT|/app/admin/merchant|修改状态或业务数据|@RequestBody AppMerchant merchant|AjaxResult|权限码：app:merchant:edit|
|12|AppAdminController|GET|/app/admin/merchant/{merchantId}|查询详情|@PathVariable Long merchantId|AjaxResult|权限码：app:merchant:query|
|13|AppAdminController|PUT|/app/admin/merchant/audit|修改状态或业务数据|@RequestBody AppMerchantAuditBody auditBody|AjaxResult|权限码：app:merchant:audit|
|14|AppAdminController|GET|/app/admin/merchant/list|查询列表/树/选项数据|AppMerchant merchant|TableDataInfo|权限码：app:merchant:list|
|15|AppAuthController|POST|/app/auth/login|登录并返回访问凭证/用户信息|@RequestBody AppLoginBody loginBody|AjaxResult|匿名放行|
|16|AppAuthController|POST|/app/auth/logout|退出登录|HttpServletRequest request|AjaxResult|需登录 token|
|17|AppAuthController|GET|/app/auth/profile|查询或维护当前用户资料||AjaxResult|需登录 token|
|18|AppAuthController|POST|/app/auth/register|注册账号|@RequestBody AppRegisterBody registerBody|AjaxResult|匿名放行|
|19|AppAuthController|GET|/app/auth/sendCode|发送短信验证码|String phone|AjaxResult|匿名放行|
|20|AppCommonController|GET|/app/common/accessory/{accessoryId}|查询详情|@PathVariable Long accessoryId|AjaxResult|匿名放行|
|21|AppCommonController|GET|/app/common/accessory/list|查询列表/树/选项数据|AppAccessory accessory|TableDataInfo|匿名放行|
|22|AppCommonController|GET|/app/common/merchant/{merchantId}|查询详情|@PathVariable Long merchantId|AjaxResult|匿名放行|
|23|AppCommonController|GET|/app/common/merchant/{merchantId}/review/list|查询列表/树/选项数据|@PathVariable Long merchantId|AjaxResult|匿名放行|
|24|AppCommonController|GET|/app/common/merchant/list|查询列表/树/选项数据|AppMerchant merchant|TableDataInfo|匿名放行|
|25|AppMerchantController|POST|/app/merchant/accessory|新增/创建数据|@RequestBody AppAccessory accessory|AjaxResult|角色：merchant|
|26|AppMerchantController|PUT|/app/merchant/accessory|修改状态或业务数据|@RequestBody AppAccessory accessory|AjaxResult|角色：merchant|
|27|AppMerchantController|DELETE|/app/merchant/accessory/{accessoryId}|删除/清理/取消数据|@PathVariable Long accessoryId|AjaxResult|角色：merchant|
|28|AppMerchantController|GET|/app/merchant/accessory/{accessoryId}|查询详情|@PathVariable Long accessoryId|AjaxResult|角色：merchant|
|29|AppMerchantController|GET|/app/merchant/accessory/list|查询列表/树/选项数据|AppAccessory accessory|TableDataInfo|角色：merchant|
|30|AppMerchantController|PUT|/app/merchant/accessoryOrder/cancel/{accessoryOrderId}|删除/清理/取消数据|@PathVariable Long accessoryOrderId|AjaxResult|角色：merchant|
|31|AppMerchantController|PUT|/app/merchant/accessoryOrder/complete/{accessoryOrderId}|执行业务动作|@PathVariable Long accessoryOrderId|AjaxResult|角色：merchant|
|32|AppMerchantController|GET|/app/merchant/accessoryOrder/list|查询列表/树/选项数据|AppAccessoryOrder order|TableDataInfo|角色：merchant|
|33|AppMerchantController|GET|/app/merchant/accessoryOrder/pendingList|查询列表/树/选项数据|AppAccessoryOrder order|TableDataInfo|角色：merchant|
|34|AppMerchantController|PUT|/app/merchant/accessoryOrder/ship/{accessoryOrderId}|执行业务动作|@PathVariable Long accessoryOrderId|AjaxResult|角色：merchant|
|35|AppMerchantController|GET|/app/merchant/info|详见 Controller 方法实现||AjaxResult|角色：merchant|
|36|AppMerchantController|PUT|/app/merchant/info|修改状态或业务数据|@RequestBody AppMerchant merchant|AjaxResult|角色：merchant|
|37|AppMerchantController|GET|/app/merchant/order/list|查询列表/树/选项数据|AppAfterSalesOrder order|TableDataInfo|角色：merchant|
|38|AppMerchantController|GET|/app/merchant/order/pendingList|查询列表/树/选项数据|AppAfterSalesOrder order|TableDataInfo|角色：merchant|
|39|AppMerchantController|PUT|/app/merchant/order/status|修改状态或业务数据|@RequestBody AppOrderActionBody actionBody|AjaxResult|角色：merchant|
|40|AppMerchantController|PUT|/app/merchant/order/take/{orderId}|执行业务动作|@PathVariable Long orderId|AjaxResult|角色：merchant|
|41|AppMerchantController|GET|/app/merchant/stats|详见 Controller 方法实现||AjaxResult|角色：merchant|
|42|AppUserController|DELETE|/app/user/accessoryCollection/{accessoryId}|新增/创建数据|@PathVariable Long accessoryId|AjaxResult|角色：user|
|43|AppUserController|POST|/app/user/accessoryCollection/{accessoryId}|新增/创建数据|@PathVariable Long accessoryId|AjaxResult|角色：user|
|44|AppUserController|GET|/app/user/accessoryCollection/list|查询列表/树/选项数据|AppAccessoryCollection collection|TableDataInfo|角色：user|
|45|AppUserController|POST|/app/user/accessoryOrder|新增/创建数据|@RequestBody AppAccessoryOrderSubmitBody submitBody|AjaxResult|角色：user|
|46|AppUserController|GET|/app/user/accessoryOrder/{accessoryOrderId}|查询详情|@PathVariable Long accessoryOrderId|AjaxResult|角色：user|
|47|AppUserController|GET|/app/user/accessoryOrder/list|查询列表/树/选项数据|AppAccessoryOrder order|TableDataInfo|角色：user|
|48|AppUserController|POST|/app/user/afterSalesOrder|新增/创建数据|@RequestBody AppAfterSalesOrder order|AjaxResult|角色：user|
|49|AppUserController|GET|/app/user/afterSalesOrder/{orderId}|查询详情|@PathVariable Long orderId|AjaxResult|角色：user|
|50|AppUserController|PUT|/app/user/afterSalesOrder/cancel/{orderId}|删除/清理/取消数据|@PathVariable Long orderId|AjaxResult|角色：user|
|51|AppUserController|GET|/app/user/afterSalesOrder/list|查询列表/树/选项数据|AppAfterSalesOrder order|TableDataInfo|角色：user|
|52|AppUserController|POST|/app/user/merchantReview|新增/创建数据|@RequestBody AppMerchantReviewSubmitBody submitBody|AjaxResult|角色：user|

### 系统认证与首页

|序号|Controller|请求方式|接口 URL|功能说明|请求/参数来源|响应类型|鉴权/权限|
|---:|---|---|---|---|---|---|---|
|1|SysIndexController|ANY|/|详见 Controller 方法实现||String|匿名放行/静态资源规则|
|2|SysLoginController|GET|/getInfo|查询详情||AjaxResult|需登录 token|
|3|SysLoginController|GET|/getRouters|查询列表/树/选项数据||AjaxResult|需登录 token|
|4|SysLoginController|POST|/login|登录并返回访问凭证/用户信息|@RequestBody LoginBody loginBody|AjaxResult|匿名放行|
|5|SysRegisterController|POST|/register|注册账号|@RequestBody RegisterBody user|AjaxResult|匿名放行|

### 系统管理

|序号|Controller|请求方式|接口 URL|功能说明|请求/参数来源|响应类型|鉴权/权限|
|---:|---|---|---|---|---|---|---|
|1|SysConfigController|POST|/system/config|新增/创建数据|@Validated @RequestBody SysConfig config|AjaxResult|权限码：system:config:add|
|2|SysConfigController|PUT|/system/config|修改状态或业务数据|@Validated @RequestBody SysConfig config|AjaxResult|权限码：system:config:edit|
|3|SysConfigController|GET|/system/config/{configId}|查询详情|@PathVariable Long configId|AjaxResult|权限码：system:config:query|
|4|SysConfigController|DELETE|/system/config/{configIds}|删除/清理/取消数据|@PathVariable Long[] configIds|AjaxResult|权限码：system:config:remove|
|5|SysConfigController|GET|/system/config/configKey/{configKey}|详见 Controller 方法实现|@PathVariable String configKey|AjaxResult|需登录 token|
|6|SysConfigController|POST|/system/config/export|导出数据|HttpServletResponse response, SysConfig config|void|权限码：system:config:export|
|7|SysConfigController|GET|/system/config/list|查询列表/树/选项数据|SysConfig config|TableDataInfo|权限码：system:config:list|
|8|SysConfigController|DELETE|/system/config/refreshCache|详见 Controller 方法实现||AjaxResult|权限码：system:config:remove|
|9|SysDeptController|POST|/system/dept|新增/创建数据|@Validated @RequestBody SysDept dept|AjaxResult|权限码：system:dept:add|
|10|SysDeptController|PUT|/system/dept|修改状态或业务数据|@Validated @RequestBody SysDept dept|AjaxResult|权限码：system:dept:edit|
|11|SysDeptController|DELETE|/system/dept/{deptId}|删除/清理/取消数据|@PathVariable Long deptId|AjaxResult|权限码：system:dept:remove|
|12|SysDeptController|GET|/system/dept/{deptId}|查询详情|@PathVariable Long deptId|AjaxResult|权限码：system:dept:query|
|13|SysDeptController|GET|/system/dept/list|查询列表/树/选项数据|SysDept dept|AjaxResult|权限码：system:dept:list|
|14|SysDeptController|GET|/system/dept/list/exclude/{deptId}|详见 Controller 方法实现|@PathVariable(value = "deptId", required = false) Long deptId|AjaxResult|权限码：system:dept:list|
|15|SysDictDataController|POST|/system/dict/data|新增/创建数据|@Validated @RequestBody SysDictData dict|AjaxResult|权限码：system:dict:add|
|16|SysDictDataController|PUT|/system/dict/data|修改状态或业务数据|@Validated @RequestBody SysDictData dict|AjaxResult|权限码：system:dict:edit|
|17|SysDictDataController|GET|/system/dict/data/{dictCode}|查询详情|@PathVariable Long dictCode|AjaxResult|权限码：system:dict:query|
|18|SysDictDataController|DELETE|/system/dict/data/{dictCodes}|删除/清理/取消数据|@PathVariable Long[] dictCodes|AjaxResult|权限码：system:dict:remove|
|19|SysDictDataController|POST|/system/dict/data/export|导出数据|HttpServletResponse response, SysDictData dictData|void|权限码：system:dict:export|
|20|SysDictDataController|GET|/system/dict/data/list|查询列表/树/选项数据|SysDictData dictData|TableDataInfo|权限码：system:dict:list|
|21|SysDictDataController|GET|/system/dict/data/type/{dictType}|详见 Controller 方法实现|@PathVariable String dictType|AjaxResult|需登录 token|
|22|SysDictTypeController|POST|/system/dict/type|新增/创建数据|@Validated @RequestBody SysDictType dict|AjaxResult|权限码：system:dict:add|
|23|SysDictTypeController|PUT|/system/dict/type|修改状态或业务数据|@Validated @RequestBody SysDictType dict|AjaxResult|权限码：system:dict:edit|
|24|SysDictTypeController|GET|/system/dict/type/{dictId}|查询详情|@PathVariable Long dictId|AjaxResult|权限码：system:dict:query|
|25|SysDictTypeController|DELETE|/system/dict/type/{dictIds}|删除/清理/取消数据|@PathVariable Long[] dictIds|AjaxResult|权限码：system:dict:remove|
|26|SysDictTypeController|POST|/system/dict/type/export|导出数据|HttpServletResponse response, SysDictType dictType|void|权限码：system:dict:export|
|27|SysDictTypeController|GET|/system/dict/type/list|查询列表/树/选项数据|SysDictType dictType|TableDataInfo|权限码：system:dict:list|
|28|SysDictTypeController|GET|/system/dict/type/optionselect|查询列表/树/选项数据||AjaxResult|需登录 token|
|29|SysDictTypeController|DELETE|/system/dict/type/refreshCache|详见 Controller 方法实现||AjaxResult|权限码：system:dict:remove|
|30|SysMenuController|POST|/system/menu|新增/创建数据|@Validated @RequestBody SysMenu menu|AjaxResult|权限码：system:menu:add|
|31|SysMenuController|PUT|/system/menu|修改状态或业务数据|@Validated @RequestBody SysMenu menu|AjaxResult|权限码：system:menu:edit|
|32|SysMenuController|DELETE|/system/menu/{menuId}|删除/清理/取消数据|@PathVariable("menuId") Long menuId|AjaxResult|权限码：system:menu:remove|
|33|SysMenuController|GET|/system/menu/{menuId}|查询详情|@PathVariable Long menuId|AjaxResult|权限码：system:menu:query|
|34|SysMenuController|GET|/system/menu/list|查询列表/树/选项数据|SysMenu menu|AjaxResult|权限码：system:menu:list|
|35|SysMenuController|GET|/system/menu/roleMenuTreeselect/{roleId}|查询列表/树/选项数据|@PathVariable("roleId") Long roleId|AjaxResult|需登录 token|
|36|SysMenuController|GET|/system/menu/treeselect|查询列表/树/选项数据|SysMenu menu|AjaxResult|需登录 token|
|37|SysNoticeController|POST|/system/notice|新增/创建数据|@Validated @RequestBody SysNotice notice|AjaxResult|权限码：system:notice:add|
|38|SysNoticeController|PUT|/system/notice|修改状态或业务数据|@Validated @RequestBody SysNotice notice|AjaxResult|权限码：system:notice:edit|
|39|SysNoticeController|GET|/system/notice/{noticeId}|查询详情|@PathVariable Long noticeId|AjaxResult|权限码：system:notice:query|
|40|SysNoticeController|DELETE|/system/notice/{noticeIds}|删除/清理/取消数据|@PathVariable Long[] noticeIds|AjaxResult|权限码：system:notice:remove|
|41|SysNoticeController|GET|/system/notice/list|查询列表/树/选项数据|SysNotice notice|TableDataInfo|权限码：system:notice:list|
|42|SysPostController|POST|/system/post|新增/创建数据|@Validated @RequestBody SysPost post|AjaxResult|权限码：system:post:add|
|43|SysPostController|PUT|/system/post|修改状态或业务数据|@Validated @RequestBody SysPost post|AjaxResult|权限码：system:post:edit|
|44|SysPostController|GET|/system/post/{postId}|查询详情|@PathVariable Long postId|AjaxResult|权限码：system:post:query|
|45|SysPostController|DELETE|/system/post/{postIds}|删除/清理/取消数据|@PathVariable Long[] postIds|AjaxResult|权限码：system:post:remove|
|46|SysPostController|POST|/system/post/export|导出数据|HttpServletResponse response, SysPost post|void|权限码：system:post:export|
|47|SysPostController|GET|/system/post/list|查询列表/树/选项数据|SysPost post|TableDataInfo|权限码：system:post:list|
|48|SysPostController|GET|/system/post/optionselect|查询列表/树/选项数据||AjaxResult|需登录 token|
|49|SysProfileController|GET|/system/user/profile|查询或维护当前用户资料||AjaxResult|需登录 token|
|50|SysProfileController|PUT|/system/user/profile|查询或维护当前用户资料|@RequestBody SysUser user|AjaxResult|需登录 token|
|51|SysProfileController|POST|/system/user/profile/avatar|上传文件或头像|@RequestParam("avatarfile") MultipartFile file|AjaxResult|需登录 token|
|52|SysProfileController|PUT|/system/user/profile/updatePwd|修改状态或业务数据|@RequestBody Map<String, String> params|AjaxResult|需登录 token|
|53|SysRoleController|POST|/system/role|新增/创建数据|@Validated @RequestBody SysRole role|AjaxResult|权限码：system:role:add|
|54|SysRoleController|PUT|/system/role|修改状态或业务数据|@Validated @RequestBody SysRole role|AjaxResult|权限码：system:role:edit|
|55|SysRoleController|GET|/system/role/{roleId}|查询详情|@PathVariable Long roleId|AjaxResult|权限码：system:role:query|
|56|SysRoleController|DELETE|/system/role/{roleIds}|删除/清理/取消数据|@PathVariable Long[] roleIds|AjaxResult|权限码：system:role:remove|
|57|SysRoleController|GET|/system/role/authUser/allocatedList|查询列表/树/选项数据|SysUser user|TableDataInfo|权限码：system:role:list|
|58|SysRoleController|PUT|/system/role/authUser/cancel|删除/清理/取消数据|@RequestBody SysUserRole userRole|AjaxResult|权限码：system:role:edit|
|59|SysRoleController|PUT|/system/role/authUser/cancelAll|删除/清理/取消数据|Long roleId, Long[] userIds|AjaxResult|权限码：system:role:edit|
|60|SysRoleController|PUT|/system/role/authUser/selectAll|查询列表/树/选项数据|Long roleId, Long[] userIds|AjaxResult|权限码：system:role:edit|
|61|SysRoleController|GET|/system/role/authUser/unallocatedList|查询列表/树/选项数据|SysUser user|TableDataInfo|权限码：system:role:list|
|62|SysRoleController|PUT|/system/role/changeStatus|修改状态或业务数据|@RequestBody SysRole role|AjaxResult|权限码：system:role:edit|
|63|SysRoleController|PUT|/system/role/dataScope|修改状态或业务数据|@RequestBody SysRole role|AjaxResult|权限码：system:role:edit|
|64|SysRoleController|GET|/system/role/deptTree/{roleId}|查询列表/树/选项数据|@PathVariable("roleId") Long roleId|AjaxResult|权限码：system:role:query|
|65|SysRoleController|POST|/system/role/export|导出数据|HttpServletResponse response, SysRole role|void|权限码：system:role:export|
|66|SysRoleController|GET|/system/role/list|查询列表/树/选项数据|SysRole role|TableDataInfo|权限码：system:role:list|
|67|SysRoleController|GET|/system/role/optionselect|查询列表/树/选项数据||AjaxResult|权限码：system:role:query|
|68|SysUserController|GET|/system/user|查询详情|@PathVariable(value = "userId", required = false) Long userId|AjaxResult|权限码：system:user:query|
|69|SysUserController|POST|/system/user|新增/创建数据|@Validated @RequestBody SysUser user|AjaxResult|权限码：system:user:add|
|70|SysUserController|PUT|/system/user|修改状态或业务数据|@Validated @RequestBody SysUser user|AjaxResult|权限码：system:user:edit|
|71|SysUserController|GET|/system/user/{userId}|查询详情|@PathVariable(value = "userId", required = false) Long userId|AjaxResult|权限码：system:user:query|
|72|SysUserController|DELETE|/system/user/{userIds}|删除/清理/取消数据|@PathVariable Long[] userIds|AjaxResult|权限码：system:user:remove|
|73|SysUserController|PUT|/system/user/authRole|修改状态或业务数据|Long userId, Long[] roleIds|AjaxResult|权限码：system:user:edit|
|74|SysUserController|GET|/system/user/authRole/{userId}|修改状态或业务数据|@PathVariable("userId") Long userId|AjaxResult|权限码：system:user:query|
|75|SysUserController|PUT|/system/user/changeStatus|修改状态或业务数据|@RequestBody SysUser user|AjaxResult|权限码：system:user:edit|
|76|SysUserController|GET|/system/user/deptTree|查询列表/树/选项数据|SysDept dept|AjaxResult|权限码：system:user:list|
|77|SysUserController|POST|/system/user/export|导出数据|HttpServletResponse response, SysUser user|void|权限码：system:user:export|
|78|SysUserController|POST|/system/user/importData|导入数据|MultipartFile file, boolean updateSupport|AjaxResult|权限码：system:user:import|
|79|SysUserController|POST|/system/user/importTemplate|下载文件/模板/代码包|HttpServletResponse response|void|需登录 token|
|80|SysUserController|GET|/system/user/list|查询列表/树/选项数据|SysUser user|TableDataInfo|权限码：system:user:list|
|81|SysUserController|PUT|/system/user/resetPwd|修改状态或业务数据|@RequestBody SysUser user|AjaxResult|权限码：system:user:resetPwd|

### 系统监控

|序号|Controller|请求方式|接口 URL|功能说明|请求/参数来源|响应类型|鉴权/权限|
|---:|---|---|---|---|---|---|---|
|1|CacheController|GET|/monitor/cache|查询详情||AjaxResult|权限码：monitor:cache:list|
|2|CacheController|DELETE|/monitor/cache/clearCacheAll|删除/清理/取消数据||AjaxResult|权限码：monitor:cache:list|
|3|CacheController|DELETE|/monitor/cache/clearCacheKey/{cacheKey}|删除/清理/取消数据|@PathVariable String cacheKey|AjaxResult|权限码：monitor:cache:list|
|4|CacheController|DELETE|/monitor/cache/clearCacheName/{cacheName}|删除/清理/取消数据|@PathVariable String cacheName|AjaxResult|权限码：monitor:cache:list|
|5|CacheController|GET|/monitor/cache/getKeys/{cacheName}|查询列表/树/选项数据|@PathVariable String cacheName|AjaxResult|权限码：monitor:cache:list|
|6|CacheController|GET|/monitor/cache/getNames|详见 Controller 方法实现||AjaxResult|权限码：monitor:cache:list|
|7|CacheController|GET|/monitor/cache/getValue/{cacheName}/{cacheKey}|查询详情|@PathVariable String cacheName, @PathVariable String cacheKey|AjaxResult|权限码：monitor:cache:list|
|8|ServerController|GET|/monitor/server|查询详情||AjaxResult|权限码：monitor:server:list|
|9|SysJobController|POST|/monitor/job|新增/创建数据|@RequestBody SysJob job|AjaxResult|权限码：monitor:job:add|
|10|SysJobController|PUT|/monitor/job|修改状态或业务数据|@RequestBody SysJob job|AjaxResult|权限码：monitor:job:edit|
|11|SysJobController|GET|/monitor/job/{jobId}|查询详情|@PathVariable("jobId") Long jobId|AjaxResult|权限码：monitor:job:query|
|12|SysJobController|DELETE|/monitor/job/{jobIds}|删除/清理/取消数据|@PathVariable Long[] jobIds|AjaxResult|权限码：monitor:job:remove|
|13|SysJobController|PUT|/monitor/job/changeStatus|修改状态或业务数据|@RequestBody SysJob job|AjaxResult|权限码：monitor:job:changeStatus|
|14|SysJobController|POST|/monitor/job/export|导出数据|HttpServletResponse response, SysJob sysJob|void|权限码：monitor:job:export|
|15|SysJobController|GET|/monitor/job/list|查询列表/树/选项数据|SysJob sysJob|TableDataInfo|权限码：monitor:job:list|
|16|SysJobController|PUT|/monitor/job/run|执行业务动作|@RequestBody SysJob job|AjaxResult|权限码：monitor:job:changeStatus|
|17|SysJobLogController|GET|/monitor/jobLog/{jobLogId}|查询详情|@PathVariable Long jobLogId|AjaxResult|权限码：monitor:job:query|
|18|SysJobLogController|DELETE|/monitor/jobLog/{jobLogIds}|删除/清理/取消数据|@PathVariable Long[] jobLogIds|AjaxResult|权限码：monitor:job:remove|
|19|SysJobLogController|DELETE|/monitor/jobLog/clean|删除/清理/取消数据||AjaxResult|权限码：monitor:job:remove|
|20|SysJobLogController|POST|/monitor/jobLog/export|导出数据|HttpServletResponse response, SysJobLog sysJobLog|void|权限码：monitor:job:export|
|21|SysJobLogController|GET|/monitor/jobLog/list|查询列表/树/选项数据|SysJobLog sysJobLog|TableDataInfo|权限码：monitor:job:list|
|22|SysLogininforController|DELETE|/monitor/logininfor/{infoIds}|删除/清理/取消数据|@PathVariable Long[] infoIds|AjaxResult|权限码：monitor:logininfor:remove|
|23|SysLogininforController|DELETE|/monitor/logininfor/clean|删除/清理/取消数据||AjaxResult|权限码：monitor:logininfor:remove|
|24|SysLogininforController|POST|/monitor/logininfor/export|导出数据|HttpServletResponse response, SysLogininfor logininfor|void|权限码：monitor:logininfor:export|
|25|SysLogininforController|GET|/monitor/logininfor/list|查询列表/树/选项数据|SysLogininfor logininfor|TableDataInfo|权限码：monitor:logininfor:list|
|26|SysLogininforController|GET|/monitor/logininfor/unlock/{userName}|执行业务动作|@PathVariable("userName") String userName|AjaxResult|权限码：monitor:logininfor:unlock|
|27|SysOperlogController|DELETE|/monitor/operlog/{operIds}|删除/清理/取消数据|@PathVariable Long[] operIds|AjaxResult|权限码：monitor:operlog:remove|
|28|SysOperlogController|DELETE|/monitor/operlog/clean|删除/清理/取消数据||AjaxResult|权限码：monitor:operlog:remove|
|29|SysOperlogController|POST|/monitor/operlog/export|导出数据|HttpServletResponse response, SysOperLog operLog|void|权限码：monitor:operlog:export|
|30|SysOperlogController|GET|/monitor/operlog/list|查询列表/树/选项数据|SysOperLog operLog|TableDataInfo|权限码：monitor:operlog:list|
|31|SysUserOnlineController|DELETE|/monitor/online/{tokenId}|退出登录|@PathVariable String tokenId|AjaxResult|权限码：monitor:online:forceLogout|
|32|SysUserOnlineController|GET|/monitor/online/list|查询列表/树/选项数据|String ipaddr, String userName|TableDataInfo|权限码：monitor:online:list|

### 系统工具

|序号|Controller|请求方式|接口 URL|功能说明|请求/参数来源|响应类型|鉴权/权限|
|---:|---|---|---|---|---|---|---|
|1|GenController|PUT|/tool/gen|新增/创建数据|@Validated @RequestBody GenTable genTable|AjaxResult|权限码：tool:gen:edit|
|2|GenController|GET|/tool/gen/{tableId}|查询详情|@PathVariable Long tableId|AjaxResult|权限码：tool:gen:query|
|3|GenController|DELETE|/tool/gen/{tableIds}|删除/清理/取消数据|@PathVariable Long[] tableIds|AjaxResult|权限码：tool:gen:remove|
|4|GenController|GET|/tool/gen/batchGenCode|详见 Controller 方法实现|HttpServletResponse response, String tables|void|权限码：tool:gen:code|
|5|GenController|GET|/tool/gen/column/{tableId}|查询列表/树/选项数据|Long tableId|TableDataInfo|权限码：tool:gen:list|
|6|GenController|POST|/tool/gen/createTable|新增/创建数据|String sql|AjaxResult|角色：admin|
|7|GenController|GET|/tool/gen/db/list|查询列表/树/选项数据|GenTable genTable|TableDataInfo|权限码：tool:gen:list|
|8|GenController|GET|/tool/gen/download/{tableName}|下载文件/模板/代码包|HttpServletResponse response, @PathVariable("tableName") String tableName|void|权限码：tool:gen:code|
|9|GenController|GET|/tool/gen/genCode/{tableName}|详见 Controller 方法实现|@PathVariable("tableName") String tableName|AjaxResult|权限码：tool:gen:code|
|10|GenController|POST|/tool/gen/importTable|导入数据|String tables|AjaxResult|权限码：tool:gen:import|
|11|GenController|GET|/tool/gen/list|查询列表/树/选项数据|GenTable genTable|TableDataInfo|权限码：tool:gen:list|
|12|GenController|GET|/tool/gen/preview/{tableId}|查询详情|@PathVariable("tableId") Long tableId|AjaxResult|权限码：tool:gen:preview|
|13|GenController|GET|/tool/gen/synchDb/{tableName}|详见 Controller 方法实现|@PathVariable("tableName") String tableName|AjaxResult|权限码：tool:gen:edit|
|14|TestController|DELETE|/test/user/{userId}|删除/清理/取消数据|@PathVariable Integer userId|R<String>|需登录 token|
|15|TestController|GET|/test/user/{userId}|查询详情|@PathVariable Integer userId|R<UserEntity>|需登录 token|
|16|TestController|GET|/test/user/list|查询列表/树/选项数据||R<List<UserEntity>>|需登录 token|
|17|TestController|POST|/test/user/save|新增/创建数据|UserEntity user|R<String>|需登录 token|
|18|TestController|PUT|/test/user/update|修改状态或业务数据|@RequestBody UserEntity user|R<String>|需登录 token|

### 公共能力

|序号|Controller|请求方式|接口 URL|功能说明|请求/参数来源|响应类型|鉴权/权限|
|---:|---|---|---|---|---|---|---|
|1|CaptchaController|GET|/captchaImage|详见 Controller 方法实现|HttpServletResponse response|AjaxResult|匿名放行|
|2|CommonController|GET|/common/download|下载文件/模板/代码包|String fileName, Boolean delete, HttpServletResponse response, HttpServletRequest request|void|需登录 token|
|3|CommonController|GET|/common/download/resource|下载文件/模板/代码包|String resource, HttpServletRequest request, HttpServletResponse response|void|需登录 token|
|4|CommonController|POST|/common/upload|上传文件或头像|MultipartFile file|AjaxResult|需登录 token|
|5|CommonController|POST|/common/uploads|上传文件或头像|List<MultipartFile> files|AjaxResult|需登录 token|

# 四、接口测试情况

## 4.1 测试基础信息

|项目|内容|
|---|---|
|测试环境|未启动服务；根据源码配置推定开发环境为 `http://localhost:8080`|
|测试工具|本次未使用 Postman/JMeter；使用 PowerShell 静态扫描 Controller 与配置文件|
|测试时间|2026-04-10 16:40|
|测试范围|Controller 路由映射、权限注解、应用配置、App 业务实体字段|
|测试结论|接口清单已生成；真实可用性、参数必填、数据库数据约束、权限菜单配置仍需启动后端并联调验证|

## 4.2 测试结果汇总

|接口模块|接口数量（个）|通过数量（个）|未通过数量（个）|通过率|说明|
|---|---:|---:|---:|---|---|
|App移动端/售后业务|52|0|0|未执行|静态扫描已覆盖，HTTP 联调待执行|
|系统认证与首页|5|0|0|未执行|静态扫描已覆盖，HTTP 联调待执行|
|系统管理|81|0|0|未执行|静态扫描已覆盖，HTTP 联调待执行|
|系统监控|32|0|0|未执行|静态扫描已覆盖，HTTP 联调待执行|
|系统工具|18|0|0|未执行|静态扫描已覆盖，HTTP 联调待执行|
|公共能力|5|0|0|未执行|静态扫描已覆盖，HTTP 联调待执行|
|合计|193|0|0|未执行|本报告不伪造接口测试通过结果|

## 4.3 建议联调用例

|序号|场景|建议验证点|
|---:|---|---|
|1|后台登录与菜单|`POST /login` 获取 token，随后调用 `/getInfo`、`/getRouters`，确认用户、角色、权限和菜单返回正确|
|2|App 注册登录|`GET /app/auth/sendCode`、`POST /app/auth/register`、`POST /app/auth/login`，验证验证码、手机号、角色类型和 token 返回|
|3|售后工单流程|用户创建售后单，商家查看待接单、接单、更新状态，管理员查看与审核处理|
|4|配件订单流程|用户创建配件订单、商家发货/完成/取消、用户评价商家，验证库存和订单状态流转|
|5|权限边界|分别使用 admin、merchant、user 角色 token 访问后台管理、商家端、用户端接口，验证 401/403 与业务错误返回|
|6|文件上传下载|调用 `/common/upload`、`/common/uploads`、`/common/download/resource`，验证大小限制、路径返回和访问权限|

# 五、接口问题及优化建议

## 5.1 现存风险

- 部分接口参数必填和业务状态约束主要依赖服务层逻辑，建议为 App 业务请求体补充 `@Validated` 与字段级校验注解。
- `application.yml` 中存在固定 JWT secret，`application-druid.yml` 中存在数据源与 Druid 控制台默认凭据，建议开发、测试、生产环境分离并使用环境变量或密钥管理。
- `app.debug.smsCodeFile` 会把验证码写入本地文件，应仅限本地联调开启，测试/生产环境建议关闭。
- 报告中存在匿名放行的 App 公共查询接口和文件资源访问接口，需结合防盗链、文件路径校验和上传白名单防止越权或任意文件访问。
- 本次未启动服务执行 HTTP 测试，接口实际可用性仍受数据库结构、初始化数据、Redis、权限菜单数据和前端代理配置影响。

## 5.2 优化建议

- 为 App 业务接口补充 Swagger/OpenAPI 注解或统一接口文档，明确字段必填、枚举状态、示例请求和示例响应。
- 对售后单、配件订单、商家审核等状态流转建立状态机或枚举校验，避免非法状态跨越。
- 对上传接口补充文件类型、大小、路径穿越和访问权限校验，并记录上传审计日志。
- 对所有后台管理接口用最小权限原则核对 `@PreAuthorize` 权限码与数据库菜单权限配置是否一致。
- 在 `src/test/java` 中补充核心服务单元测试，并准备 Postman/Apifox 集合覆盖登录、权限、售后单、配件订单、上传下载和定时任务管理流程。

# 六、接口文档维护说明

1. Controller 新增、删除或修改路由后，应同步更新本报告的接口清单、权限说明和联调用例。
2. App 业务实体字段、状态枚举或数据库表结构变更后，应同步更新“App 业务实体与请求体字段概要”。
3. 环境配置、鉴权策略、Swagger 前缀或部署端口发生变化后，应同步更新“接口基本信息”。
4. 每次接口联调后，应在测试章节记录实际通过/失败数量、问题原因和修复计划。

|版本号|修改日期|修改人|修改内容|
|---|---|---|---|
|V1.0|2026-04-10|AI 辅助生成|初始编制，基于 Controller、SecurityConfig、application 配置和 App 实体生成接口报告|

# 七、总结

本次报告覆盖 `RuoYi-Vue-master` 后端 193 个 URL 映射，主要包括 App 移动端/售后业务、后台系统管理、系统监控、代码生成工具和公共文件/验证码能力。项目接口整体延续若依 `AjaxResult`、`TableDataInfo`、JWT token 和 `@PreAuthorize` 权限控制模式；新增 App 业务围绕手机号注册登录、用户售后单、商家接单、配件商城订单与评价展开。

由于本次未启动服务和数据库进行真实 HTTP 调用，报告适合作为接口盘点与联调准备文档；正式验收前建议按第四章建议用例完成接口级测试，并把测试结果回填到本文档。

# 八、附件（可选）

- 源码 Controller 路径：`ruoyi-admin/src/main/java/com/ruoyi/web/controller`、`ruoyi-generator/src/main/java/com/ruoyi/generator/controller`、`ruoyi-quartz/src/main/java/com/ruoyi/quartz/controller`
- 关键配置：`ruoyi-admin/src/main/resources/application.yml`、`ruoyi-admin/src/main/resources/application-druid.yml`、`ruoyi-framework/src/main/java/com/ruoyi/framework/config/SecurityConfig.java`
- SQL 参考：`sql/init_hanzhong_project_full.sql`、`sql/seed_hanzhong_project_demo_data.sql`
