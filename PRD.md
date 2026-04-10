# Umami v3.0.3 产品需求文档 (PRD)

---

## 1. 产品概述

### 1.1 产品定位
**Umami** 是一款现代的、注重隐私的网站分析平台（Web Analytics Platform），定位为 **Google Analytics 的开源替代方案**。

- **产品名称**: Umami
- **当前版本**: 3.0.3
- **开源协议**: MIT License
- **官方地址**: https://umami.is
- **代码仓库**: https://github.com/umami-software/umami
- **开发公司**: Umami Software, Inc.

### 1.2 核心价值主张
- **隐私优先**: 不使用 Cookie，符合 GDPR/CCPA 等隐私法规
- **轻量高效**: 脚本体积 < 1KB，页面加载影响极小
- **开源可控**: 数据完全自托管，用户拥有 100% 数据所有权
- **简单易用**: 界面简洁直观，降低上手门槛

### 1.3 目标用户
- 个人博客站长 / 独立开发者
- 中小企业（SMB）技术团队
- 注重数据隐私的组织机构
- 需要 Google Analytics 替代方案的网站运营者

---

## 2. 技术架构

### 2.1 整体架构图

```
┌─────────────────────────────────────────────────────────────┐
│                        客户端 (Browser)                       │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │ Tracker JS   │  │ Pixel (图片) │  │ Link (短链)  │       │
│  │ (script.js)  │  │ (/p/:slug)   │  │ (/q/:slug)   │       │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘       │
└─────────┼─────────────────┼─────────────────┼────────────────┘
          │ POST /api/send  │   自动触发        │ 自动触发
          └────────┬────────┴────────┬─────────┘
                   ▼                 ▼
┌──────────────────────────────────────────────────────────────┐
│                    Umami 服务端 (Next.js)                      │
│                                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐ │
│  │ API 层   │  │ 认证授权  │  │ 数据收集  │  │  报告查询    │ │
│  │ /api/*   │  │ JWT/Auth │  │ /send    │  │  /reports/*  │ │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └──────┬───────┘ │
│       │              │             │               │         │
│  ┌────┴──────────────┴─────────────┴───────────────┴───────┐ │
│  │                  业务逻辑层 (lib/)                        │ │
│  │  auth | db | clickhouse | kafka | redis | detect        │ │
│  └────────────────────────┬────────────────────────────────┘ │
│                           │                                   │
│  ┌────────────────────────┴────────────────────────────────┐ │
│  │                    查询层 (queries/)                      │ │
│  │  prisma/ (CRUD)  |  sql/ (分析查询)                      │ │
│  └────────────────────────┬────────────────────────────────┘ │
└───────────────────────────┼──────────────────────────────────┘
                            │
              ┌─────────────┼─────────────┐
              ▼             ▼             ▼
     ┌────────────┐ ┌────────────┐ ┌────────────┐
     │ PostgreSQL │ │ ClickHouse │ │   Redis    │
     │ (主数据库)  │ │ (时序数据)  │ │  (缓存)    │
     └────────────┘ └────────────┘ └────────────┘
                            │ (可选)
                     ┌──────▼──────┐
                     │    Kafka    │
                     │ (消息队列)   │
                     └─────────────┘
```

### 2.2 技术栈详情

| 层级 | 技术 | 说明 |
|------|------|------|
| **前端框架** | Next.js 15 + React 19 | 全栈框架，SSR/SSG 支持 |
| **语言** | TypeScript 5.9 | 全项目类型安全 |
| **UI 组件库** | @umami/react-zen | 自研组件库 |
| **状态管理** | Zustand 5 | 轻量级状态管理 |
| **数据请求** | @tanstack/react-query 5 | 服务端状态管理 |
| **图表库** | Chart.js 4 | 数据可视化 |
| **地图** | react-simple-maps | 世界地图展示 |
| **国际化** | react-intl 7 | 支持 52 种语言 |
| **CSS 方案** | PostCSS + CSS Modules | 模块化样式 |
| **ORM** | Prisma 6 | PostgreSQL 数据库操作 |
| **时序数据库** | ClickHouse (@clickhouse/client) | 高性能分析查询 |
| **缓存** | Redis (@umami/redis-client) | 会话缓存/数据缓存 |
| **消息队列** | Kafka (kafkajs) | 异步事件处理 |
| **认证** | JSON Web Token (jsonwebtoken) | 无状态认证 |
| **GeoIP** | MaxMind GeoLite2 | IP 地理位置解析 |
| **UA 解析** | ua-parser-js | 浏览器/设备/OS 检测 |
| **表单验证** | Zod 4 | 运行时类型校验 |
| **构建工具** | Rollup (tracker) / tsup (components) | 打包工具链 |
| **Linter** | Biome 2 | 代码格式化和检查 |
| **测试** | Jest + Cypress E2E | 单元测试和端到端测试 |
| **包管理** | pnpm + workspace | monorepo 管理 |
| **容器化** | Docker (多阶段构建) | 生产环境部署 |

