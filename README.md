# 万家安良居

基于 Vue 3 + Node.js + MongoDB 的全功能房屋租赁管理平台，支持租客、房东、管理员三种角色。

## 功能概览

### 租客端
- **房源浏览** — 按区域、价格、类型筛选搜索房源
- **房源详情** — 查看房源信息、房东信息、用户评价
- **预约看房** — 提交看房预约，管理预约状态
- **合同管理** — 查看和签署电子合同，支持拒绝合同（清空甲方签署状态）
- **评价系统** — 对租住过的房源进行评分和评价

### 房东端
- **房源管理** — 发布/编辑/上下架房源，上架时自动检查无未到期合同
- **预约管理** — 查看租客预约，确认或拒绝看房请求
- **合同管理** — 创建和管理租赁合同，电子签署
  - 合同创建/签署时自动检查日期冲突（同房源不可重叠）
  - 签署后房源自动锁定下架，不再对外展示
  - 签署前可修改，一旦有一方签署则锁定不可改
  - 租户拒绝后清空签署状态，回到草稿可修改
- **财务管理** — 按月统计租金收入，支持柱状图可视化
  - **收入自动生成** — 合同签署后按月自动生成财务记录
  - **收入实时同步** — 修改合同租金时自动同步更新对应财务记录
  - **数据可视化** — 柱状图展示收入趋势，支持月/季/年粒度切换

### 管理后台
- **用户管理** — 查看所有用户，启用/禁用账号
- **房源审核** — 审核房东发布的房源，通过或拒绝
- **数据统计** — 用户数、房源数、预约数、合同数等概览
- **系统设置** — 配置房屋类型、支付方式、审核开关

## 技术栈

| 层 | 技术 |
|---|------|
| **前端** | Vue 3 (Composition API), Vite, Vue Router, Pinia, Element Plus, Axios, ECharts |
| **后端** | Node.js, Express.js, JWT 认证, Mongoose ODM |
| **数据库** | MongoDB |
| **设计** | 自定义主题（深青主色 `#1d4359`），毛笔字体 Logo（Ma Shan Zheng） |

## 快速开始

### 前置条件

- Node.js >= 18
- MongoDB >= 6.0（本地运行或 Docker）
- npm 或 yarn

### 1. 克隆并安装依赖

```bash
# 安装后端依赖
cd backend
npm install

# 安装前端依赖
cd ../frontend
npm install
```

### 2. 启动 MongoDB

**方式一：本地 MongoDB 服务**
```bash
# Windows - 确保 MongoDB 服务已启动
Start-Service -Name "MongoDB"
```

**方式二：Docker**
```bash
docker run -d -p 27017:27017 --name mongo mongo:7
```

**方式三：无需安装（自动降级）**

后端会自动检测本地 MongoDB，若连接失败则自动启用内存数据库（`mongodb-memory-server`），数据会在重启后清空。

### 3. 初始化管理员账号

```bash
cd backend
node seed.js
```

默认管理员：
```
手机号: 13800000000
密码:   admin123
角色:   管理员
```

### 4. 启动项目

打开两个终端窗口：

```bash
# 终端 1 - 启动后端
cd backend
node server.js
# 输出: MongoDB 连接成功 / 服务器启动成功 http://localhost:3000

# 终端 2 - 启动前端
cd frontend
npm run dev
# 输出: http://localhost:5173
```

### 5. 访问使用

浏览器打开 **http://localhost:5173**

- 注册新账号 → 选择角色（租客/房东）
- 使用管理员账号登录后台管理

## 角色登录方式

登录页支持三种角色入口：

| 角色 | 入口 |
|------|------|
| **租户** | 默认登录页 `/login`，输入账号密码直接登录 |
| **房东** | 点击「房东登录」→ `/login/landlord` → 标题「房东登录」 |
| **管理员** | 点击「管理员登录」→ `/login/admin` → 标题「管理员登录」 |

角色专属页面底部有「返回普通登录」链接。

## 核心业务逻辑

### 合同签署 → 收入生成 → 房源锁定

```
双方签署合同 → 合同状态变为 signed
  ↓
房源自动下架 (status: offline)，公开列表不再展示
  ↓
按月自动生成 FinanceRecord（从 startDate 到 endDate）
  ↓
财务页面即可查看收入数据和柱状图趋势
```

