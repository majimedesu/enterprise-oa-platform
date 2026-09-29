# 企业级自动化办公平台（enterprise-oa-platform）

一套面向企业内部的办公自动化（OA）系统，**管理端 + 员工端** 双端协同：

- **管理端**：组织与权限管理、审批流程模板配置、流程监控与审批、公众号菜单配置；
- **员工端**：依托微信公众号（H5），发起请假/加班等审批、接收待办与结果通知。

后端以 Spring Boot 多模块结构组织，认证授权、工作流引擎、消息推送三块能力相互解耦。

## 技术栈

| 分类 | 选型 |
|---|---|
| 基础框架 | Spring Boot 2.3.6、Java 8、Maven 多模块 |
| 持久层 | MyBatis-Plus 3.4.1、MySQL 8、HikariCP |
| 认证授权 | Spring Security + JWT（jjwt）、MD5 加盐密码编码 |
| 工作流 | Activiti（BPMN 2.0） |
| 缓存 | Redis |
| 微信生态 | weixin-java-mp（公众号 SDK） |
| 接口文档 | Knife4j 3.0.3 |
| 其他 | FastJSON、Lombok |
| 前端 | 管理端 Vue2 + ElementUI（基于 vue-admin-template 4.4.0）；员工端 Vue2 + Vant + form-create |

## 模块结构

```
guigu-oa-parent                 # 后端父工程（依赖与版本统一管理）
├── common
│   ├── common-util             # 通用工具：JWT、统一响应、MD5、结果码
│   ├── service-util            # 通用配置：MyBatis-Plus、Knife4j、全局异常
│   └── spring-security         # 认证授权：过滤器、自定义用户、密码编码器
├── model                       # 实体类与 VO（system / process / wechat）
└── service-oa                  # 业务服务：权限管理、审批流、公众号对接
guigu-oa-admin                  # 管理端前端
guigu-oa-web                    # 员工端前端（微信公众号 H5）
sql/schema.sql                  # 业务库表结构（Activiti 引擎表启动时自动创建）
```

## 核心功能

### 1. 组织与权限管理（RBAC）

- 用户、角色、菜单、部门、岗位的完整维护，以及用户–角色、角色–菜单的授权关系维护；
- 菜单驱动的前端动态路由：后端用 `RouterVo` / `MetaVo` 构造路由树，由 `MenuHelper` 完成父子菜单的递归组装，前端按返回结果注册路由与按钮权限；
- 认证流程：登录 → 校验账号密码 → 签发 JWT → 同时把该用户的权限集合写入 Redis；
  后续请求由 `TokenAuthenticationFilter` 解析 token，**从 Redis 取回权限**回填到 SecurityContext，避免每个请求都查库；
- `WebSecurityConfig` 统一配置过滤链：关闭 CSRF、放行登录接口、其余请求均需认证；
  密码使用自定义 MD5 编码器（`CustomMd5PasswordEncoder`）加密存储，用户信息由 `UserDetailsServiceImpl` 装载。

### 2. 审批流（Activiti）

- **流程模板管理**：`ProcessTemplateController` 支持流程定义的上传（`uploadProcessDefinition`）、保存、修改、发布（`publish`）；`ProcessServiceImpl#deployByZip` 将流程包部署到 Activiti 引擎；
- **流程类型管理**：`ProcessTypeController` 维护审批类型（如请假、加班），与模板关联；
- **审批全流程**：发起（`startUp`，提交表单数据）→ 待我审批（`findPending`）→ 审批处理（`approve`，通过/驳回并写审批意见）→ 已办（`findProcessed`）→ 我发起的（`findStarted`）→ 流程详情（`show`，返回流程信息 + 审批记录）；
- 业务表（`oa_process`、`oa_process_record`、`oa_process_template`、`oa_process_type`）与 Activiti 引擎表分离，业务侧只记录业务状态，流程状态由引擎维护；
- 内置 7 个 BPMN 2.0 流程定义（请假 3 个、加班 4 个），包含网关分支与任务监听器（`MyTaskListener`），用于在任务节点上扩展业务逻辑；
- `ProcessApiController` 暴露员工端调用的流程接口。

### 3. 微信公众号集成

- **菜单管理**：可在系统内配置公众号菜单（`MenuController` / `MenuServiceImpl#syncMenu`），保存后同步到微信服务器，也支持删除远端菜单；
- **OAuth2 网页授权登录**：`WechatController#authorize` 引导跳转微信授权页，`#userInfo` 用 code 换取用户信息，`#bindPhone` 完成手机号绑定，实现员工端免密登录；
- **模板消息推送**：`MessageServiceImpl#pushPendingMessage` 在流程流转到新审批人时推送待办通知，
  `#pushProcessedMessage` 在审批结束后把结果推送给发起人，二者均通过 `WxMpTemplateMessage` 发送，调用点在 `ProcessServiceImpl` 的流程发起与审批环节。

### 4. 通用能力

- 统一响应体 `Result` + 结果码 `ResultCodeEnum` + `ResponseUtil`，业务异常统一由 `GlobalExceptionHandler` 转换为标准响应；
- 实体公共字段由 `BaseEntity` 承载；MyBatis-Plus 分页、自动填充等配置集中在 `service-util`；
 - MyBatis-Plus 分页与字段自动填充、Knife4j 在线接口文档（`/doc.html`）由 `service-util` 统一配置。

## 本地运行

1. 环境：JDK 8、Maven、MySQL 8、Redis、Node.js（前端）；
2. 创建数据库并导入业务表结构：`mysql -uroot -p < sql/schema.sql`（Activiti 引擎表在应用启动时按 `database-schema-update: true` 自动创建）；
 3. 配置 `guigu-oa-parent/service-oa/src/main/resources/application-dev.yml`：
   - MySQL 连接与账号密码；
   - Redis 连接；
   - 微信公众号参数（`wechat.mpAppId` / `mpAppSecret` / `userInfoUrl`）通过环境变量
     `WECHAT_MP_APP_ID`、`WECHAT_MP_APP_SECRET`、`WECHAT_USER_INFO_URL` 注入，建议使用微信公众平台测试号；
4. 启动 `ServiceAuthApplication`（端口 `8800`）；
5. 前端：`guigu-oa-admin`（管理端）与 `guigu-oa-web`（员工端）分别 `npm install && npm run serve`。

## 说明

- 管理端前端基于开源模板 [vue-admin-template](https://github.com/PanJiaChen/vue-admin-template)（MIT），保留了其原始 License 与说明文件；
- 公众号相关的 AppID / AppSecret / 回调地址已改为占位配置，使用前请替换为自己的值。