### 2.3 核心设计模式

**双数据库架构**:
- **PostgreSQL (Prisma ORM)**: 存储用户、团队、网站、报告配置等结构化元数据
- **ClickHouse (原生 SQL)**: 存储网站事件（pageviews、events）、session 数据等高写入量时序数据
- 通过 `runQuery()` 统一调度层实现透明切换 (`src/lib/db.ts`)

**多渠道数据采集**:
- **Tracker Script** (`script.js`): 标准 JavaScript 嵌入式追踪脚本
- **Pixel Tracking** (`/p/:slug`): 图片像素追踪（适用于邮件/第三方场景）
- **Link Tracking** (`/q/[slug]`): 短链接追踪（适用于营销链接）

---

## 3. 数据模型

### 3.1 ER 关系图

```
User ◄──────┬──────► Website ◄──────► Team
│           │              │               │
│           │              │               │
│     WebsiteEvent         │          TeamUser
│           │              │               │
│           ├──► EventData │          Link, Pixel
│           │              │
│     Session ◄──► SessionData
│           │
│           └──► Revenue
│
├──► Report
├──► Link
├──► Pixel
└──► TeamUser ◄──► Team
```

### 3.2 核心实体详解

#### User（用户）
| 字段 | 类型 | 说明 |
|------|------|------|
| id | UUID | 主键 |
| username | String(255) | 登录名（唯一） |
| password | String(60) | bcrypt 加密密码 |
| role | String(50) | 角色：admin / user / view-only |
| logoUrl | String(2183) | 头像 URL |
| displayName | String(255) | 显示名称 |

#### Website（网站/应用）
| 字段 | 类型 | 说明 |
|------|------|------|
| id | UUID | 主键 |
| name | String(100) | 网站名称 |
| domain | String(500) | 域名 |
| shareId | String(50) | 分享令牌（唯一，用于公开分享） |
| userId | UUID | 所属用户 ID |
| teamId | UUID | 所属团队 ID |
| createdBy | UUID | 创建者 ID |

#### Session（会话）
| 字段 | 类型 | 说明 |
|------|------|------|
| id | UUID | 主键 |
| websiteId | UUID | 网站 ID |
| browser/os/device | String(20) | 浏览器/操作系统/设备类型 |
| screen | String(11) | 屏幕分辨率 |
| language | String(35) | 语言 |
| country/region/city | - | 地理位置（ISO 国家码/地区/城市） |
| distinctId | String(50) | 用户自定义标识 |

#### WebsiteEvent（网站事件）-- **核心事实表**
| 字段 | 类型 | 说明 |
|------|------|------|
| id | UUID | 主键 |
| websiteId/sessionId/visitId | UUID | 关联标识 |
| urlPath/urlQuery | String(500) | 页面路径和查询参数 |
| referrerPath/referrerDomain | - | 来源页面和域名 |
| pageTitle | String(500) | 页面标题 |
| eventType | Int | 事件类型：1=页面浏览 2=自定义事件 3=链接事件 4=像素事件 |
| eventName | String(50) | 自定义事件名称 |
| tag | String(50) | 事件标签 |
| utm_source/medium/campaign/content/term | - | UTM 营销参数 |
| gclid/fbclid/msclkid/ttclid/lifatid/twclid | - | 广告平台 Click ID |
| hostname | String(100) | 主机名 |

