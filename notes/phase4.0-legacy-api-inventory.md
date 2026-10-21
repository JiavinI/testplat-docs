# Phase 4.0 旧项目接口基础信息清单

> **2026-10-15 阅读提示**：本文是 4.0 调研时的接口清单，部分“待核实 / 不支持”是当时观察，不代表当前完整能力。Engine 已支持基础 output alias 跨步骤传值；本轮另参考了旧项目的只读冒烟、商品解析与跨系统拿货指派场景，证据与采用范围见 [4.1 设计稿 §2](phase4.1-design-draft.md#2-旧项目参考与新项目现状)。不据本清单直接整体迁移旧 DSL；账号和环境归属以 [当前基线](phase4-roadmap.md) 为准。

来源：`/Users/jiavin/Developer/products/testplat-new`（只读）。统计口径：assets/api 下 46 个 YAML，其中 1 个是 `create.checks.yaml`（用例检查集，不是接口），实际接口 45 个。
场景引用次数 = `assets/scenarios/**` 中 `use: api.xxx` 出现的次数（同一场景多次引用重复计数）。

## 1. 业务系统总览

| 系统目录 | host 名 | 接口数 | 需登录 | 登录方式（策略名） | 被场景引用最多的 3 个接口（次数） |
|---|---|---|---|---|---|
| gsb | gsb | 24 | 是 | gsb_token (http_form, GET /site/login) | `gsb/b2/order/search_product` (4); `gsb/b2/account/balance` (3); `gsb/b2/aftersale/orders/create` (2) |
| daifa_admin | daifa_admin | 14 | 是 | df_admin_api (vcode_redis) | `daifa_admin/aftersale/batch_return_assign` (3); `daifa_admin/aftersale/query_all_sign_info` (3); `daifa_admin/aftersale/sign_in_new` (3) |
| daifa_picker | daifa_picker | 5 | 是 | df_picker_api (http_form) | `daifa_picker/pick/query_settle_detail_info` (14); `daifa_picker/pick/settle_total_price` (8); `daifa_picker/pick/update_refund_item_price` (3) |
| admin_go2 | admin_go2 | 2 | 是 | admin_go2_api (cookie_session + flow admin_form_session) | `admin_go2/withdraw/approve` (1); `admin_go2/withdraw/refuse_detail` (1) |

合计 45 个接口。

域名分组（test 环境，域名可记录）：

- `java.qa.gogoo.cn`：gsb(:8306)、admin_go2(:16640)、daifa_picker(:8318)，同一主机不同端口。
- `df.qa.go2b2b.com`：daifa_admin。
- `tt.ximgs.net`：unified_login，仅作 gsb 登录入口，没有接口资产使用它。
- `tcat.ximgs.net`：cat，有登录配置，但 assets/api 下没有 cat 接口。

## 2. 各业务系统接口表

### 2.1 gsb（24 个接口）

| 相对路径 | 名称 | method | url | 请求体类型 | inputs 字段名 | body/params 顶层字段 | success | 引用 |
|---|---|---|---|---|---|---|---|---|
| `b1/withdraw/apply.yaml` | 提交提现申请 | POST | `/api/b1/withdraw/apply` | json | `bank_card_no`, `withdraw_amount` | `bankCardNo`, `withdrawAmount` | `$.errorCode=0` | 1 |
| `b2/account/balance.yaml` | 查询账户余额 | POST | `/api/account/balance` | json | - | - | `$.errorCode=0` | 3 |
| `b2/aftersale/orders/create.yaml` | 申请售后 | POST | `/api/b2/aftersale/orders/create` | json | `apply_reason`, `items`, `order_no`, `business_after_sale_type`, `return_goods_type` | `applyReason`, `items`, `orderNo`, `businessAfterSaleType`, `returnGoodsType` | `$.errorCode=0` | 2 |
| `b2/aftersale/orders/return_service_fee_product_info.yaml` | 查询售后商品与退货服务费 | POST | `/api/b2/aftersale/orders/returnServiceFee/getProductInfo` | json | `items` | `items` | `$.errorCode=0` | 2 |
| `b2/area/user_area_place.yaml` | 地址识别 | POST | `/api/b2/area/userAreaPlace` | json | `receiver_text`, `receiver_suffix` | `text` | `$.errorCode=0` | 2 |
| `b2/fund/transfer/list.yaml` | 查询资金划拨记录 | POST | `/api/fund/transfer/list` | json | `page_no`, `page_size`, `search_field`, `search_terms` | `pageNo`, `pageSize`, `searchField`, `searchTerms` | `$.errorCode=0` | 0 |
| `b2/fund/transfer/settle.yaml` | 预结算资金划拨 | POST | `/api/fund/transfer/settle` | json | `ids` | `ids` | `$.errorCode=0` | 0 |
| `b2/order/e3e3/coupons.yaml` | 查询 e3e3 可用优惠券 | POST | `/api/b2/orders/coupons` | json | `product`, `quantity` | `items`, `selectedCouponIds` | `$.errorCode=0` | 0 |
| `b2/order/e3e3/coupons_items.yaml` | 查询 e3e3 整单可用优惠券 | POST | `/api/b2/orders/coupons` | json | `items` | `items`, `selectedCouponIds` | `$.errorCode=0` | 1 |
| `b2/order/e3e3/create.yaml` | 创建 e3e3 普通待支付订单 | POST | `/api/b2/orders/create` | json | `product`, `receiver`, `express`, `service_fee`, `service_fee_total_origin`, `service_fee_total`, `quantity`, `freight_fee`, `web_order_amount`, `channel`, `shipping`, `source_channel`, `shop_source_type`, `create_from` | `orderCreateParameters` | `$.errorCode=0` | 0 |
| `b2/order/e3e3/create_items.yaml` | 创建 e3e3 items 普通待支付订单 | POST | `/api/b2/orders/create` | json | `order_items`, `receiver`, `express`, `service_fee`, `service_fee_total_origin`, `service_fee_total`, `total_quantity`, `freight_fee`, `web_order_amount`, `supplier_id`, `send_type`, `send_time`, `channel`, `source_channel`, `shop_source_type`, `create_from` | `orderCreateParameters` | `$.errorCode=0` | 1 |
| `b2/order/e3e3/create_items_with_coupon.yaml` | 创建 e3e3 items 优惠券待支付订单 | POST | `/api/b2/orders/create` | json | `order_items`, `receiver`, `express`, `service_fee`, `service_fee_total_origin`, `service_fee_total`, `total_quantity`, `freight_fee`, `web_order_amount`, `discounted_web_order_amount`, `coupon`, `supplier_id`, `send_type`, `send_time`, `channel`, `source_channel`, `shop_source_type`, `create_from` | `orderCreateParameters` | `$.errorCode=0` | 1 |
| `b2/order/e3e3/create_with_coupon.yaml` | 创建 e3e3 优惠券待支付订单 | POST | `/api/b2/orders/create` | json | `product`, `receiver`, `express`, `service_fee`, `service_fee_total_origin`, `service_fee_total`, `quantity`, `freight_fee`, `web_order_amount`, `coupon`, `discounted_web_order_amount`, `channel`, `shipping`, `source_channel`, `shop_source_type`, `create_from` | `orderCreateParameters` | `$.errorCode=0` | 0 |
| `b2/order/freight.yaml` | 计算运费 | POST | `/api/b2/orders/freight` | json | `express_id`, `quantity`, `receiver_province` | `expressId`, `productCount`, `receiverProvince` | `$.errorCode=0` | 2 |
| `b2/order/go2/create.yaml` | 创建订单 | POST | `/api/b2/orders/create` | json | `product`, `receiver`, `express`, `quantity`, `freight_fee`, `web_order_amount`, `channel`, `send_type`, `source_channel`, `shop_source_type`, `create_from`, `goods_pay_later_flag`, `remark` | `orderCreateParameters` | `$.errorCode=0` | 0 |
| `b2/order/go2/create_items.yaml` | 创建 GO2 多商品待支付订单 | POST | `/api/b2/orders/create` | json | `order_items`, `receiver`, `express`, `freight_fee`, `web_order_amount`, `channel`, `send_type`, `source_channel`, `shop_source_type`, `create_from`, `goods_pay_later_flag`, `remark` | `orderCreateParameters` | `$.errorCode=0` | 2 |
| `b2/order/go2/settle.yaml` | 预结算 GO2 订单 | POST | `/api/b2/orders/settle` | json | `order_nos` | `orderNos` | `$.errorCode=0` | 2 |
| `b2/order/intellect_express.yaml` | 智能快递 | POST | `/api/b2/orders/intellect/express` | json | `product`, `receiver`, `sku_color`, `sku_size`, `quantity`, `source_channel` | `sourceChannel`, `provinceId`, `cityId`, `districtId`, `productInfos` | `$.errorCode=0` | 0 |
| `b2/order/intellect_express_items.yaml` | 查询任意 items 智能快递 | POST | `/api/b2/orders/intellect/express` | json | `receiver`, `product_infos`, `source_channel` | `sourceChannel`, `provinceId`, `cityId`, `districtId`, `productInfos` | `$.errorCode=0` | 2 |
| `b2/order/search_page.yaml` | 查询订单分页 | POST | `/api/b2/orders/page` | json | `search_field`, `search_terms`, `order_state`, `order_start_time`, `order_end_time`, `take_states`, `page_size`, `page_no`, `error_print_tag` | `searchField`, `searchTerms`, `orderState`, `orderStartTime`, `orderEndTime`, `takeStates`, `pageSize`, `pageNo`, `errorPrintTag` | `$.errorCode=0` | 2 |
| `b2/order/search_product.yaml` | 搜索商品 | GET | `/api/b2/orders/search/product` | query | `product_keyword` | `keywords` | `$.errorCode=0` | 4 |
| `b2/order/service_fee.yaml` | 查询服务费 | GET | `/api/b2/orders/serviceFee` | query | - | - | `$.errorCode=0` | 2 |
| `b2/pay/e3e3/order.yaml` | e3e3 余额支付订单 | POST | `/api/b2/pay/order` | json | `pay_way`, `password`, `order_nos` | `payWay`, `password`, `orderNos` | `$.errorCode=0` | 1 |
| `b2/pay/withdraw_refuse_notify.yaml` | 提现退票通知 | POST | `/api/b2/pay/withdraw/refuse/notify` | json | `ori_biz_order_no`, `ori_order_no`, `refuse_order_no`, `refuse_amount`, `fee`, `refuse_reason` | `oriBizOrderNo`, `oriOrderNo`, `refuseOrderNo`, `refuseAmount`, `fee`, `refuseReason` | `$.errorCode=0` | 1 |

### 2.2 daifa_admin（14 个接口）

| 相对路径 | 名称 | method | url | 请求体类型 | inputs 字段名 | body/params 顶层字段 | success | 引用 |
|---|---|---|---|---|---|---|---|---|
| `account/info.yaml` | 查询代发管理账户信息 | GET | `/api/platform/account/info` | query | - | - | `$.errorCode=0` | 1 |
| `aftersale/batch_refund_by_item.yaml` | 查询批量退款资格与费用 | POST | `/api/platform/afterSales/batchRefundByItem` | json | `ids` | `ids` | `$.errorCode=0` | 1 |
| `aftersale/batch_return_assign.yaml` | 批量指派退货任务 | POST | `/api/platform/afterSales/batchReturnAssign` | json | `ids`, `operator_id` | `ids`, `operatorId` | `$.errorCode=0` | 3 |
| `aftersale/confirm_refund.yaml` | 确认退货退款 | POST | `/api/platform/aftersale/orders/confirm` | json | `password`, `items` | `password`, `items` | `$.errorCode=0` | 1 |
| `aftersale/get_returned.yaml` | 查询退货退款快递费 | POST | `/api/platform/afterSales/getReturned` | json | `ids`, `return_freight_condition_tag` | `ids`, `returnFreightCondtionTag` | `$.errorCode=0` | 1 |
| `aftersale/query_all_sign_info.yaml` | 查询退件签收入库候选 | POST | `/api/platform/afterSales/queryAllSignInfo` | json | `express_no`, `take_code`, `order_no`, `express_id` | `expressNo`, `takeCode`, `orderNo`, `expressId` | `$.errorCode=0` | 3 |
| `aftersale/query_list.yaml` | 查询退货任务列表 | POST | `/api/platform/afterSales/queryList` | json | `page_no`, `page_size`, `search_field`, `search_terms`, `return_progress_list`, `api_version`, `start_time`, `end_time` | `pageNo`, `pageSize`, `searchField`, `searchTerms`, `returnProgressList`, `apiVersion`, `startTime`, `endTime` | `$.errorCode=0` | 2 |
| `aftersale/query_list_by_return_code.yaml` | 按退货编码查询退货任务 | POST | `/api/platform/afterSales/queryList` | json | `page_no`, `page_size`, `return_progress_list`, `return_list_item_no`, `api_version`, `start_time`, `end_time` | `pageNo`, `pageSize`, `returnProgressList`, `returnListItemNo`, `apiVersion`, `startTime`, `endTime` | `$.errorCode=0` | 1 |
| `aftersale/refund_goods_list.yaml` | 查询退款候选商品行 | POST | `/api/platform/afterSales/refund/goodsList` | json | `return_item_id`, `order_no`, `association_after_sale_flag` | `returnItemId`, `orderNo`, `associationASaleFlag` | `$.errorCode=0` | 1 |
| `aftersale/sign_in_new.yaml` | 退件签收入库 | POST | `/api/platform/afterSales/signInNew` | json | `items` | `(整体) ${items!}` | `$.errorCode=0` | 3 |
| `order_take/batch_assign.yaml` | 批量指派拿货员 | POST | `/api/platform/orderTake/batchAssign` | json | `take_list_ids`, `operator_id` | `ids`, `operatorId` | `$.errorCode=0` | 2 |
| `order_take/page_info.yaml` | 查询代发拿货单分页 | POST | `/api/platform/orderTake/pageInfo` | json | `searchTerms`, `searchField` | `pageSize`, `pageNo`, `searchField`, `searchTerms`, `filingState`, `assignState`, `noIncludeTakeState` | `$.errorCode=0` | 2 |
| `order_take/today_statistics.yaml` | 查询今日拿货统计 | GET | `/api/platform/orderTake/today/statistics` | query | - | - | `$.errorCode=0` | 3 |
| `product/search.yaml` | 搜索商品 | GET | `/api/platform/product/search` | query | `keywords` | `keywords` | `$.errorCode=0` | 1 |

### 2.3 daifa_picker（5 个接口）

| 相对路径 | 名称 | method | url | 请求体类型 | inputs 字段名 | body/params 顶层字段 | success | 引用 |
|---|---|---|---|---|---|---|---|---|
| `pick/data_list.yaml` | 查询拿货员待处理列表 | POST | `/api/pick/dataList` | json | `state`, `take_code` | `state`, `takeCode` | `$.errorCode=0` | 1 |
| `pick/query_settle_detail_info.yaml` | 查询拿货结算确认信息 | POST | `/api/pick/querySettleDetailInfo` | json | `return_list_item_ids`, `take_ids` | `returnListItemsIds`, `takeIds` | `$.errorCode=0` | 14 |
| `pick/settle_total_price.yaml` | 提交拿货结算 | POST | `/api/pick/settleTotalPrice` | json | `return_list_item_ids`, `take_ids`, `take_settles`, `total_price`, `pay_way` | `returnListItemsIds`, `takeIds`, `takeSettles`, `totalPrice`, `payWay` | `$.errorCode=0` | 8 |
| `pick/update_refund_item_price.yaml` | 修改退货价格 | POST | `/api/pick/updateRefundItemPrice` | json | `price`, `return_list_ids` | `price`, `returnListIds` | `$.errorCode=0` | 3 |
| `pick/update_take_item_price.yaml` | 修改拿货价格 | POST | `/api/pick/updateTakeItemPrice` | json | `price`, `take_list_ids`, `type` | `price`, `takeListIds`, `type` | `$.errorCode=0` | 2 |

### 2.4 admin_go2（2 个接口）

| 相对路径 | 名称 | method | url | 请求体类型 | inputs 字段名 | body/params 顶层字段 | success | 引用 |
|---|---|---|---|---|---|---|---|---|
| `withdraw/approve.yaml` | 审核提现 | POST | `/api/admin/withdraw/approve` | json | `withdraw_id`, `withdraw_skey` | `withdrawId`, `withdrawSkey` | `$.errorCode=0` | 1 |
| `withdraw/refuse_detail.yaml` | 查询退票详情 | GET | `/api/admin/withdraw/refuse/detail` | query | `withdraw_id` | `withdrawId` | `$.errorCode=0` | 1 |

## 3. 登录与鉴权

只记录策略名、接口路径、字段名与 header 名。账号、密码、验证码等值一律不记录（秘密，需 `${secret:}` 引用）。

| 系统 | 登录 host / 路径 | 旧策略 | 需要的秘密字段名 | token 注入 header |
|---|---|---|---|---|
| gsb | host `unified_login`，`GET /site/login`，账号字段走 query，另带 `source`（项目名）。token 取 `$.data.hash`，user_id 取 `$.data.userId`，ttl 14400s | `gsb_token`（http_form） | `username`、`password` | `X-TOKEN`、`X-USER-ID` |
| daifa_admin | host `daifa_admin`，`POST /createVCode`（取 `$.data.uuid`）后从 Redis 读验证码，再 `POST /login`（表单，含 `source`、`userType`）。token 取 `$.data.token`，ttl 72000s | `df_admin_api`（vcode_redis） | `username`、`password`、`login_password`（额外字段）；Redis 连接 `redis_main` 的密码 | `X-TOKEN`、`X-USER-ID` |
| daifa_picker | host `daifa_picker`，`POST /login`，表单，带 `source`。token 取 `$.data.token`，user_id 取 `$.data.userId`，ttl 14400s | `df_picker_api`（http_form） | `username`、`password`（旧配置有 `field_mapping`，别名未记录） | `Authorization` |
| admin_go2 | host `admin_go2`，先 `GET /login`（拿 JSESSIONID），再 `POST /login`（表单，带 `Referer`）。探针 `GET /nx/withdrawView`：状态 200，Location 不含 `/login`，body 不含登录表单。ttl 1000s | `admin_go2_api`（cookie_session + flow `admin_form_session`） | `username`、`password` | `Cookie` |

### 3.1 新 Engine 0.5 能否表达

| 系统 | 结论 | 说明 |
|---|---|---|
| gsb | 能 | `generic_flow` 单步，body 类型 `query`，`validity.ttl_seconds=14400`，extract 用 `body_json`。`source` 按项目写字面量（e3e3 / go2）。登录 host（`unified_login`）与资产 host（`gsb`）不同，需实测 host 映射是否支持 |
| daifa_admin | 能，依赖插件 | 必须用 `vcode_redis`（`testplat-control/plugins/testplat_login_plugins`）。需安装插件、在 `login_plugins` 白名单声明、配置 `redis_main` 连接。纯 `generic_flow` 做不了（要读 Redis） |
| daifa_picker | 能 | `generic_flow` 单步，body 类型 `form`，inject `Authorization: ${login:token}` |
| admin_go2 | 能，有限制 | `generic_flow` 两步，extract 源 `cookies`（`cookie_header`），`probe` 与 `${login_host}` 已支持。旧探针字段 `expected_status` 要改成 `success.status_code`。Cookie 身份不跨执行缓存（0.5 的 R5），每次执行重登 |

补充：
- 旧配置另有 `gsb_csrf` 和 `cat_api`，但 assets/api 下没有接口使用它们，本期不需要。
- 旧默认身份：`gsb`→`gsb.b1.primary`，`admin_go2`→`admin_go2.default`，`daifa_admin`→`df_admin.default`，`daifa_picker`→`df_picker.default`。场景里另用 `as:` 指定过 `gsb.b2.primary`（6 次）和 `df_admin.go2`（2 次）。
- 新 Engine 资产用 `auth: <身份名>` 声明身份，缺省走默认身份，`none` 为匿名。
- 以上能力判断来自 `login_flow.py`、`interface.py` 与 `COMPATIBILITY-0.5.md` 的阅读，没有实际跑过登录。

## 4. 旧 api YAML 与新格式的差异

统计对象是 45 个接口文件（不含 `create.checks.yaml`）。频次为 `assets/api` 中的静态扫描结果，其中“约”表示用文本启发式统计。

| 差异点 | 旧格式 | 新 Engine 现状 | 频次 | 处理建议 |
|---|---|---|---|---|
| 缺 `id` 字段 | 无，靠目录路径推导 | `yaml_to_api` 强制要求 `id`，且 `Registry` 要求与文件路径一致（`api.a.b.c` 对应 `a/b/c.yaml`） | 45 / 45 | 迁移时按路径补 `id: api.<目录>.<文件名>` |
| `${x!}` 必填标记 | 占位符后加 `!` | `resolver.py` 只按 `${name}` 做字典取值，`x!` 会被当成变量名，取不到时原样保留 | 138 处 / 41 个文件 | 去掉 `!`；“必填”要靠 `inputs` 声明或 Case 校验表达，需确认 |
| `${x|默认值}` | 占位符内置默认值 | 不支持，同上会原样保留 | 34 处 / 10 个文件 | 默认值挪到 Case 参数，或在资产里写死字面量 |
| 点号路径 `${a.b.c}` | 取嵌套对象字段 | 不支持，只查平铺键 | 约 221 处 / 9 个文件，集中在订单创建类 | 拆成平铺 inputs，或由 Case 先整理参数 |
| 整体 body 为单个占位符 | `body: "${items!}"` | `type: json` 下无限制；整串占位符且 `preserve_types` 时保留原 JSON 类型（需验证） | 1（`daifa_admin/aftersale/sign_in_new`） | 去掉 `!` 后验证 |
| query 类型的参数位置 | 写在 `request.params` | 发送时读的是 `request.body`（`send_kwargs` 的 query 分支），且值必须是标量 | 6 个文件（`GET` 接口） | 迁移时把 `params` 改名为 `body`；我只 grep 了三个执行文件，没发现读取 `request.params` 的地方 |
| `inputs` 值含义 | 字段名 → 中文说明 | 参考样例里是字段名 → 类型名（`username: str`） | 41 个文件有说明文字，4 个为空 `{}` | 说明文字可保留在 `desc`，类型要重新标注 |
| 顶层 `desc` | 有 | `Api` 模型没有该字段，加载时被忽略 | 45 / 45 | 无需处理 |
| 顶层 `idempotent` | 有 | 模型没有该字段 | 1（`gsb/b2/aftersale/orders/create`） | 被忽略，需要时放到 Case 层 |
| 首行 `# 运行命令：uv run testplat call ...` 注释 | 有 | 无对应 | 45 / 45 | 迁移时删除 |
| `*.checks.yaml` | 用例检查集，顶层是 `target` 和 `cases` | 不是接口资产；放进 `api` 目录会因缺少必需字段而加载失败 | 1（`gsb/b2/order/go2/create.checks.yaml`，3 个 cases） | 不迁移到 api，另行归入 Case |
| `success` 判定 | 全部是 `path: $.errorCode`、`value: "0"` | 支持 `path`/`value`，另支持 `status_code` | 45 / 45 | 可直接沿用 |
| `auth` 字段 | 接口文件里都没写，身份由环境配置的默认身份和场景的 `as:` 决定 | 资产可写 `auth: <身份名>`，缺省走默认身份，`none` 为匿名 | 0 / 45 | 新资产需要决定是否显式写 `auth` |
| `host` 取值 | `gsb`、`daifa_admin`、`daifa_picker`、`admin_go2` | 对应环境配置的 `api_hosts` 键 | 45 / 45 | 新环境配置里要补这 4 个 host（登录用的 `unified_login` 另算） |

另外，请求体字段名和 `inputs` 名常常不同（前者 camelCase，后者 snake_case），这是正常映射，不算格式差异。

## 5. 首批建议

选择标准：登录优先，其次是场景引用高、且不含 `|默认值` 与点号引用的接口。下列接口经扫描都满足后两点，只需去掉 `!`、补 `id`（query 类再把 `params` 改 `body`）就能用现有 Engine 表达。

### 5.1 推荐接口

| 系统 | 推荐接口（相对 `assets/api`） | 理由 |
|---|---|---|
| gsb | 登录 `gsb_token`（身份配置，不是资产）；`b2/account/balance`（引用 3，无入参）；`b2/order/search_product`（引用 4）；`b2/order/service_fee`（引用 2，无入参）；`b2/order/freight`（引用 2）；`b2/order/search_page`（引用 2） | 都是 json/query 的简单查询，无嵌套对象 |
| daifa_admin | 登录 `df_admin_api`（需 `vcode_redis` 插件）；`account/info`（无入参）；`order_take/today_statistics`（引用 3，无入参）；`order_take/page_info`（引用 2）；`product/search`；`aftersale/query_list`（引用 2） | 前三个最简单；`query_list` 入参较多（8 个必填），但无嵌套 |
| daifa_picker | 登录 `df_picker_api`；`pick/query_settle_detail_info`（引用 14，全库最高）；`pick/data_list`；`pick/update_take_item_price`；`pick/update_refund_item_price`；`pick/settle_total_price`（引用 8） | 该系统只有 5 个接口，全部可迁移；登录是单步表单，最容易先验证 |
| admin_go2 | 登录 `admin_go2_api`；`withdraw/refuse_detail`（query）；`withdraw/approve` | 只有 2 个接口，且场景只在提现流程里各用 1 次 |

暂不建议首批迁移：`gsb/b2/order/{e3e3,go2}/create*`、`coupons*`、`intellect_express*`。它们有大量点号引用（9 个文件约 221 处）和 `|默认值`，Engine 现在不支持，需先改资产结构或扩展 Engine。

### 5.2 可用顺序 steps 表达的简单流程

只写步骤与接口，不写数据。流程 A、B 的顺序取自旧场景的 `use:` 行，流程 C 是我对只读接口的组合，旧场景里没有这个顺序。

- 流程 A（daifa_picker，来自 `go2/take_return/headless/return_offline_settle_decrease.case.yaml`）：1) 查询拿货结算确认信息 `pick/query_settle_detail_info`；2) 修改退货价格 `pick/update_refund_item_price`；3) 再次查询结算确认信息 `pick/query_settle_detail_info`；4) 提交拿货结算 `pick/settle_total_price`。
- 流程 B（daifa_admin，来自 `shared/take_return/common/take_assigned.flow.yaml` 的前两步）：1) 查询代发拿货单分页 `order_take/page_info`；2) 查询今日拿货统计 `order_take/today_statistics`。旧流程的第 3 步 `gsb.b2.order.search_page` 属于另一个系统，可作为扩展。
- 流程 C（gsb 只读，自拟）：1) `b2/account/balance`；2) `b2/order/search_product`；3) `b2/order/service_fee`；4) `b2/order/freight`。

