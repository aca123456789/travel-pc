# 旅游日记 - 后台管理系统文档

本文档介绍后台管理系统的相关信息，包括功能模块和对应代码实现。

## 技术栈

- Remix 框架
- React
- TypeScript
- Tailwind CSS 
- PostgreSQL 数据库
- React Icons 库 (提供UI图标)

## 系统架构

后台管理系统采用基于Remix的单页应用架构，主要包含以下核心部分：

1. **布局组件**: `app/routes/admin.tsx`
   - 提供后台共享布局
   - 实现导航菜单和权限控制

2. **服务层**: `app/services/admin.server.ts`
   - 处理管理员身份验证
   - 提供会话管理

3. **数据操作**: `app/services/notes.server.ts`
   - 提供笔记数据操作方法
   - 实现审核功能

## 已实现功能与代码对应关系

### 1. 管理员账户系统

#### 管理员登录 (`app/routes/admin.login.tsx`)

```typescript
// 登录表单处理
export const action = async ({ request }: ActionFunctionArgs) => {
  const formData = await request.formData();
  const username = formData.get("username") as string;
  const password = formData.get("password") as string;

  try {
    const { user } = await adminLogin({ username, password });
    return createAdminSession(user, "/admin");
  } catch (error) {
    return json(
      { error: error instanceof Error ? error.message : "登录失败" },
      { status: 400 }
    );
  }
};
```

#### 用户角色与权限控制 (`app/services/admin.server.ts`)

```typescript
// 权限检查函数
export const requireAdminUser = async (request: Request) => {
  const adminUser = await getLoggedInAdmin(request);
  
  if (!adminUser) {
    throw redirect("/admin/login");
  }
  
  return adminUser;
};

// 超级管理员权限检查
export const requireAdminRole = async (request: Request) => {
  const adminUser = await requireAdminUser(request);
  
  if (adminUser.role !== "admin") {
    throw json({ error: "需要管理员权限" }, { status: 403 });
  }
  
  return adminUser;
};
```

#### 管理员登出 (`app/routes/admin.logout.tsx`)

```typescript
export const action = async ({ request }: ActionFunctionArgs) => {
  return destroyAdminSession(request);
};
```

### 2. 游记审核管理系统

#### 审核列表与筛选 (`app/routes/admin._index.tsx`)

```typescript
export const loader = async ({ request }: LoaderFunctionArgs) => {
  // 认证检查
  const user = await requireAdminUser(request);
  
  // 解析查询参数
  const url = new URL(request.url);
  const status = url.searchParams.get("status") as TravelNoteStatus | undefined;
  const page = Number(url.searchParams.get("page") || "1");
  const search = url.searchParams.get("search") || undefined;
  
  // 获取审核列表
  const { notes, pagination } = await getNotesForReview({ 
    status, 
    page,
    search
  });
  
  return json({ 
    user,
    notes,
    pagination,
    params: { status, page, search }
  });
};
```

#### 审核操作 (`app/routes/admin._index.tsx`)

```typescript
export const action = async ({ request }: ActionFunctionArgs) => {
  // 认证检查
  const user = await requireAdminUser(request);
  
  // 解析表单数据
  const formData = await request.formData();
  const action = formData.get("_action") as string;
  const noteId = formData.get("noteId") as string;
  
  switch (action) {
    case "approve":
      // 批准游记
      await updateNoteStatus({ noteId, status: "approved" });
      return json({ success: true });
      
    case "reject": {
      // 拒绝游记
      const rejectionReason = formData.get("rejectionReason") as string;
      await updateNoteStatus({ noteId, status: "rejected", rejectionReason });
      return json({ success: true });
    }
      
    case "delete":
      // 删除游记（仅管理员）
      if (user.role !== "admin") {
        return json({ error: "没有权限执行此操作" }, { status: 403 });
      }
      await adminDeleteNote(noteId);
      return json({ success: true });
  }
};
```

#### 审核状态更新服务 (`app/services/notes.server.ts`)

```typescript
// 更新笔记状态
export const updateNoteStatus = async ({ 
  noteId, 
  status, 
  rejectionReason 
}: { 
  noteId: string; 
  status: TravelNoteStatus; 
  rejectionReason?: string;
}) => {
  try {
    await db.update(travelNotes)
      .set({ 
        status, 
        rejectionReason: status === 'rejected' ? rejectionReason : null,
        updatedAt: new Date()
      })
      .where(eq(travelNotes.id, noteId));
    
    return { success: true };
  } catch (error) {
    console.error("Error updating note status:", error);
    return { error: "更新游记状态失败" };
  }
};
```

### 3. 管理界面 UI 组件

#### 布局组件 (`app/routes/admin.tsx`)

```tsx
export default function AdminLayout() {
  const { user } = useLoaderData<typeof loader>();
  
  return (
    <div className="min-h-screen bg-gray-50">
      {/* 导航栏 */}
      <nav className="bg-primary-800 text-white shadow-md">
        <div className="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
          <div className="flex justify-between h-16">
            <div className="flex items-center">
              <Link to="/admin" className="flex items-center">
                <MdAdminPanelSettings className="h-8 w-8 mr-2" />
                <span className="font-semibold text-xl">游记管理</span>
              </Link>
            </div>
            
            <div className="flex items-center space-x-4">
              <div className="text-sm">
                {user.name} ({user.role === "admin" ? "管理员" : "审核员"})
              </div>
              <Form method="post" action="/admin/logout">
                <button
                  type="submit"
                  className="text-white hover:text-gray-200 focus:outline-none px-2 py-1 rounded text-sm"
                >
                  <MdLogout className="h-5 w-5" />
                </button>
              </Form>
            </div>
          </div>
        </div>
      </nav>
      
      {/* 主内容区 */}
      <main className="max-w-7xl mx-auto py-6 sm:px-6 lg:px-8">
        <Outlet />
      </main>
    </div>
  );
}
```