#### EventData（事件属性数据）
| 字段 | 类型 | 说明 |
|------|------|------|
| dataKey | String(500) | 属性键名 |
| stringValue / numberValue / dateValue | - | 支持字符串/数字/日期三种值类型 |
| dataType | Int | 数据类型：1=string 2=number 3=boolean 4=date 5=array |

#### Team & TeamUser（团队协作）
- **Team**: 团队实体，包含 name、accessCode（邀请码）、logoUrl
- **TeamUser**: 团队成员关联，支持角色：team-owner / team-manager / team-member / team-view-only

#### Report（报告）
| 字段 | 类型 | 说明 |
|------|------|------|
| type | String(50) | 报告类型：funnel / journey / retention / breakdown / utm / revenue / goal / attribution |
| parameters | Json | 报告参数配置（JSON 存储） |

#### Segment（分群）
| 字段 | 类型 | 说明 |
|------|------|------|
| type | String(50) | segment（静态分群）/ cohort（队列分群） |
| parameters | Json | 分群过滤条件 |

#### Revenue（收入追踪）
| 字段 | 类型 | 说明 |
|------|------|------|
| eventName | String(50) | 触发事件名 |
| currency | String(10) | 货币代码（USD/CNY 等 33 种） |
| revenue | Decimal(19,4) | 收入金额 |

#### Link & Pixel（追踪工具）
- **Link**: 短链接追踪（name, url, slug）
- **Pixel**: 图片像素追踪（name, slug）

---

## 4. 功能模块详述

### 4.1 数据采集系统 (Collection System)

#### 4.1.1 JavaScript Tracker（`src/tracker/index.js`）

**核心能力**:
- **自动页面追踪**: 监听页面加载，自动发送 pageview 事件
- **SPA 路由追踪**: Hook `pushState`/`replaceState`，支持单页应用路由变化追踪
- **自定义事件追踪**: `umami.track(eventName, data)` API
- **用户身份识别**: `umami.identify(userId, properties)` API
- **元素点击追踪**: 通过 `data-umami-event` 和 `data-umami-event-*` 属性声明式绑定
- **Do Not Track 尊重**: 检测浏览器 DNT 设置
- **域名过滤**: 仅追踪配置域名的数据
- **客户端缓存**: 使用 JWT 缓存 session/visit 信息，减少服务端查询

**配置属性**（通过 script 标签 data 属性）:
```html
<script src="/script.js"
  data-website-id="xxx"
  data-host-url="https://your-umami.com"
  data-auto-track="true"
  data-do-not-track="false"
  data-domains="yourdomain.com"
  data-exclude-search="false"
  data-exclude-hash="false"
  data-tag="production"
></script>
```

**数据发送流程**:
1. 构建 payload（website, screen, language, title, hostname, url, referrer, tag, identity）
2. 检查是否禁用追踪（DNT / localStorage / 域名匹配）
3. 执行 beforeSend 回调（如配置）
4. POST 到 `/api/send` endpoint
5. 响应中返回 cache token 用于后续请求复用

#### 4.1.2 数据接收 API（`/api/send`）

**处理流程** (`src/app/api/send/route.ts`):
1. **请求验证**: Zod schema 校验（必须且仅能提供 website/link/pixel 其中之一）
2. **缓存检查**: 解析 `x-umami-cache` header 中的 JWT token
3. **客户端信息提取**:
   - IP 地址提取（支持代理头）
   - User-Agent 解析（browser / OS / device）
   - GeoIP 定位（优先 Cloudflare/Vercel/CloudFront 头 → MaxMind 数据库回退）
4. **安全检查**:
   - Bot 过滤（isbot 库检测爬虫）
   - IP 黑名单过滤（支持 CIDR 格式）
5. **Session 管理**:
   - 基于 IP + UA + 月度盐值生成 session ID
   - 基于 session + 小时盐值生成 visit ID
   - visit 30 分钟过期自动续期
6. **事件保存**:
   - URL 解析与规范化
   - UTM 参数提取（source/medium/campaign/content/term）
   - Click ID 提取（Google/FB/Microsoft/TikTok/LinkedIn/Twitter）
   - Referrer 解析（path/query/domain）
7. **响应**: 返回新的 cache token

