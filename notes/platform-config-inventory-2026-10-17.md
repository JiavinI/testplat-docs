# 旧项目平台配置核对

来源只读：`/Users/jiavin/Developer/products/testplat-new/config`。本文件不含密码，仅记录导入前核对事实；尚未执行数据库写入。业务关系与导入要求以 [权威需求](../docs/requirements/platforms-and-projects.md) 为准，旧配置结构不能替代用户业务定义。

## API Host 与登录方式

| 系统 | 逻辑 Host | test 地址 | 旧登录 |
| --- | --- | --- | --- |
| 统一登录 | unified_login | http://tt.ximgs.net | GSB 登录入口 |
| 购商宝 | gsb | http://java.qa.gogoo.cn:8306 | GET /site/login，按业务项目 source 取 hash/userId，注入 X-TOKEN / X-USER-ID |
| GO2 管理后台 | admin_go2 | http://java.qa.gogoo.cn:16640 | GET /login 建 Cookie，POST /login 表单登录 |
| 代发管理后台 | daifa_admin | http://df.qa.go2b2b.com | /createVCode + Redis 验证码，再 POST /login，X-TOKEN / X-USER-ID |
| 代发拿货端 | daifa_picker | http://java.qa.gogoo.cn:8318 | POST /login 表单，Authorization token；旧凭据映射 phone / verification_code |
| CAT | cat | http://tcat.ximgs.net | /login 读取 CSRF，表单登录与 Cookie |

## 业务项目网站地址

| 项目 | 网站逻辑名 | 地址 |
| --- | --- | --- |
| GO2 | sso | http://tsso.go2.cn |
| GO2 | portal | http://test.go2.cn |
| GO2 | gsb | http://test.gsb.seller.go2.cn |
| E3E3 | sso | http://tsso.3e3e.cn |
| E3E3 | gsb | http://test.gsb.seller.3e3e.cn |
| GO2 管理后台 | admin_go2 | http://admin.go2b2b.com |

旧 E3E3 覆盖没有 portal，不能凭空补域名。API 和网站存在同名 gsb / admin_go2 逻辑键；导入时需要分别表达网站地址与 API 地址，避免覆盖；是否增加字段或独立键仍待实现评估，不预设 website_* 命名，也不暗示已接入 Page 执行。

## 账号与风险

旧 secrets 中有 GO2、E3E3 的 b1/b2 分组以及管理后台、代发后台、拿货端、CAT 独立凭据。只核对字段及存在性，不复制秘密到文档或 Git。用户点名的三个账号按权威需求中的指定用户名创建；密码来源已确认，分别采用旧 secrets 中对应账号的密码。

当前数据库仅 `main/gsb`，这只是旧实现的单一范围，不代表业务只有一个 GSB 平台。如何兼容映射到网站及所属 GSB 尚未定稿；现有真实余额配置、账号与历史必须保留。

代发后台依赖 Redis，当前 Web AuthTable 不传递 profile connections，也未发现已安装适配插件。该项应区分“配置已保存”与“登录可执行”，不得用其它平台登录流程冒充，或为本次配置导入扩展成鉴权架构重写。

## 用户后续澄清

GO2、E3E3 本身也是被测平台，而且各自拥有自己的 GSB；此前“只有四个平台”与“一个公共 GSB 下选择项目”的描述作废。代发后台、拿货端、GO2 管理后台仍各自独立登录。完整关系、账号与验收见权威需求；CAT 仅作为旧配置盘点项，不自动导入。