流程 A 的第 2、4 步入参依赖第 1 步响应里的数据。我没有核实新 Case 是否支持跨步骤引用响应字段，所以 A 可能需要人工传入固定参数。

## 6. 风险与无法表达的点

- 占位符语法：`${x!}`、`${x|默认值}`、`${a.b.c}` 三种写法在新 Engine 的 `resolver.py` 里都没有对应实现。订单创建类接口（`gsb/b2/order/{e3e3,go2}/create*`、`coupons*`、`intellect_express*`）重度依赖，首批无法迁移。
- 必填语义：去掉 `!` 之后，“必填”不再有强制校验，缺参会原样把 `${x}` 发出去。需要确认 Case 层是否校验。
- 插件版本：`testplat_login_plugins` 的 README 写的是 `testplat-engine>=0.4.0,<0.5`，而当前 Engine 是 0.5.0。我没有跑过插件测试，兼容性未验证。
- 登录 host 与资产 host 不一致：gsb 登录走 `unified_login`（`tt.ximgs.net`），接口走 `gsb`（`java.qa.gogoo.cn:8306`）。`generic_flow` 能否指定与资产不同的登录 host，我只看到 `ctx.host_url` 和 `${login_host}`，没有实测。
- 项目维度：gsb 的账号按项目区分（旧配置 `scope: project`），登录请求里还带项目名 `source`。新平台需要为 e3e3 和 go2 分别配置身份。
- 密码类入参：`daifa_admin/aftersale/confirm_refund`（`password`）、`gsb/b2/pay/e3e3/order`（`password`）、`admin_go2/withdraw/approve`（`withdraw_skey`）都要求业务密码。新 Engine 的资产加载器禁止在资产里写 `${secret:}`，这些值只能由 Case 传入；Case 如何引用 secrets，我没有核实。
- 跨步骤取值：流程 A 等需要把上一步响应里的字段传给下一步，我没有核实新 Case 是否支持，因此顺序 steps 建议里的数据依赖可能需要人工传参。
- Cookie 身份：admin_go2 属于 cookie 会话，不进跨执行缓存，每次执行重登。该系统只有 2 个接口，影响有限。
- 有副作用的接口：提现申请与审核、退款确认、余额支付、批量指派、结算提交都会改变测试环境数据。首批若包含它们，需要先确认环境是否允许，旧项目里仅 `gsb/b2/aftersale/orders/create` 标了 `idempotent`。
- 同 URL 多个资产：`/api/platform/afterSales/queryList`（2 个）、`/api/b2/orders/create`（5 个）、`/api/b2/orders/coupons`（2 个）、`/api/b2/orders/intellect/express`（2 个）各对应多份资产，只有 inputs 不同。迁移时要保持 id 唯一，不要误当作重复文件合并。
- 未被场景引用的接口（8 个）：`gsb/b2/fund/transfer/{list,settle}`、`gsb/b2/order/e3e3/{coupons,create,create_with_coupon}`、`gsb/b2/order/go2/create`、`gsb/b2/order/intellect_express`，另有 `create.checks.yaml`。其中 `go2/create` 在旧场景里只通过 `create_items` 间接使用，不建议首批迁移。
- 场景身份切换：旧场景用 `as:` 在同一系统内切换账号（`gsb.b2.primary` 6 次、`df_admin.go2` 2 次）。如果新平台的 Case 需要同样的切换，要先确认身份命名方式。

## 7. 本次调研说明

- 旧项目只读：只运行了 `ls`、`find`、`grep`、`sed` 和一段读取 YAML 的 Python（解释器用的是 testplat-engine 的 venv），没有运行旧项目的程序，也没有写入旧项目。
- 读取 `config/envs/_global.yaml` 时对账号、密码、token、验证码相关字段做了脱敏处理；没有打开 `config/envs/secrets/` 和任何 `.env` 文件。
- 场景只用于统计 `use: api.xxx` 的出现次数，没有抄录场景内容。
- 没有运行新 Engine 或插件的测试，第 3 节和第 4 节关于新 Engine 能力的结论来自阅读源码与兼容文档。