**支持的事件类型**:
- `event` (type=1): 页面浏览
- `event` with name (type=2): 自定义事件
- `event` via link (type=3): 链接点击事件
- `event` via pixel (type=4): 像素曝光事件
- `identify`: 用户身份/属性数据

#### 4.1.3 Pixel 追踪（`/p/[slug]`）

通过 1x1 透明 GIF 图片实现追踪：
- 适用于邮件追踪、第三方 HTML 环境
- GET 请求触发，返回 GIF 图片
- 内部转换为标准 event 发送流程
- Redis 缓存 slug→pixel 映射（TTL 86400s）

#### 4.1.4 Link 追踪（`/q/[slug]`）

短链接点击追踪：
- 302 重定向到目标 URL
- 记录点击事件

### 4.2 用户认证与权限系统

#### 4.2.1 认证机制

**JWT Token 认证**:
- Access Token: Bearer Token 放入 Authorization header
- Share Token: 通过 `x-umami-share-token` header 传递（用于公开分享报告）
- Redis Auth Key: 可选的 Redis 会话存储机制

**角色体系**（7 种角色）:

| 角色 | 权限范围 |
|------|----------|
| **admin** | 全部权限（平台管理员） |
| **user** | 创建/编辑/删除网站、创建团队 |
| **view-only** | 只读访问 |
| **team-owner** | 团队全部权限 + 网站管理 + 转移 |
| **team-manager** | 团队编辑 + 网站管理 + 转移到团队 |
| **team-member** | 团队内网站 CRUD |
| **team-view-only** | 团队内只读 |

**权限矩阵** (`src/lib/constants.ts`):
```
website:create / website:update / website:delete
website:transfer-to-team / website:transfer-to-user
team:create / team:update / team:delete
```

#### 4.2.2 SSO 单点登录
- 支持 SSO 集成（`/api/auth/sso`）
- 云模式（CLOUD_MODE）下可对接外部认证

### 4.3 分析报告系统

#### 4.3.1 实时看板（Realtime）
- **端点**: `/api/realtime/[websiteId]`
- **刷新间隔**: 10 秒
- **时间窗口**: 最近 30 分钟
- **数据维度**:
  - 活跃访客数趋势图
  - 页面浏览实时列表
  - 访客地理分布（国家维度）
  - 引荐来源统计
  - 自定义事件实时计数
  - URL 点击热力

#### 4.3.2 核心指标（Metrics）
- **综合指标卡片**: 浏览量(PV)、访客数(UV)、访问时长、跳出率
- **对比分析**: 支持与上一时段对比（环比）
- **活跃访客**: 当前在线独立访客数（`getActiveVisitors`）

#### 4.3.3 趋势图表（Charts）
- **浏览量趋势图** (`PageviewsChart`): 时间序列折线图，支持 minute/hour/day/month/year 粒度
- **事件趋势图** (`EventsChart`): 自定义事件时间分布
- **周流量对比** (`WeeklyTraffic`): 星期几流量分布对比

#### 4.3.4 维度明细（Breakdown）
支持按以下维度进行分组聚合分析：
- **页面维度**: path / entry / exit / title / query
- **来源维度**: referrer / domain（自动归类为 Search/Social/Shopping/Email/Video）
- **设备维度**: browser / os / device / screen
- **地理维度**: country / region / city / language
- **事件维度**: event / tag / hostname

**内置域名分组** (`GROUPED_DOMAINS`): Google/Bing/Facebook/Twitter/LinkedIn/Reddit 等 20+ 来源归类

#### 4.3.5 高级分析报告

**漏斗分析 (Funnel)** (`queries/sql/reports/getFunnel.ts`):
- 定义最多 N 步转化漏斗
- 每步支持 URL 路径或事件名称
- 配置时间窗口（分钟级）
- 输出：每步访客数、流失数、流失率、留存率
- 使用 CTE (Common Table Expression) 递归查询实现

**用户旅程分析 (Journey)** (`queries/sql/reports/getJourney.ts`):
- 分析用户在站点内的 N 步行为路径
- 支持设置起始步骤和结束步骤过滤
- 输出 Top 100 行为路径组合及频次
- 使用 `row_number() OVER (PARTITION BY visit_id)` 构建事件序列