### 合同冲突保护

合同创建、从预约创建、签署时，均会检查同房源是否存在时间重叠的有效合同：

```
新合同:       |------ 2025-06-01 ~ 2025-12-31 ------|
现有合同:            |-- 2025-08-01 ~ 2025-10-31 --|
                        ↻ 冲突！拒绝创建/签署
```

### 合同拒绝机制

租户可拒绝待签署的合同：

```
房东创建合同(draft)
  ↓
房东签署 → 状态 pending_sign (signedByLandlord = true)
  ↓
┌─ 租户签署 → signed → 房源锁定下架，生成收入记录
└─ 租户拒绝 → signedByLandlord 清空, 回到 draft → 房东可修改再发
```

### 收入同步机制

修改合同租金时，系统自动更新该合同关联的所有财务记录金额。

## 项目结构

```
home/
├── frontend/                      # 前端 Vue 3 项目
│   ├── public/
│   │   ├── carousel/              # 首页轮播图（6 张房源主题图片）
│   │   ├── login_two.png          # 登录/注册页背景图
│   │   └── login_bg_phone.png     # 移动端登录背景图
│   ├── src/
│   │   ├── views/
│   │   │   ├── HomePage.vue       # 首页（轮播图 + 房源列表搜索）
│   │   │   ├── HouseDetail.vue    # 房源详情
│   │   │   ├── LoginPage.vue      # 登录（含角色路由 /login/:role）
│   │   │   ├── RegisterPage.vue   # 注册（双栏布局，与登录一致）
│   │   │   ├── tenant/            # 租客端页面
│   │   │   │   ├── MyAppointments.vue
│   │   │   │   ├── MyContracts.vue    # 含拒绝合同功能
│   │   │   │   ├── MyProfile.vue
│   │   │   │   └── MyReviews.vue
│   │   │   ├── landlord/          # 房东端页面
│   │   │   │   ├── HouseManage.vue
│   │   │   │   ├── HouseForm.vue
│   │   │   │   ├── AppointmentManage.vue
│   │   │   │   ├── ContractManage.vue
│   │   │   │   └── FinanceManage.vue   # 柱状图 + 统计卡片
│   │   │   └── admin/             # 管理后台页面
│   │   │       ├── UserManage.vue
│   │   │       ├── HouseReview.vue
│   │   │       ├── DataStats.vue
│   │   │       └── SystemSettings.vue
│   │   ├── components/            # 公共组件
│   │   │   ├── NavBar.vue         # 导航栏（毛笔字体 Logo）
│   │   │   ├── FooterBar.vue
│   │   │   └── HouseCard.vue
│   │   ├── router/index.js        # 路由 + 权限守卫
│   │   ├── stores/auth.js         # Pinia 认证状态
│   │   ├── style.css              # 全局样式 + Element Plus 主题
│   │   └── utils/request.js       # Axios 封装（Vite 代理 /api）
│   └── package.json
│
├── backend/                       # 后端 Express 项目
│   ├── routes/
│   │   ├── auth.js                # 认证（注册/登录/资料）
│   │   ├── houses.js              # 房源 CRUD + 审核 + 上下架（含合同检查）
│   │   ├── appointments.js        # 预约（角色自适应列表）
│   │   ├── contracts.js           # 合同（创建/签署/拒绝/终止/租金修改）
│   │   ├── finance.js             # 财务记录 + 收入统计（柱状图数据）
│   │   ├── reviews.js             # 评价（评分/可见性）
│   │   └── admin.js               # 管理（用户/统计/设置）
│   ├── models/                    # Mongoose 数据模型
│   │   ├── User.js
│   │   ├── House.js
│   │   ├── Appointment.js
│   │   ├── Contract.js
│   │   ├── FinanceRecord.js
│   │   ├── Review.js
│   │   ├── OperationLog.js
│   │   └── Setting.js
│   ├── middleware/                 # 认证中间件 + 错误处理
│   ├── config/index.js            # 配置文件
│   ├── server.js                  # 入口文件
│   └── seed.js                    # 管理员初始化脚本
│
├── regist/                        # 登录页设计参考素材
├── doc/
│   └── BEGINNER_GUIDE.md          # 初学者项目文档
├── API_DOCUMENTATION.md           # API 详细文档
└── README.md                      # 本文件
```