#### 审核列表界面 (`app/routes/admin._index.tsx`)

```tsx
export default function AdminIndex() {
  const { notes, pagination, params, user } = useLoaderData<typeof loader>();
  
  // UI 渲染代码...
  return (
    <div>
      {/* 顶部统计卡片 */}
      <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-4 gap-4 mb-6">
        <StatCard
          title="总游记数"
          count={pagination.totalItems}
          icon={<MdArticle className="h-6 w-6 text-blue-600" />}
          color="blue"
        />
        <StatCard
          title="待审核"
          count={pendingCount}
          icon={<MdWarning className="h-6 w-6 text-yellow-600" />}
          color="yellow"
        />
        <StatCard
          title="已通过"
          count={approvedCount}
          icon={<MdCheckCircle className="h-6 w-6 text-green-600" />}
          color="green"
        />
        <StatCard
          title="已拒绝"
          count={rejectedCount}
          icon={<MdCancel className="h-6 w-6 text-red-600" />}
          color="red"
        />
      </div>
      
      {/* 筛选与搜索 */}
      <div className="flex flex-col sm:flex-row justify-between gap-4 mb-6">
        {/* 状态筛选 */}
        <div className="flex gap-2">
          <button
            onClick={() => handleStatusChange("")}
            className={getStatusButtonClass("")}
          >
            全部
          </button>
          <button
            onClick={() => handleStatusChange("pending")}
            className={getStatusButtonClass("pending")}
          >
            <MdWarning className="inline-block mr-1" />
            待审核
          </button>
          {/* 其他状态按钮 */}
        </div>
        
        {/* 搜索框 */}
        <form onSubmit={handleSearch} className="flex">
          <input
            type="text"
            name="searchQuery"
            placeholder="搜索游记标题或内容..."
            className="border border-gray-300 rounded-l-lg px-4 py-2 focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-transparent"
            defaultValue={params.search || ""}
          />
          <button
            type="submit"
            className="bg-blue-600 text-white px-4 py-2 rounded-r-lg hover:bg-blue-700 focus:outline-none"
          >
            <MdSearch className="h-5 w-5" />
          </button>
        </form>
      </div>
      
      {/* 游记列表 */}
      <div className="bg-white shadow-md rounded-lg overflow-hidden">
        {/* 表格或卡片列表... */}
      </div>
      
      {/* 分页控件 */}
      <div className="mt-6">
        <Pagination
          currentPage={pagination.currentPage}
          totalPages={pagination.totalPages}
          onPageChange={handlePageChange}
          isLoading={navigation.state !== "idle"}
        />
      </div>
      
      {/* 拒绝理由模态窗 */}
      {showRejectionModal && (
        <Modal
          title="拒绝原因"
          onClose={() => setShowRejectionModal(false)}
        >
          <div className="mt-4">
            <textarea
              value={rejectionReason}
              onChange={(e) => setRejectionReason(e.target.value)}
              className="w-full border border-gray-300 rounded-lg px-4 py-2 focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-transparent"
              rows={4}
              placeholder="请输入拒绝原因"
            />
          </div>
          <div className="mt-6 flex justify-end gap-3">
            <button
              onClick={() => setShowRejectionModal(false)}
              className="px-4 py-2 border border-gray-300 rounded-lg text-gray-700 hover:bg-gray-50"
            >
              取消
            </button>
            <button
              onClick={handleReject}
              disabled={!rejectionReason.trim() || isSubmitting}
              className="px-4 py-2 bg-red-600 text-white rounded-lg hover:bg-red-700 disabled:bg-red-300"
            >
              确认拒绝
            </button>
          </div>
        </Modal>
      )}
    </div>
  );
}
```

## 数据流程

1. **登录流程**
   - 用户输入管理员账号/密码
   - `admin.login.tsx` 处理登录请求
   - `adminLogin` 函数验证凭据
   - `createAdminSession` 创建会话并重定向到管理界面

2. **审核流程**
   - 管理员/审核员查看待审核游记列表
   - 查看游记详情，包括内容和媒体文件
   - 执行审核操作：批准、拒绝或删除
   - 更新操作通过 `updateNoteStatus` 或 `adminDeleteNote` 函数处理

## 启动与运行

```bash
# 安装依赖
npm install

# 开发模式启动
npm run dev

# 构建生产版本
npm run build

# 启动生产服务器
npm run start
```

## 测试账号

- 管理员账号: admin / admin123
  - 拥有全部权限，包括删除游记

## 路由结构

```
app/routes/
├── admin.tsx           # 管理后台布局组件
│   ├── loader          # 验证管理员登录状态
│   └── UI              # 提供导航和布局
│
├── admin._index.tsx    # 管理后台主页（游记审核列表）
│   ├── loader          # 加载游记列表，支持分页和筛选
│   ├── action          # 处理审核操作
│   └── UI              # 审核界面和控件
│
├── admin.login.tsx     # 管理员登录页面
│   ├── loader          # 检查是否已登录
│   ├── action          # 处理登录请求
│   └── UI              # 登录表单
│
└── admin.logout.tsx    # 管理员登出处理
    └── action          # 销毁管理员会话
```