**留存分析 (Retention)** (`queries/sql/reports/getRetention.ts`):
- Cohort（队列）留存分析
- 计算每日首次访问用户在后续 31 天内的回归情况
- 输出经典的留存表格（日期 x 天数 → 留存率百分比）

**归因分析 (Attribution)**: 多触点归因分析

**UTM 分析**: UTM 参数效果分析

**收入分析 (Revenue)**: 基于事件的收入追踪，支持 33 种货币

**目标转化 (Goal)**: 目标达成分析

#### 4.3.6 动态事件数据 (Event Data)
- 支持为事件附加自定义键值对属性
- 三种数据类型: string / number / date
- 支持属性的统计分析和值分布查看

#### 4.3.7 Session 数据 (Session Data)
- 通过 identify API 上报的用户属性
- 按 session 维度关联
- 支持属性级别的聚合分析

#### 4.3.8 分群与筛选 (Segment & Filter)
- **静态分群 (Segment)**: 预定义过滤条件组合
- **队列分群 (Cohort)**: 基于时间段+条件的动态分群
- **过滤器**: 支持 16 种操作符（equals/contains/greaterThan/before 等）
- **过滤字段**: 覆盖事件和 session 全部维度

#### 4.3.9 数据导出
- 支持 CSV 格式导出详细数据

### 4.4 网站管理 (Website Management)
- CRUD 操作（创建/编辑/删除网站）
- 网站转移（用户↔团队）
- 分享链接生成（shareId）
- 数据重置（resetAt）
- 网站详情：统计数据概览、追踪代码获取

### 4.5 团队协作 (Team Management)
- 创建/编辑/删除团队
- 邀请码加入团队（accessCode）
- 成员角色分配
- 团队级别网站管理
- 团队内 Link/Pixel 资源共享

### 4.6 报告面板 (Boards / Dashboards)
- 自定义报告面板
- 多个报告组件组合
- 报告模板保存与管理

### 4.7 管理后台 (Admin Panel)
- **用户管理**: 查看/编辑/删除用户，角色变更
- **团队管理**: 管理所有团队
- **网站管理**: 管理所有网站

### 4.8 设置 (Settings)
- **偏好设置**: 主题（亮/暗）、语言（52种）、时区、日期范围默认值
- **个人资料**: 头像、显示名、密码修改
- **团队设置**: 团队信息编辑
- **网站设置**: 网站配置管理

### 4.9 公开分享 (Sharing)
- 通过 shareId 生成公开访问链接
- 匿名访问（Share Token 认证）
- 自定义 Header/Footer

---

## 5. API 接口总览

### 5.1 数据采集接口
| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/api/send` | 数据上报主入口 |
| GET | `/p/[slug]` | Pixel 图片追踪 |
| GET | `/q/[slug]` | Link 短链追踪 |
| POST | `/api/batch` | 批量数据上报 |

### 5.2 认证接口
| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/api/auth/login` | 登录 |
| POST | `/api/auth/logout` | 登出 |
| POST | `/api/auth/sso` | SSO 登录 |
| POST | `/api/auth/verify` | 验证 |

### 5.3 网站数据接口
| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/websites` | 网站列表 |
| POST | `/api/websites` | 创建网站 |
| GET/PUT/DELETE | `/api/websites/[id]` | 网站 CRUD |
| GET | `/api/websites/[id]/stats` | 综合统计 |
| GET | `/api/websites/[id]/metrics` | 指标数据 |
| GET | `/api/websites/[id]/pageviews` | 浏览量数据 |
| GET | `/api/websites/[id]/sessions` | 会话数据 |
| GET | `/api/websites/[id]/events` | 事件数据 |
| GET | `/api/websites/[id]/active` | 活跃访客 |
| GET | `/api/websites/[id]/daterange` | 数据日期范围 |
| GET | `/api/websites/[id]/realtime` | 实时数据 |
| GET | `/api/websites/[id]/export` | 数据导出 |

### 5.4 报告接口
| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/reports/breakdown` | 维度明细 |
| GET | `/api/reports/funnel` | 漏斗分析 |
| GET | `/api/reports/journey` | 旅程分析 |
| GET | `/api/reports/retention` | 留存分析 |
| GET | `/api/reports/utm` | UTM 分析 |
| GET | `/api/reports/revenue` | 收入分析 |
| GET | `/api/reports/attribution` | 归因分析 |
| GET | `/api/reports/goal` | 目标分析 |

