# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

管伊佳ERP (jshERP) — an open-source ERP system (Apache 2.0) covering 进销存 (purchase/sales/inventory), financial management, and production. Multi-tenant SaaS architecture with a plugin framework.

## Build & Run

### Backend (`jshERP-boot/`)

```bash
cd jshERP-boot
mvn clean package          # Build (produces jshERP.jar)
mvn spring-boot:run        # Run directly
```

- Main class: `com.jsh.erp.ErpApplication`
- Port: `9999`, context path: `/jshERP-boot`
- API docs: `http://localhost:9999/jshERP-boot/doc.html` (Swagger)
- Requires: JDK 1.8, MySQL 8.0.24 (database: `jsh_erp`), Redis 6.2.1
- Config: [application.yml](jshERP-boot/src/main/resources/application.yml)

### Frontend (`jshERP-web/`)

```bash
cd jshERP-web
yarn install               # Install dependencies
yarn serve                 # Dev server on port 3000 (proxies to :9999)
yarn build                 # Production build
```

- Requires: Node 20.17.0
- Dev proxy config in [vue.config.js](jshERP-web/vue.config.js)

### Running the Full App

Running the backend alone starts the API server. You must also start the frontend dev server to have a working UI. Default login: tenant `jsh`, user `admin`, password `123456`.

## Architecture

### Backend (`jshERP-boot/`)

Standard SpringBoot layered architecture, Jeecg-Boot-derived:

```
controller/     → REST controllers, all extend BaseController
service/        → @Service classes (NO interface/impl split — direct classes)
datasource/
  mappers/      → MyBatis mapper interfaces (XxxMapper = generated CRUD, XxxMapperEx = custom SQL)
  entities/     → Entity + Example classes (MyBatis-Plus), plus hand-written VO classes
  vo/           → Shared value objects (tree nodes, etc.)
```

**Key patterns:**

- **Multi-tenancy**: MyBatis-Plus `TenantSqlParser` injects `tenant_id` into all SQL. Tenant ID resolved from `X-Access-Token` header. Some tables excluded from tenant filtering (see [TenantConfig.java](jshERP-boot/src/main/java/com/jsh/erp/config/TenantConfig.java)).
- **Authentication**: Session-based via Redis. `LogCostFilter` checks `userId` in session; whitelisted URLs bypass auth (login, register, Swagger docs, specific plugin URLs).
- **Pagination**: `BaseController.startPage()` sets up PageHelper, called at the top of each controller method. `getDataTable(list)` wraps results with `{rows, total}`.
- **Response format**: Controllers return `JSONObject` via `returnJson()` / `returnStr()` helpers, or `AjaxResult`/`TableDataInfo` from BaseController.
- **Plugin system**: `springboot-plugin-framework` loads external plugins from `plugins/` directory at runtime. Config in application.yml `plugin.*` keys.
- **MyBatis code generation**: MyBatis Generator configured at `src/test/resources/generatorConfig.xml`. Run via `mybatis-generator:generate`.

### Frontend (`jshERP-web/`)

Vue 2 + Ant Design Vue 1.5 app derived from Jeecg-Boot:

```
src/
  api/          → Backend API call functions (api.js is the main file)
  views/        → Pages organized by business domain
  components/   → Shared UI components (jeecg/ layouts, table, chart, etc.)
  store/        → Vuex (modules: app, dict, permission, user)
  config/       → Router and app config
  utils/        → Utilities (i18n, encryption, dict helpers)
```

- **Routing**: Dynamic — `asyncRouterMap` in [router.config.js](jshERP-web/src/config/router.config.js) is populated from the backend based on user permissions.
- **State**: `vuex` with `vue-ls` (localStorage) for persistence.
- **i18n**: `vue-i18n` with 73 language support. Language switched in 界面设置 panel.
- **API calls**: `src/api/api.js` contains most backend API functions. `manage.js` covers system management endpoints.
- **Bill modules**: Purchase/sales/inventory documents share a common pattern — views are in `views/bill/` with shared `Dialog` components and `mixins`.

### Business Domains

| Domain | Backend Entities | Frontend Views |
|--------|-----------------|----------------|
| 商品管理 (Product) | Material, MaterialCategory, MaterialExtend, MaterialProperty, MaterialAttribute | views/material/ |
| 仓库管理 (Inventory) | Depot, DepotHead, DepotItem, MaterialCurrentStock, SerialNumber | views/bill/ |
| 财务管理 (Financial) | Account, AccountHead, AccountItem | views/financial/ |
| 报表 (Reports) | Various Ex mappers for complex queries | views/report/ |
| 系统管理 (System) | User, Role, Tenant, Function, Log, SystemConfig | views/system/ |
| 基本资料 (Basic Data) | Supplier, Organization, Person, Unit, InOutItem | views/system/ |

## Multi-Tenancy Details

- Tenant ID column: `tenant_id` (MyBatis-Plus auto-injects based on `X-Access-Token` header)
- Tables excluded from tenant isolation: `jsh_sequence`, `jsh_function`, `jsh_platform_config`, `jsh_tenant`, `jsh_sys_dict_data`, `jsh_sys_dict_type`
- Super admin (tenantId=0) bypasses all tenant filters
- Each tenant has a role ID (config key `manage.roleId` = 10)
- Tenant limits: `tenant.userNumLimit` (max users), `tenant.tryDayLimit` (trial days)

## Key Constants

Reference [BusinessConstants.java](jshERP-boot/src/main/java/com/jsh/erp/constants/BusinessConstants.java) for:
- Document status codes (`BILLS_STATUS_*`)
- Document types (`SUB_TYPE_*` for purchase/sales/retail/transfer/check/assemble/disassemble)
- Financial types (`TYPE_MONEY_IN`, `TYPE_MONEY_OUT`, `TYPE_GIRO`)
- Delete flags (`DELETE_FLAG_*`), user defaults, etc.

## Database

- Database name: `jsh_erp`
- Tables prefixed `jsh_` (e.g., `jsh_user`, `jsh_depothead`, `jsh_material`)
- MyBatis XML mappers in `src/main/resources/mapper_xml/`