## API 概览

Base URL: `/api`（开发环境通过 Vite 代理转发到 `http://localhost:3000`）

### 认证
| 方法 | 端点 | 说明 |
|------|------|------|
| POST | `/auth/register` | 用户注册 |
| POST | `/auth/login` | 用户登录（返回角色信息） |
| GET | `/auth/me` | 获取当前用户 |
| PUT | `/auth/profile` | 更新个人资料 |

### 房源
| 方法 | 端点 | 说明 |
|------|------|------|
| GET | `/houses` | 房源列表（仅 approved） |
| GET | `/houses/my` | 我的房源（房东） |
| GET | `/houses/pending` | 待审核房源（管理员） |
| GET | `/houses/all` | 全部房源（管理员） |
| GET | `/houses/:id` | 房源详情 |
| POST | `/houses` | 发布房源（房东） |
| PUT | `/houses/:id` | 编辑房源（房东） |
| PUT | `/houses/:id/review` | 审核房源（管理员） |
| PUT | `/houses/:id/status` | 上架/下架（上架时检查合同） |

### 预约
| 方法 | 端点 | 说明 |
|------|------|------|
| GET | `/appointments` | 预约列表（角色自适应） |
| POST | `/appointments` | 创建预约（租客） |
| PUT | `/appointments/:id/confirm` | 确认预约（房东） |
| PUT | `/appointments/:id/reject` | 拒绝预约（房东） |
| PUT | `/appointments/:id/cancel` | 取消预约（租客） |

### 合同
| 方法 | 端点 | 说明 |
|------|------|------|
| GET | `/contracts` | 合同列表（角色自适应） |
| GET | `/contracts/:id` | 合同详情 |
| POST | `/contracts` | 创建合同（检查冲突 + 房源锁定） |
| POST | `/contracts/from-appointment/:id` | 从预约创建合同 |
| PUT | `/contracts/:id` | 修改合同（已签署则锁定不可改） |
| PUT | `/contracts/:id/sign` | 签署合同（检查冲突 + 房源下架 + 生成收入） |
| PUT | `/contracts/:id/reject` | 拒绝合同（租户，清空签署状态） |
| PUT | `/contracts/:id/terminate` | 终止合同 |

### 财务
| 方法 | 端点 | 说明 |
|------|------|------|
| GET | `/finance` | 财务记录列表（房东） |
| GET | `/finance/stats` | 收入统计数据（按时间分组，柱状图用） |

### 评价
| 方法 | 端点 | 说明 |
|------|------|------|
| GET | `/reviews/house/:houseId` | 房源评价（公开） |
| GET | `/reviews/my` | 我的评价（租客） |
| POST | `/reviews` | 创建评价（租客） |
| PUT | `/reviews/:id/visibility` | 切换评价可见性（管理员） |

### 管理
| 方法 | 端点 | 说明 |
|------|------|------|
| GET | `/admin/users` | 用户列表（支持搜索、角色筛选） |
| PUT | `/admin/users/:id/status` | 启用/禁用用户 |
| GET | `/admin/stats` | 系统统计 |
| GET | `/admin/settings` | 获取系统设置 |
| PUT | `/admin/settings` | 更新系统设置 |

详见 [API 文档](API_DOCUMENTATION.md)。

## 设计主题

| 元素 | 值 | 说明 |
|------|-----|------|
| **主色** | `#1d4359` | 深青蓝，传递专业、可靠、沉稳感 |
| **按钮色** | `#3b4859` | 深灰蓝 |
| **辅助文字** | `#516473` | 中灰蓝 |
| **边框/分割线** | `#d6dee4` | 浅灰蓝 |
| **Logo 字体** | Ma Shan Zheng | Google Fonts 毛笔楷体，32px |
| **圆角** | 40px（输入框）、12px（卡片）、6px（按钮） |
| **阴影** | 轻量级 `rgba(29, 67, 89, 0.08)` |

## 测试

```bash
# 财务模块测试用例
tests/finance-module.test-cases.json

# 注册模块测试用例
tests/register.test-cases.json
```

## 初学者指南

详细的项目架构说明、开发环境搭建、前后端联调指南和关键技术点解释，请参见 [doc/BEGINNER_GUIDE.md](doc/BEGINNER_GUIDE.md)。