### 5.5 管理接口
| 方法 | 路径 | 说明 |
|------|------|------|
| GET/POST/DELETE | `/api/admin/users` | 用户管理 |
| GET/POST/DELETE | `/api/admin/websites` | 网站管理 |
| GET/POST/DELETE | `/api/admin/teams` | 团队管理 |

### 5.6 其他接口
| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/me` | 当前用户信息 |
| PUT | `/api/me/password` | 修改密码 |
| GET | `/api/config` | 系统配置 |
| GET | `/api/heartbeat` | 健康检查 |
| GET | `/api/share/[shareId]` | 公开分享数据 |
| CRUD | `/api/teams` | 团队管理 |
| CRUD | `/api/links` | 链接管理 |
| CRUD | `/api/pixels` | 像素管理 |
| CRUD | `/api/reports` | 报告管理 |

---

## 6. 国际化 (i18n)

支持 **52 种语言**，包括:
ar-SA, be-BY, bg-BG, bn-BD, bs-BA, ca-ES, cs-CZ, da-DK, de-CH, de-DE, el-GR, en-US, en-GB, es-ES, fa-IR, fi-FI, fo-FO, fr-FR, ga-ES, he-IL, hi-IN, hr-HU, hu-HU, id-ID, it-IT, ja-JP, km-KH, ko-KR, lt-LT, mn-MN, ms-MY, my-MM, nb-NL, nl-NL, pl-PL, pt-BR, pt-PT, ro-RO, ru-RU, si-LK, sk-SK, sl-SI, sv-SE, ta-IN, th-TH, tr-TR, uk-UA, ur-UZ, uz-UZ, vi-VN, zh-CN, zh-TW

---

## 7. 部署方案

### 7.1 Docker 部署（推荐）
- **多阶段构建**: deps → builder → runner
- **基础镜像**: Node.js 22 Alpine
- **运行用户**: nextjs (UID 1001, 非 root)
- **暴露端口**: 3000
- **依赖**: 外部 PostgreSQL 数据库

### 7.2 Docker Compose
一键启动 Umami + PostgreSQL 全套服务

### 7.3 Podman
提供 systemd user service 支持，适合 Linux 无 root 部署

### 7.4 Netlify
通过 `@netlify/plugin-nextjs` 支持 Edge 部署

### 7.5 源码部署
```bash
git clone https://github.com/umami-software/umami.git
cd umami && pnpm install
# 配置 .env (DATABASE_URL=postgresql://...)
pnpm build && pnpm start
```

### 7.6 环境变量

| 变量 | 说明 |
|------|------|
| DATABASE_URL | PostgreSQL 连接串（必需） |
| CLICKHOUSE_URL | ClickHouse 连接串（可选，用于高性能分析） |
| REDIS_URL | Redis 连接串（可选，用于缓存） |
| KAFKA_URL / KAFKA_BROKER | Kafka 配置（可选，消息队列） |
| BASE_PATH | 子路径部署前缀 |
| CLOUD_MODE / CLOUD_URL | 云模式配置 |
| DISABLE_BOT_CHECK | 禁用机器人检测 |
| IGNORE_IP | 忽略的 IP 列表（逗号分隔，支持 CIDR） |
| GEOLITE_DB_PATH | GeoLite2 数据库路径 |
| FORCE_SSL | 强制 HTTPS |
| DEFAULT_LOCALE | 默认语言 |
| TRACKER_SCRIPT_NAME | 自定义 tracker 脚本名 |

---

## 8. 安全机制

### 8.1 应用层安全
- **CSP (Content Security Policy)**: 严格的资源加载策略
- **HSTS**: 可选强制 SSL
- **Bot 检测**: isbot 库过滤爬虫流量
- **IP 黑名单**: 支持单 IP 和 CIDR 网段过滤
- **输入校验**: Zod schema 全面的请求体验证
- **SQL 注入防护**: 参数化查询（Prisma + ClickHouse 参数绑定）
- **JWT 安全体**: secret 密钥签名，含过期机制

### 8.2 隐私保护
- **无 Cookie 依赖**: 使用 LocalStorage + JWT 缓存策略
- **Do Not Track 尊重**: 可选 DNT 检测
- **数据自托管**: 用户完全拥有数据
- **合规设计**: GDPR / CCPA 友好

---

## 9. 项目目录结构说明

```
umami/
├── prisma/                  # 数据库 Schema 和迁移
│   ├── schema.prisma        # 数据模型定义（11 个核心实体）
│   └── migrations/          # 14 个版本迁移文件
├── db/
│   ├── clickhouse/          # ClickHouse Schema 和迁移
│   └── postgresql/          # PG 数据迁移脚本
├── src/
│   ├── app/                 # Next.js App Router
│   │   ├── (collect)/       # 数据采集入口（pixel/link）
│   │   ├── (main)/          # 主应用布局和页面
│   │   │   ├── admin/       # 管理后台
│   │   │   ├── boards/      # 报告面板
│   │   │   ├── console/     # 测试控制台
│   │   │   ├── dashboard/   # 仪表盘
│   │   │   ├── links/       # 链接管理
│   │   │   ├── pixels/      # 像素管理
│   │   │   ├── settings/    # 设置
│   │   │   ├── teams/       # 团队管理
│   │   │   └── websites/    # 网站管理与分析
│   │   ├── api/             # API 路由（60+ 接口）
│   │   ├── login/           # 登录页
│   │   ├── share/           # 公开分享页
│   │   └── sso/             # SSO 页
│   ├── components/          # React 组件
│   │   ├── charts/          # 图表组件（Bar/Pie/Bubble）
│   │   ├── common/          # 通用组件（Grid/Modal/Button等）
│   │   ├── hooks/           # 47 个自定义 Hooks
│   │   ├── input/           # 表单输入组件
│   │   ├── metrics/         # 指标展示组件
│   │   └── svg/             # SVG 图标组件（44个）
│   ├── lib/                 # 核心库（34 个模块）
│   ├── queries/             # 数据查询层
│   │   ├── prisma/          # PG CRUD 查询（9 个实体）
│   │   └── sql/             # 分析 SQL 查询（42 个查询函数）
│   ├── permissions/         # 权限校验（7 个模块）
│   ├── store/               # Zustand 状态管理
│   ├── tracker/             # 前端追踪脚本
│   └── lang/                # 52 种语言包
├── public/
│   ├── intl/                # 编译后的语言包
│   ├── images/              # 图标资源（浏览器/国家/设备/OS）
│   └── datamaps.world.json  # 世界地图 GeoJSON
├── scripts/                 # 构建和工具脚本
├── cypress/                 # E2E 测试
├── Dockerfile               # Docker 多阶段构建
├── docker-compose.yml       # 编排文件
└── package.json             # 项目依赖
```

---

## 10. 关键业务流程

### 10.1 数据采集完整流程
```
用户访问网站 → 加载 umami script.js
    → 自动 track() 或手动 umami.track()
    → 构建 payload {website, url, referrer, screen, language...}
    → POST /api/send (带 x-umami-cache header)
    → 服务端: 校验 → 提取客户端信息 → Bot/IP检查
    → 生成/复用 Session & Visit
    → 写入 ClickHouse / PostgreSQL
    → 返回 {cache: newToken, sessionId, visitId}
```

### 10.2 报告查询流程
```
用户打开仪表盘 → 选择网站和时间范围
    → 并行请求: stats + metrics + pageviews + sessions + events...
    → API 层: checkAuth → 权限校验
    → Queries 层: 根据 DATABASE_TYPE 路由到 Prisma 或 ClickHouse
    → SQL 查询执行（带 filter/dateRange/cohort 条件）
    → 返回聚合数据 → 前端渲染图表
```

### 10.3 漏斗分析查询原理
```
定义步骤: [URL:/home, URL:/product, EVENT:purchase, window:30min]
→ SQL CTE 递归:
  level1: SELECT session_id FROM events WHERE url_path='/home'
  level2: JOIN level1 WHERE url_path='/product' AND time ∈ [level1.time, level1.time+30min]
  level3: JOIN level2 WHERE event_name='purchase' AND time ∈ [level2.time, level2.time+30min]
→ 计算每步 visitors / dropoff / remaining
```
