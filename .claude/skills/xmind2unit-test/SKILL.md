---
name: xmind2unit-test
description: "通用XMind→UI单元测试转化引擎：解析任意XMind，OCR容器树+可执行前置+数据规则断言+结构化选择器，语义描述→可运行Playwright测试。TRIGGER: .xmind文件, /xmind2unit-test"
version: "1.2.0"
allowed-tools:
  - Read
  - Bash
  - Edit
  - Write
  - Agent
  - WebFetch
  - WebSearch
---

# /xmind2unit-test: 通用 XMind → UI 单元测试转化引擎

解析任意业务的 XMind 思维导图文件，通过 OCR + 图像识别关联 UI 截图和设计稿，
将语义化测试描述映射为可执行的 Web 自动化测试代码。

## 触发条件

TRIGGER when:
- 用户提供 `.xmind` 文件路径并要求生成测试用例
- 用户提到从思维导图、脑图、测试用例矩阵生成自动化测试
- 用户执行 `/xmind2unit-test` 命令

SKIP when:
- 纯自然语言描述测试（无 XMind 文件）—— 应使用 /nl2unit-test
- 后端 API 测试、数据库测试、非 Web 相关的测试生成
- 用户仅想查看/编辑 XMind 内容而不生成测试代码

## 行为

### 核心原则

1. **通用引擎**：不硬编码任何业务领域词汇，所有领域知识动态来源于 XMind 文件内容、OCR 结果、外部文档
2. **OCR 桥接**：语义描述 → 提取关键词 → OCR 容器树匹配 → 生成选择器。OCR 无法匹配时才使用语义推断
3. **渐进式确认**：4 个关卡穿插在工作流中，避免一次性信息过载
4. **前置状态自动推导**：从 XMind 路径层级和描述中自动推导隐含的前置条件，生成 beforeEach 和 mock scaffold
5. **数据依赖模板化**：对数据驱动的测试场景自动生成 mock 代码框架

### 工作流概览

```
关卡0(输入) → 阶段1(解析+前置状态提取) → 关卡1(优先级过滤) → 关卡2(框架选择)
→ 阶段2(文档) → 阶段3(OCR结构化分析) → 阶段4(语义映射+前置推导) → 关卡3(映射确认)
→ 阶段5(生成代码含mock) → 关卡4(缺失信息) → 阶段6(输出) → 阶段7(未解决问题)
```

---

### 🚪 关卡 0：接收输入

1. 检查用户消息是否包含 `.xmind` 文件路径
2. 未提供则询问：`请提供需要解析的 XMind 文件路径`
3. 验证文件存在且为 ZIP 格式

---

### 阶段 1：解析 XMind 文件 + 前置状态提取

#### 1.1 提取原始数据
```bash
unzip -p "<file>" content.json | python3
```
- 管道读取，不解压到磁盘
- 同时读取 `metadata.json` 获取创建工具和版本
- 列出 `resources/` 下所有图片文件（`unzip -l`）

#### 1.2 识别文档资源 section

遍历根节点的一级子节点，识别包含外部资源的 section（可能 0 到多个）：

**识别规则**：
- 标题中包含 `https?://` URL
- `attributedTitle` 字段中有链接
- 标题含文档类关键词（中文："文档"/"需求"/"PRD"/"设计"/"交互"/"UI"/"参考"，英文："doc"/"spec"/"design"/"reference"/"link"）

**处理**：
- 提取所有 URL，分类为：需求文档 URL、设计稿 URL（含 MasterGo/Figma 等）、其他 URL
- 该 section 标记为「资源区」，不纳入功能模块树
- 如果没有文档 section，正常跳过，不报错

#### 1.3 构建功能模块与测试用例树

排除资源区后，根节点的直接子节点 = **功能模块**。
递归遍历，叶子节点 = **测试用例**。

对每个叶子节点记录：
```
{
  module: string,           // 所属一级模块名
  path: string[],           // 完整层级路径（根 → 叶子）
  depth: number,            // 深度
  markers: string[],        // XMind markers
  priority: "high" | "medium" | "low",
  hasImageRef: boolean,     // 是否有 image.src 引用
  imageRef: string | null,  // 引用的 resources 文件路径
  note: string | null       // 节点的备注内容
}
```

#### 1.4 提取隐含前置状态（关键增强）

在构建树的同时，从 XMind 节点层级中提取隐含的前置状态。这是实现"先打开弹窗/先切到某 tab/先进入某知识点态"自动补全的基础。

**提取策略（通用，不绑定业务）**：

遍历路径中每个非叶子节点，识别其隐含的页面/状态前置条件。每个前置条件直接映射为**可执行 Playwright 代码**，禁止生成只含注释的占位符。

| 节点标题模式 | 前置类型 | 生成的可执行代码 |
|------------|---------|----------------|
| 含"弹窗"/"dialog"/"modal"/"popup"/"浮层" | open-dialog | `await page.getByRole('button', { name: '<入口文字>' }).click();` `await expect(page.getByRole('dialog')).toBeVisible();` |
| 含"tab"/"Tab"/"标签页" | switch-tab | `await page.getByRole('tab', { name: '<tab名>' }).click();` `await expect(page.getByRole('tab', { name: '<tab名>', selected: true })).toBeVisible();` |
| 含"页面"/"页"/"详情"（非根层级） | navigate | `await page.goto('/<路由>'}` 或 `await page.getByRole('link', { name: '<页面名>' }).click(); await page.waitForURL('**/<路径模式>**');` |
| 含"首次"/"第一次"/"无X" | mock-empty | `await page.route('**/api/<推断路径>**', route => route.fulfill({ body: JSON.stringify({ data: [], total: 0 }) }));` |
| 含"非首次"/"已有X"/"已选X" | mock-data | `await page.route('**/api/<推断路径>**', route => route.fulfill({ body: JSON.stringify({ data: <fixture>, total: <N> }) }));` |
| 含"断网"/"弱网"/"加载失败" | network-error | `await page.route('**/api/<推断路径>**', route => route.abort('failed'));` 或 `await context.setOffline(true);` |
| 含"空"/"无数据"/"无内容" | empty-state | `assert: await expect(page.locator('[class*="empty"]')).toBeVisible();` 或 mock API 返回空列表 |
| 含"X=Y"模式 | config | `await page.evaluate(() => { localStorage.setItem('<X>', '<Y>'); });` |
| 含"已诊断"/"未诊断"/"已选择"/"未选择" | state | `await page.route('**/api/<状态接口>**', route => route.fulfill({ body: JSON.stringify({ status: '<状态值>' }) }));` |

**入口按钮推导规则**（当弹窗/浮层节点缺少入口描述时）：
- 从弹窗节点同级或上级查找「入口」子节点
- 从入口子节点的叶子描述中提取触发按钮文字
- 如果 XMind 中未出现入口按钮名称 → 从该模块关联截图的 OCR 容器树中查找关联按钮
- 实在无法推测时使用 `// FIXME: 请补充打开 <弹窗名> 的入口按钮`

**路径层级累积规则**：

一个叶子节点的完整前置条件 = 路径上所有节点隐含前置状态的累积。
例如路径 `模块A > 弹窗B > 非首次进入 > 点击确定`：
- `弹窗B` 节点 → 前置：打开弹窗B（生成 `click()` + `expect(dialog).toBeVisible()`）
- `非首次进入` 节点 → 前置：mock 历史数据为非首次状态（生成 `page.route()` mock）
- `点击确定` 叶子 → 操作：点击确定按钮

产物：每个叶子节点附带 `preconditions: { type, code, priority: "required"|"optional" }[]` 包含所有需要的前置步骤及其对应可执行代码。

#### 1.5 优先级定级规则（通用）

| Marker | 优先级 |
|--------|--------|
| `star-red` / `priority-1` | high |
| `star-orange` / `priority-2` | medium |
| 其他 marker 或无 marker | 默认 medium，完全孤立的描述性叶子归为 low |

也可以结合节点深度判断：越深的叶子越具体，越可能是核心测试用例。

#### 1.6 展示解析摘要

用从实际数据动态生成的摘要，不预设任何栏目名称：

```
## XMind 解析摘要

- 文件: <文件名>
- 创建工具: <工具名 版本>
- 外部资源:
  - 需求文档: <N> 个URL
  - 设计稿: <M> 个URL
  - UI 截图: <K> 张
- 功能模块: <X> 个
- 测试用例总数: <Y>（🔴高: a, 🟡中: b, 🟢低: c）
- 含前置条件的用例: <P> 个（从路径自动推导）

### 模块分布
| 模块 | 高 | 中 | 低 | 含前置 | 合计 |
|------|----|----|----|--------|------|
| <动态列出的模块名> | ... | ... | ... | ... | ... |
```

---

### 🚪 关卡 1：优先级过滤

展示摘要后立即询问：

```
生成哪些优先级的测试用例？
[A] 仅高优先级（默认，推荐）- 核心流程
[B] 高 + 中优先级
[C] 全部
```

默认选 A，用户直接回复可确认。

---

### 🚪 关卡 2：目标项目框架

询问用户（与 XMind 内容无关）：

```
目标项目使用的前端框架？
[A] Vue 3（默认）→ Playwright + Vue 3
[B] React → Playwright + React  
[C] 纯 Playwright E2E（默认，推荐）- 框架无关黑盒测试，跨项目通用

脚本语言：TypeScript（默认）/ JavaScript？
```

默认：纯 Playwright E2E + TypeScript（框架无关，跨项目通用）。

---

### 阶段 2：读取外部文档（可选）

对阶段 1 收集到的需求文档 URL，逐条通过 WebFetch 获取：

1. **可访问** → 提取关键信息：
   - 页面 URL / 路由路径
   - 功能交互流程描述
   - API 接口和数据格式（对数据 mock 模板化尤其重要）
   - 异常状态和边界条件
   - 页面元素命名（如有）

2. **不可访问**（内网限制/需登录） → 标记状态：
   ```
   ⚠️ 无法自动访问: <URL>（原因：需内网权限）
   这不会阻塞流程，后续可通过阶段 7 补充。
   ```

3. 将成功获取的文档内容与 XMind 模块做关键词交叉匹配（文本相似度），建立文档-模块关联。
   - **API 接口信息**单独提取，用于阶段 5 的数据 mock 模板化

---

### 阶段 3：UI 图像分析（OCR + 结构化交互区域映射）

**目标**：为每张 UI 截图建立**结构化的 UI 容器树**——不仅识别"有哪些文字"，更关键的是识别"每段文字在什么交互容器中（按钮/弹窗/侧边栏/Tab/列表项）"。这个容器树是语义映射的核心桥梁。

#### 3.1 提取截图
```bash
unzip -o "<xmind_file>" "resources/*" -d /tmp/xmind_ocr_<timestamp>/
```
- 将截图提取到临时目录
- 记录图片文件名、大小、尺寸

#### 3.2 逐张图像分析（OCR 文字 → 交互区域 → 容器树）

##### 3.2.1 识别页面/组件类型

从截图整体布局判断：

- `full-page`（全屏页面）— 有完整页头+内容区
- `modal`（居中弹窗）— 有遮罩层 + 居中面板 + 标题 + 关闭 + 底部按钮组
- `bottom-sheet`（底部抽屉）— 从屏幕底部滑出，有拖拽手柄
- `side-panel`（侧边面板）— 固定在左侧或右侧的面板
- `toast`（短暂提示）— 浮层，通常无操作元素
- `popover`（锚点浮层）— 靠近某个触发元素的小浮层
- `tab-bar`（Tab 栏）— 水平或垂直排列的切换项组
- `list-panel`（列表面板）— 可滚动的列表区域

##### 3.2.2 交互容器识别（核心增强）

对截图中的每个可见元素，判断它属于哪类**交互容器**。
这是将"点击右上角章节选择"映射到正确选择器的关键依据。

**容器类型定义**（通用，不绑定任何业务）：

| 容器类型 | 视觉特征 | Playwright 选择器策略 |
|---------|---------|---------------------|
| `button` | 有边框/底色/圆角矩形的可点击区域，文字居中 | `getByRole('button', { name: '...' })` |
| `primary-button` | 高亮/填充色的主操作按钮 | `getByRole('button', { name: '...' })` |
| `secondary-button` | 边框按钮/灰色按钮 | `getByRole('button', { name: '...' })` |
| `text-button` | 纯文字无边框的可点击文本 | `getByText('...')` |
| `input` | 下划线/边框输入区，有 placeholder | `getByRole('textbox')` / `getByPlaceholder('...')` |
| `select` | 有下拉箭头 + 当前选中值，点击弹出选项 | `getByRole('combobox')` |
| `tab` | 水平/垂直排列的切换项，有选中态（高亮下划线或底色） | `getByRole('tab', { name: '...' })` |
| `dialog-header` | 弹窗顶部区域：标题文字 + 关闭 × 按钮 | `getByRole('dialog').locator('header, [class*="header"], [class*="title"]')` |
| `dialog-body` | 弹窗中部滚动内容区，含选择器/列表/表单等 | `getByRole('dialog').locator('[class*="body"], [class*="content"]')` |
| `dialog-footer` | 弹窗底部按钮组（取消+确定） | `getByRole('dialog').locator('footer, [class*="footer"], [class*="actions"]')` |
| `sidebar` | 页面左侧固定面板，含导航/菜单/状态信息 | `page.locator('[class*="sidebar"], [class*="left-panel"], aside, nav')` |
| `main-content` | 页面主内容区（右侧/中间） | `page.locator('main, [class*="content"], [class*="main"]')` |
| `list-item` | 列表中的单行/单个卡片，可点击或展示 | `getByRole('listitem')` / `page.locator('[class*="item"]')` |
| `grid-item` | 网格中的单个卡片（如教材封面网格） | `page.locator('[class*="card"], [class*="grid"] > *')` |
| `badge-tag` | 小标签/徽标（如状态、优先级、计数） | `page.locator('[class*="tag"], [class*="badge"]')` |
| `icon-button` | 纯图标按钮（×、←、🔍、⚙ 等，无文字或仅有 tooltip） | `getByRole('button').locator('svg, img, [class*="icon"]')` |
| `backdrop` | 弹窗/浮层背后的半透明遮罩 | `page.locator('[class*="overlay"], [class*="backdrop"], [class*="mask"]')` |
| `toast` | 短暂浮层提示条 | `page.locator('[class*="toast"], [class*="notification"], [class*="snackbar"]')` |
| `loading` | 加载中骨架屏/spinner | `page.locator('[class*="loading"], [class*="skeleton"], [class*="spinner"]')` |
| `empty-state` | 空数据占位图+文案 | `page.locator('[class*="empty"]')` |
| `progress-bar` | 进度条/掌握度指示条 | `page.locator('progress, [role="progressbar"], [class*="progress"]')` |
| `tooltip` | 鼠标悬停后出现的提示浮层 | `page.locator('[class*="tooltip"]')` |

**识别方法**：
1. 观察文字所在区域的视觉边界（边框、底色、阴影变化）
2. 观察文字旁是否有交互标识（箭头▼、关闭×、选中高亮边框下划线）
3. 判断文字在布局中的语义角色（标题 vs 选项 vs 数据值 vs 提示）
4. 结合组件类型（page/modal/sidebar）确定容器归属

##### 3.2.3 输出：UI 交互容器树

对每张截图构建层级化的容器树：

```
[截图: resources/xxx.png]
页面类型: modal

容器树:
dialog (modal, 居中, 宽约80vw)
├── dialog-header
│   ├── [text] "选择教材" (标题, 居中)
│   └── [icon-button] × (关闭, 右上角, 无文字)
├── dialog-body (可滚动)
│   ├── [select] "省份" → "城市" (并列两个选择器)
│   ├── [tab-bar] 年级选择器 (单选, 横向排列)
│   │   ├── [tab] "一年级"  
│   │   ├── [tab] "二年级" → 当前选中(高亮)
│   │   └── ...
│   ├── [tab-bar] 学期选择器 (单选, 横向排列)
│   │   ├── [tab] "上学期"
│   │   ├── [tab] "下学期"
│   │   └── [tab] "全一册"
│   └── [grid] 教材版本选择器
│       ├── [grid-item] 封面图 + "人教版" (选中态)
│       ├── [grid-item] 封面图 + "北师大版"
│       └── ...
└── dialog-footer
    ├── [secondary-button] "取消"
    └── [primary-button] "确定"
```

**容器树的使用**：
当语义映射阶段遇到 "点击右上角的教材版本切换" 时：
1. 在容器树中搜索右上角区域 → 找到 dialog-header 中的 icon-button
2. 搜索"教材版本"关键词 → 在 dialog-body > grid 中找到 grid-item 列表
3. 生成选择器：`getByRole('dialog').locator('header').getByRole('button')`

##### 3.2.4 状态差异识别

同一功能区域的不同状态截图，对比识别：
- 选中态 vs 未选中态（高亮颜色/边框/下划线差异）
- 有数据 vs 空数据（列表项 vs 空状态占位）
- 加载中 vs 已加载（spinner/skeleton vs 实际内容）
- 不同权限/配置下的界面差异

#### 3.3 关联截图到模块/用例

使用启发式方法将截图关联到 XMind 节点：

1. **路径关键词匹配**：截图中识别到的文字与模块/用例的节点标题做文本匹配
2. **XMind image 引用**：如果节点有 `image.src` 指向 `xap:resources/xxx.png`，直接关联
3. **内容相似度**：OCR 文字集合与模块下所有文本的 Jaccard 相似度
4. 无法关联的截图标记为「通用参考」

#### 3.4 MasterGo / 设计稿（可选）

- WebFetch 尝试获取 MasterGo URL
- 可访问 → 提取：组件命名、颜色/字体规范、交互标注、页面层级结构
- 不可访问 → 告知用户可手动提供信息，不阻塞

产物：每个截图 → 结构化 UI 交互容器树（作为语义映射的核心输入）

---

### 阶段 4：语义映射 + 前置状态推导

将 XMind 叶子节点的语义描述转化为可执行的操作序列。
**输入**：XMind 路径 + OCR 容器树 + 阶段1提取的隐式前置条件
**输出**：含前置步骤的操作序列

#### 4.0 前置状态自动推导与合并（关键增强）

**核心约束：所有前置步骤必须生成可执行代码，禁止生成仅含注释的 `// TODO 打开弹窗` 类占位符。**

**4.0.1 三层前置来源**（从粗到细）：

| 层级 | 来源 | 代码生成风格 |
|------|------|-------------|
| L1 路径前置 | 阶段1.4从路径节点提取 | 直接使用阶段1.4的可执行代码模板生成 |
| L2 兄弟共享前置 | 同一父节点下所有测试共享的状态 | 提取到 `test.describe('...', () => { test.beforeEach(async ({ page }) => { ... }) })` |
| L3 描述内嵌前置 | 叶子描述中显式的前提 | 生成 `test.step('前置: xxx', async () => { ... })` 在 beforeEach 或 test 体内 |

**4.0.2 前置步骤合成规则**：

对每个测试用例，按照 L1 → L2 → L3 的顺序合成完整前置。每一步都必须是可执行 Playwright 代码：

```
原始: XMind 路径 "模块A > 弹窗B > 非首次进入 > 点击确定"
描述: "点击确定关闭弹窗并进入章节选择页面"

L1 路径前置（从阶段1.4模板生成，从路径节点提取）:
  open-dialog: await page.getByText('年级教材').click();  // 弹窗B入口（从兄弟节点「入口」提取）
               await expect(page.getByRole('dialog')).toBeVisible();
  mock-data:   await page.route('**/api/教材/selection**', route =>
                 route.fulfill({ body: JSON.stringify({ grade: 2, semester: 1 }) }));

L2 兄弟共享前置（从同级节点场景推断）:
  父节点 "非首次进入" → 所有子节点共享 mock 历史数据
  提取到 test.describe('非首次进入', () => { beforeEach(() => { ... }) })

L3 描述内嵌前置:
  无明显前置
  
合成最终 beforeEeach（L1 + L2 + L3 合并去重）:
  test.describe('弹窗B > 非首次进入', () => {
    test.beforeEach(async ({ page }) => {
      // 步骤1: 导航到模块页 (L1)
      await page.goto('/模块A');
      // 步骤2: mock 历史数据 (L1 + L2)
      await page.route('**/api/教材/selection**', route =>
        route.fulfill({ body: JSON.stringify({ grade: 2, semester: 1 }) }));
      // 步骤3: 打开弹窗 (L1)
      await page.getByText('年级教材').click();
      await expect(page.getByRole('dialog')).toBeVisible();
    });

    test('[P1] 点击确定关闭弹窗并进入章节选择页面', async ({ page }) => {
      // 仅包含该用例特有步骤
      await page.getByRole('button', { name: '确定' }).click();
      await expect(page.getByRole('dialog')).toBeHidden();
      await expect(page).toHaveURL(/chapter/);
    });
  });
```

**4.0.3 可执行前置代码生成强制规则**：

每条前置必须按以下规则转换为代码，违反即为生成错误：

| 前置类型 | 强制输出 | 禁止输出 |
|---------|---------|---------|
| open-dialog | `await page.getBy<Selector>(...).click(); await expect(page.getByRole('dialog')).toBeVisible();` | `// 需要先打开弹窗` |
| switch-tab | `await page.getByRole('tab', { name: '...' }).click();` | `// 切换到对应 Tab` |
| navigate | `await page.goto('...');` 或 `await page.locator('...').click(); await page.waitForURL(...);` | `// TODO: 页面导航` |
| mock-empty | `await page.route(...)` 具体路由拦截代码 | `// TODO: mock 空数据` |
| mock-data | `await page.route(...)` 含 fixture JSON | `// TODO: mock 数据` |
| network-error | `await page.route(...).abort('failed')` | `// 模拟网络异常` |
| empty-state | `await expect(page.locator('[class*="empty"]')).toBeVisible();` | `// 断言空状态` |

**4.0.4 兄弟节点前置共享（beforeEach 自动分组）**：

如果同一父节点下的多个测试用例共享相同的前置条件，将前置提取到 `test.describe` 的 `beforeEach` 中：

```
示例: 父节点 "非首次进入" 下 3 个用例共享前置
→ test.describe('非首次进入', () => {
    test.beforeEach(async ({ page }) => {
      // 共享前置: mock 非首次数据 + 打开弹窗
      await mockHistoryData(page, 'existing');
      await page.getByText('年级教材').click();
      await expect(page.getByRole('dialog')).toBeVisible();
    });
    test('[P1] 用例1...', ...);
    test('[P1] 用例2...', ...);
    test('[P1] 用例3...', ...);
  })
```

#### 4.1 动作词提取

扫描叶子节点描述 D，匹配动作关键词。

**中文动作词表**（通用，不区分业务）：

| 关键词 | 操作类型 | Playwright 实现 |
|--------|---------|----------------|
| 点击 / 点 / 单击 / 选择 / 勾选 / 切换 / 按 | click | `click()` |
| 双击 | dblclick | `dblclick()` |
| 输入 / 填写 / 键入 / 录入 / 搜索 | input | `fill()` / `type()` |
| 打开 / 进入 / 跳转 / 前往 / 访问 / 进 | navigate | `goto()` 或 `click()` + `waitForNavigation()` |
| 展示 / 显示 / 可见 / 出现 / 弹出 / 呈现 | visible | `toBeVisible()` |
| 不展示 / 隐藏 / 不可见 / 消失 / 关闭(弹窗) | hidden | `toBeHidden()` / `not.toBeVisible()` |
| 默认 / 默认选中 / 默认展示 / 初始 | default | 初始状态断言（不执行操作）|
| 高亮 / 选中态 / 激活 / 选中 | highlight | CSS class / aria 断言 |
| 滑动 / 滚动 / 上滑 / 下滑 / 左滑 / 右滑 | scroll | `scroll()` / `swipe()` |
| 关闭 / 退出 / 取消(弹窗关闭) | close | `click()` 关闭按钮 + waitFor 关闭 |
| 返回 / 后退 / 上一页 | back | `click()` 返回按钮 |
| 拖拽 / 拖动 | drag | `drag()` / `dragTo()` |
| 悬停 / 浮入 / hover | hover | `hover()` |
| 长按 | longpress | `click({ delay: 500 })` 或 `longPress()` |
| 缩放 | zoom | `pinch()` 手势 |
| 刷新 | reload | `reload()` |
| 加载 / 加载中 | loading | 断言 loading 状态可见 |
| 空 / 空状态 / 空数据 | empty | 断言空状态文案或无数据元素 |
| 网络 / 断网 / 弱网 | network | 网络条件模拟 |
| 复用 / 参考 | reference | 关联已有测试或逻辑 |
| 支持 / 不支持 | toggle | 功能开关/条件断言 |

**英文动作词表**：

| Keywords | Action |
|----------|--------|
| click / tap / press / select / choose / toggle / pick | click |
| double click | dblclick |
| type / input / enter / fill / write / search | input |
| navigate / go to / open / visit / access / enter | navigate |
| display / show / appear / visible / render / pop up | visible |
| hide / disappear / invisible / dismiss / close | hidden |
| default / initially / by default | default |
| highlight / active / selected / focused | highlight |
| scroll / swipe / drag | scroll |
| hover / mouse over | hover |
| zoom / pinch | zoom |
| reload / refresh | reload |
| loading / skeleton / spinner | loading |
| empty / no data / blank | empty |
| offline / network error / disconnected | network |
| reuse / same as | reference |

#### 4.2 目标元素识别（利用容器树）

**核心规则：选择器必须反映元素在容器树中的位置。禁止在容器信息可用时输出裸 `getByText()`。**

**选择器输出优先级（强制）**：

| 优先级 | 条件 | 选择器形式 | 示例 |
|--------|------|-----------|------|
| 1（最高） | 容器类型 + 文字均已知 | `parentContainer.getByRole('button', { name: '文字' })` | `dialog.getByRole('button', { name: '确定' })` |
| 2 | 容器类型已知，文字部分已知 | `parentContainer.locator('[class*="xxx"]').getByText(/文字/)` | `sidebar.locator('[class*="item"]').getByText(/章节/)` |
| 3 | 仅容器类型已知 | `parentContainer.getByRole('...')` | `dialog.locator('footer').getByRole('button').first()` |
| 4 | 仅文字已知 | `page.getByRole('button', { name: '文字' })` | `page.getByRole('button', { name: '确定' })` |
| 5（最低） | 容器和文字均未知 | `FIXME` 占位 | `// FIXME: 请确认选择器 [原因]` |

**禁止的选择器格式（当容器信息可用时）**：

| 禁止 | 原因 | 应替换为 |
|------|------|---------|
| `page.getByText('确定')` | 丢失容器信息；"确定"可能出现在页面多处 | `dialog.getByRole('button', { name: '确定' })` |
| `page.getByText('一年级')` | 不知道是 tab 还是 list-item | `page.getByRole('tab', { name: '一年级' })` |
| `page.locator('button').getByText('关闭')` | 太泛，不用容器限定 | `dialog.locator('header').getByRole('button')` |

按优先级降级尝试：

**策略 A — OCR 容器树直接匹配（置信度: HIGH）**

从描述中提取目标词 → 在阶段3的 OCR 容器树中搜索 → 找到匹配的文字节点 → 获取其容器类型 + 完整路径 → 生成容器限定选择器：

```
描述: "点击确定按钮"
容器树中搜索 "确定" → 
  找到: dialog-footer > primary-button "确定"
→ 判断: 这是弹窗底部的确定主按钮
→ 选择器: page.getByRole('dialog').getByRole('button', { name: '确定' })
✅ 置信度 HIGH — 容器类型(primary-button) + 父容器(dialog-footer) + 文字(确定) 三重匹配
```

```
描述: "点击右上角章节选择"
容器树中搜索右上角区域 → dialog-header 中的 icon-button 或 text-button
同时搜索 "章节" → dialog-body 中有 tab(含章节名)
→ 判断: 右上角指的是 header 区域，而"章节选择"指的是 tab 选择器
→ 选择器: page.getByRole('dialog').locator('header').getByRole('button')
✅ 置信度 HIGH — 位置(header右上角) + 容器类型(icon-button) 匹配
```

```
描述: "查看侧边栏第3个小节的掌握度进度条"
容器树中搜索(sidebar + 掌握度 + 进度条) →
  sidebar > list-panel > list-item[2] > progress-bar "掌握度"
→ 选择器: page.locator('[class*="sidebar"]').getByRole('listitem').nth(2).locator('progress')
✅ 置信度 HIGH — 容器类型(sidebar->list-item->progress-bar) + 位置(第3个) 匹配
```

**策略 B — 位置词 + 容器区域匹配（置信度: MEDIUM）**

没有精确容器树匹配时，利用位置词缩小到容器区域：

| 描述位置 | 容器树定位策略 | 选择器模板 |
|---------|-------------|-----------|
| 右上角 / 右上 | 查找 header / top-right 区域中的 button / icon-button | `page.locator('header').getByRole('button')` |
| 左上角 / 左上 | 查找 header / top-left 区域中的元素 | `page.locator('header').locator('[class*="left"]')` |
| 左侧 / 左边 / 侧边栏 | 查找 sidebar 容器下的 list-item / tab / text | `page.locator('[class*="sidebar"]').getByText(/关键词/)` |
| 右侧 / 主内容区 | 查找 main-content 容器下的元素 | `page.locator('main, [class*="content"]').getByRole(...)` |
| 底部 / 下方 | 查找 footer / bottom 区域中的 button | `page.locator('footer').getByRole('button', { name: /关键词/ })` |
| 弹窗内 | 在整个 dialog 容器子树中搜索 | `page.getByRole('dialog').getByRole(...)` |
| 弹窗标题 | 在 dialog-header 容器中搜索 | `page.getByRole('dialog').locator('header').getByText(/关键词/)` |
| 弹窗底部按钮 | 在 dialog-footer 容器中搜索 | `page.getByRole('dialog').locator('footer').getByRole('button')` |
| 弹窗外 / 遮罩 | 指向 backdrop 容器类型 | `page.locator('[class*="overlay"], [class*="backdrop"]')` |
| 列表中 / 列表项 | 在 list-panel > list-item 子树中搜索 | `page.getByRole('listitem').filter({ hasText: /关键词/ })` |
| popover / 浮层 | 在 popover / tooltip 容器中搜索 | `page.locator('[class*="popover"], [class*="tooltip"]').getByText(...)` |

**策略 C — 层级上下文推断（置信度: LOW）**

```
描述中的操作对象在容器树中找不到明确匹配 → 
利用 XMind 路径上下文推断最可能的容器类型 → 
在生成的代码中用 FIXME 行标注推理依据:
  // FIXME: 推断容器为 <容器类型>，来源: <XMind路径上下文>
  // 请确认选择器: <AI推断的选择器>
```

#### 4.3 对每个测试用例执行映射

```
输入: {
  path: string[], markers: string[], priority: string,
  preconditions: string[],          // 阶段1.4推导
  sharePreconditions: boolean,      // 是否兄弟共享
  hasDataDependency: boolean        // 是否有数据依赖
}

输出:
{
  confidence: "HIGH" | "MEDIUM" | "LOW",
  description: string,
  xmindPath: string,
  priority: string,
  preconditions: [                  // 完整前置步骤
    { action: "navigate" | "click" | "mock", target: string, value: string }
  ],
  sharePreconditions: boolean,      // true → 提取到 beforeEach
  steps: [
    {
      action: "navigate" | "click" | "input" | "assert" | "wait" | "scroll" | "hover" | "conditional",
      target: string,               // 选择器（含容器路径）
      value: string | null,
      assertion: string | null,
      containerType: string | null, // 容器类型(button/dialog/tab/...)
      confidence: "HIGH" | "MEDIUM" | "LOW",
      ocrSource: string | null,
      fixme: string | null
    }
  ],
  dataDependency: {                 // 数据依赖信息（阶段5用于mock）
    type: "api" | "state" | "config" | "none",
    keywords: string[],
    suggestedMockFields: string[]
  } | null,
  screenshotRefs: string[]
}
```

#### 4.4 数据依赖检测

在语义映射时同时检测测试用例是否涉及数据依赖。

**数据依赖关键词检测**（通用，不绑定业务）：

| 关键词模式 | 数据依赖类型 | mock 策略 |
|-----------|------------|----------|
| "数据"/"接口"/"API"/"返回" | api | `page.route()` 拦截 |
| "状态"/"开关"/"配置"/"设置" | config | mock 全局状态/配置 |
| "映射"/"关系"/"对应" | mapping | mock 映射表数据 |
| "命中"/"命中率"/"匹配" | match-rate | mock 命中率数值 |
| "频"/"频率"/"次数" | frequency | mock 频次数据 |
| "掌握度"/"进度"/"百分比"/"率"/"分" | score | mock 分数/百分比数据 |
| "列表"/"清单"/"数组"/"集合" | list | mock 列表数据 |
| "详情"/"信息"/"属性" | entity | mock 实体详情数据 |
| "判断"/"条件"/"规则" | rule | mock 条件判断逻辑 |
| "推荐"/"建议"/"策略" | strategy | mock 推荐结果 |

**检测步骤**：
1. 扫描叶子描述 + 路径节点标题 → 匹配数据关键词
2. 匹配到 → 提取关键词，设置 `hasDataDependency: true`
3. 标记 `dataDependency.type` 和 `suggestedMockFields`

产物：数据依赖检测结果，传递给阶段 5 用于生成 mock scaffold。

#### 4.5 数据规则断言模板化（关键增强）

将数据依赖型用例从 "mock + 文本可见性断言" 升级为 **规则关联的结构化断言**。
**禁止**仅生成 `await expect(page.getByText('xxx')).toBeVisible()` 后结束。

##### 4.5.1 规则模式 → 断言模板（通用）

对每个检测到数据依赖的用例，根据描述中的规则关键词自动推导结构化断言：

| 规则关键词模式 | 断言模板 | 示例 |
|-------------|---------|------|
| "取 N 条"/"取 Top N"/"最多 N 条" | `count` | `await expect(page.locator('<list-容器>.list-item')).toHaveCount(N);` |
| "不足补齐"/"不足取"/"不足 N 取 M" | `count + dual-fixture` | 生成 2 组 fixture（不足+充足），各自断言计数和数据来源 |
| "展示"/"不展示"/"有"/"无"/"可见"/"不可见" | `visibility-toggle` | 生成正反两套 `test()`：`test('有数据时展示', ...)` + `test('无数据时不展示', ...)` |
| "命中率"/"掌握度"/"百分比"/"正确率" X% | `numeric-range` | `await expect(page.locator('<数值元素>')).toContainText(/\d{1,3}%/);` 或 `await expect(page.locator('progress')).toHaveValue(X/100);` |
| "倒序"/"顺序"/"降序"/"升序" | `sorted-order` | `const items = await page.locator('<list-item>').all(); const texts = await Promise.all(items.map(i => i.textContent())); expect(texts.sort(comparator)).toEqual(texts);` |
| "高亮"/"选中态"/"激活" | `css-class` | `await expect(page.locator('<元素>')).toHaveClass(/active|selected|highlight/);` |
| "同步"/"更新"/"联动" | `multi-locator` | 在操作前后各断言一次，对比两个 `locator` 的值是否一致或变化 |
| "映射"/"对应"/"关联" | `conditional` | `const mapping = await getMapping(page); await expect(page.locator('<v2>')).toHaveText(mapping.expected);` |
| "切换"/"变更后会..." | `state-change` | `const before = await page.locator('<元素>').textContent(); await page.locator('<触发>').click(); await expect(page.locator('<元素>')).not.toHaveText(before!);` |
| "复用"/"参考" | `reference-test` | 引用已有测试的逻辑，生成 `test.step` 复用或提取共享函数 |
| "默认"/"默认选中"/"初始" | `default-assert` | 不执行操作，仅断言初始状态：`await expect(page.locator('<元素>')).toHaveAttribute('checked');` |

##### 4.5.2 正反场景自动拆解

如果描述中出现条件分支模式，自动拆解为正反两个 `test()`：

```
描述: "有映射关系展示教材，无映射关系不展示"
      ↓
  test('[P1] 有映射关系时展示教材版本', ...)  // 正常场景
  test('[P1] 无映射关系时不展示教材版本', ...)  // 边界场景
```

**条件拆解触发词**：
- "有X / 无X" / "有X 反之不展示"
- "支持 / 不支持" / "开启 / 关闭"
- "大于/小于/超过/不足 阈值 N"
- "满足/不满足条件 X"

每个拆解出的 test 自动配置对应的 mock scenario 参数（如 `'default'` vs `'empty'`）。

##### 4.5.3 数量驱动断言

对描述中包含数量规则的用例，强制生成精确的数量断言：

| 描述关键词 | 断言 |
|-----------|------|
| "N 条"/"N 个"/"N 项" | `toHaveCount(N)` |
| "至少 N 条" | `locator.count() >= N` 或 `toHaveCount(greaterThanOrEqual(N))` |
| "最多 N 条" | `locator.count() <= N` |
| "不足 N 条取 M 条补齐" | count: N（总览）+ M（补齐来源标识） |

##### 4.5.4 联动场景结构化断言

对联动/"同步"/"更新"类场景，强制使用「操作前快照 → 操作 → 操作后断言」三步结构：

```typescript
test('[P1] 切换知识点后掌握度数值同步更新', async ({ page }) => {
  // 操作前: 记录当前值
  const beforeText = await page.locator('[class*="mastery-score"]').textContent();
  // 操作: 触发切换
  await page.getByRole('tab', { name: '知识点B' }).click();
  await page.waitForLoadState('networkidle');
  // 操作后: 断言值已变化
  await expect(page.locator('[class*="mastery-score"]')).not.toHaveText(beforeText!);
  // 断言: 非空
  await expect(page.locator('[class*="mastery-score"]')).not.toBeEmpty();
});
```

##### 4.5.5 断言优先级

对于同一条测试用例，按以下次序生成断言语句：
1. **存在性/可见性**（基础断言，`toBeVisible` / `toBeHidden`）
2. **数量/计数**（数据规则断言，`toHaveCount` / `count()` 比较）
3. **文本/内容**（语义断言，`toHaveText` / `toContainText`）
4. **状态/样式**（交互断言，`toHaveClass` / `toHaveAttribute`）
5. **联动对比**（变化断言，前后值一致性或变化）

产物：每个含数据依赖的测试用例附带结构化断言序列。

### 🚪 关卡 3：模块确认 + 映射确认

在生成代码之前，展示以下信息请用户确认：

#### 3.1 模块选择
```
将按以下优先级为各模块生成测试：
| 模块 | 高优 | 中优 | 低优 | 共享前置 | 数据依赖 | 选中 |
|------|------|------|------|---------|---------|------|
| <模块名> | <N> | <M> | <L> | <S> 组 | <D> 个 | <N+M+L> |
...

默认全部选中。
```

#### 3.2 前置状态推导确认（新增）
```
从 XMind 路径自动推导了以下前置条件组：

| # | 前置描述 | 推导来源 | 影响用例数 |
|---|---------|---------|-----------|
| 1 | 打开弹窗B → 进入非首次状态 | 弹窗B > 非首次进入 | 12 |
| 2 | 切换到Tab X → mock空数据 | Tab X > 空状态 | 4 |
...

请确认这些前置条件是否正确？格式: "1: 修正描述" 或 "全部确认"
```

#### 3.3 低置信度映射确认
```
⚠️ 以下用例的语义映射需要确认或修正：

| # | 用例 | AI 推断 | 容器类型 | 置信度 |
|---|------|---------|---------|--------|
| 1 | ... | getByText('七年级') | tab | MEDIUM |
| 2 | ... | FIXME: 目标URL未知 | navigate | LOW |
...

请逐一确认或修正。
```

---

### 阶段 5：生成测试代码（含 mock 模板化）

#### 5.1 框架模板选择

根据关卡 2 的选择。默认 Vue 3 + Playwright E2E：

```typescript
import { test, expect } from '@playwright/test';
import { <mockFunctions> } from './mocks/<module>.mock';

test.describe('<一级模块名>', () => {
  // ========== 共享前置（从XMind路径自动推导，禁止空注释） ==========
  test.describe('<子模块> > <场景（含共享前置）>', () => {
    test.beforeEach(async ({ page }) => {
      // 以下每个步骤都是可执行代码，禁止 // TODO 注释
      // L1 路径前置: 导航
      await page.goto('/<路由>');
      // L1 路径前置: mock 数据（如有数据依赖）
      await mock<DataName>API(page, 'default');
      // L1 路径前置: 打开弹窗/切换tab（从路径节点推断）
      await page.getByRole('button', { name: '<入口按钮名>' }).click();
      await expect(page.getByRole('dialog')).toBeVisible();
      // L3 描述内嵌前置（如有）
    });

    test('[P1] <叶子节点原始描述>', async ({ page }) => {
      // 仅包含操作和断言（仅包含该用例特有步骤，共享前置已在 beforeEach 中）
    });
  });

  // ========== 无共享前置的独立测试 ==========
  test.describe('<其他场景>', () => {
    test('[P2] <独立用例描述>', async ({ page }) => {
      // 独立用例: 如果同组仅一个用例或前置不共享，前置内联在 test 体内
      await page.goto('/<路由>');
      // 操作 & 断言
    });
  });
});
```

#### 5.2 文件命名规则

从 XMind 一级模块名自动生成文件名，输出到用户指定目录（默认 `tests/`）。

#### 5.3 选择器生成优先级（强制容器优先）

生成任何选择器前，必须遍历阶段3的容器树确认元素的容器归属。禁止容器信息可用时退化为 `getByText()`。

1. ✅ 容器路径 + 角色 + 文字 → `getByRole('dialog').getByRole('button', { name: '确定' })`
2. ✅ 容器区域 + 角色 → `getByRole('dialog').locator('header').getByRole('button')`
3. ✅ 容器区域 + 文字 → `locator('[class*="sidebar"]').getByText('章节')`
4. ⚠️ 仅角色 → `page.getByRole('button', { name: '...' })` （仅当 OCR 无容器信息时使用）
5. ⚠️ 仅位置推断 → `locator('header').getByText(/.../)` （仅当容器树无匹配且无角色时使用）
6. ❌ 裸 `getByText()` → 禁止使用，至少包裹在容器定位器内
7. ❌ 无法确定 → `// FIXME: 选择器 [原因]` 占位

#### 5.4 测试名和标签

- 测试名：`[P<1/2/3>] <叶子节点原始描述>`
- HIGH 置信度：无额外注释
- MEDIUM：`// 推断容器: <容器类型>`
- LOW：`// FIXME: <原因> - 请手动确认`

#### 5.5 数据 Mock 模板化生成（关键增强）

对阶段 4.4 检测到的数据依赖，自动生成 mock scaffold，**不依赖任何业务词汇**。

##### 5.5.1 Mock 生成策略

| 数据依赖类型 | 自动生成内容 |
|------------|------------|
| `api` | `page.route()` 拦截对应 URL + 响应 JSON 骨架 |
| `config` | 全局 config/开关 mock 函数 |
| `mapping` | 映射表 mock 数据 + 查找函数 |
| `match-rate` | 命中率数值 mock（0-100%） |
| `frequency` | 频次统计数据 mock |
| `score` | 分数/百分比数据 mock |
| `list` | 列表数据 mock（含翻页） |
| `entity` | 实体详情 mock |
| `rule` | 条件判断 mock |
| `strategy` | 推荐结果 mock |

##### 5.5.2 Mock 代码模板（通用，自动适配）

对每个数据依赖用例，生成以下 mock scaffold：

```typescript
// ========== 自动生成的 Mock 帮助函数 ==========
// 推导依据: XMind路径包含 <节点关键词> , 描述含 <数据关键词>

/**
 * Mock <数据描述> 的 API 响应
 * @param scenario 场景: 'default' | 'empty' | 'error' | 'single' | 'multiple'
 */
async function mock<DataName>API(
  page: Page,
  scenario: 'default' | 'empty' | 'error' | 'single' | 'multiple' = 'default'
) {
  // 根据 scenario 返回不同数据结构
  const fixtures = {
    default: { /* 从 OCR 截图 / 需求文档推断的典型数据结构 */ },
    empty: { data: [], total: 0 },
    error: { status: 500, body: { message: '服务器错误' } },
    single: { /* 单条数据 */ },
    multiple: { /* 多条数据 */ },
  };

  await page.route('**/api/<推断的API路径>**', async (route) => {
    await route.fulfill({
      status: 200,
      contentType: 'application/json',
      body: JSON.stringify(fixtures[scenario]),
    });
  });
}

// 在 beforeEach 或测试中使用
test.beforeEach(async ({ page }) => {
  await mock<DataName>API(page, 'default');
});
```

##### 5.5.3 常见数据场景的通用 mock 值规则

**不需要知道具体业务数据，但可以推断数据类型**：

| 描述中出现 | 推断字段类型 | Mock 示例值 |
|-----------|------------|-----------|
| "百分比"/"率"/"掌握度"/"进度" | number (0-100) | `Math.floor(Math.random() * 100)` |
| "名称"/"标题"/"标签" | string | `"示例名称"` |
| "显示"/"展示"/"可见" | boolean | `true` / `false` |
| "数量"/"次数"/"个数" | number | `Math.floor(Math.random() * 100)` |
| "列表"/"数组" | Array | `[{ ... }, { ... }]` |
| "是否"/"是否开启" | boolean | `true` |
| "得分"/"评分"/"分值" | number (0-100) | `85` |
| "颜色"/"状态色"/"图标色" | string (hex) | `"#52c41a"` |
| "排序"/"倒序"/"顺序" | string enum | `"asc"` / `"desc"` |
| "级别"/"等级" | string enum | `"高"` / `"中"` / `"低"` |

##### 5.5.4 Mock 文件组织

当同一模块存在多个数据依赖用例时，生成独立的 `mocks/` 文件：

```
tests/
├── <module>.spec.ts
├── mocks/
│   └── <module>.mock.ts     # 模块专用 mock 帮助函数
```

#### 5.6 生成规范

- 每个 `test()` 独立可运行，禁止依赖执行顺序
- `beforeEach` 必须含可执行代码，禁止 `// TODO 需要先打开弹窗` 类注释占位
- 数据 mock 不写死具体业务值，用描述性占位 + scenario 参数化（`'default' | 'empty' | 'error' | 'single' | 'multiple'`）
- **容器优先选择器**：`parentContainer.getByRole()` > `parentContainer.getByText()` > `page.getByRole()` > `page.getByText()`
- **禁止裸 getByText**：当 OCR 容器树有信息时，`getByText()` 必须包裹在容器定位器内
- **强制数据断言**：含数量/规则/条件关键词的用例，必须生成结构化断言（count, visibility-toggle, numeric-range, sorted-order 等），禁止仅生成 `toBeVisible()`
- 不写业务注释，仅保留技术性注释（容器类型、OCR引用、FIXME）
- HIGH/MEDIUM 置信度选择器直接使用，LOW 置信度用 `// FIXME:` 占位

---

### 🚪 关卡 4：缺失信息确认

扫描生成的测试代码，提取所有占位符，汇总展示：

```
⚠️ 以下信息需要补充后才能运行测试：

[缺少的页面 URL]
| # | 测试文件 | 测试名 | 目标页面 | 当前占位符 |
|---|---------|--------|----------|-----------|
| 1 | xxx.spec.ts | ... | 进入学习流程页面 | TODO: 学习流程URL |
...

[不确定的元素选择器]
| # | 测试文件 | 测试名 | 原始描述 | AI推断(含容器类型) |
|---|---------|--------|----------|-----------------|
| 1 | xxx.spec.ts | ... | 点击右上角... | getByRole('button') in dialog-header |
...

[Mock API 路径待确认]
| # | Mock 函数 | 当前占位 API 路径 | 推导依据 |
|---|----------|-----------------|---------|
| 1 | mockDataAPI | TODO: /api/xxx | XMind路径含 "数据+映射" |
...

[无法访问的外部资源]
| # | URL | 类型 | 失败原因 |
|---|-----|------|---------|
| 1 | https://docs.xxx | 需求文档 | 需内网权限 |
...
```

用户补充后，更新对应测试文件中的占位符。

---

### 阶段 6：写入与汇总

将测试代码写入文件后，展示最终摘要：

```
## ✅ 测试生成完成

| 项目 | 值 |
|------|-----|
| 输入文件 | <xmind文件名> |
| 优先级过滤 | <仅高/高+中/全部> |
| 目标框架 | <Vue3/React/Playwright> |
| 模块数 | <N> |
| 生成用例数 | <M> |
| 置信度分布 | HIGH: x, MEDIUM: y, LOW: z |
| 共享前置组 | <P> 组 beforeEach |
| 数据Mock | <Q> 个 mock 函数 |
| 输出目录 | <path> |

### 文件列表
- <path>/<module1>.spec.ts (N 用例, M mock)
- <path>/mocks/<module1>.mock.ts (Q 个 mock 函数)
...

### 运行测试
npx playwright test <path>/
```

---

### 阶段 7：未解决问题汇总

```
## 📋 待补充信息

### 无法访问的文档
- [ ] <URL>（需求文档）

### 待确认的 FIXME
- [ ] <file>.spec.ts:45 — 选择器 FIXME
- [ ] <file>.spec.ts:120 — URL FIXME
- [ ] <file>.mock.ts:10 — API 路径 FIXME

建议补充信息后重新执行 /xmind2unit-test。
```

---

## 约束

### 通用性约束
- 不在指令或生成的代码中硬编码任何业务领域词汇
- 领域知识仅来自三处：XMind 文件内容、OCR 容器树、外部文档
- 功能模块名称、测试用例描述完全保留 XMind 原始内容

### 技术约束
- XMind 解析使用 `unzip -p` 管道，不落盘
- OCR 容器树通过 Read 工具读取截图实现（Claude 多模态）
- 优先语义化选择器：`getByRole()` > `getByText()` > `getByLabel()` > `locator()`
- 不引入项目未安装的测试依赖（默认 `@playwright/test`）
- Mock 代码不写死业务具体值，使用 scenario 参数化 + 通用数据类型推断

### 流程约束
- 每个阶段结束后展示进度摘要
- 4 个关卡处必须暂停等待用户输入
- 资源不可达（文档/设计稿）不阻塞流程，在阶段 7 汇总
- 低置信度映射标记 FIXME 而非臆造选择器
- 前置状态自动推导结果需在关卡3展示给用户确认

### 文件操作约束
- 仅在关卡确认后写入测试文件
- 数据 mock 多时自动拆分到独立 `mocks/` 文件

## 示例（供参考，不硬编码）

用户: `/xmind2unit-test xmind/某业务测试.xmind`

Skill 执行流程:
1. 关卡0：验证文件存在
2. 阶段1：解析 XMind → 展示摘要（5模块, 120用例, 36个含前置, 45个含数据依赖, 3张截图）
3. 关卡1：用户选择 [A] 仅高优先级 → 筛选 48 个用例
4. 关卡2：用户选择 [A] Vue 3 + TypeScript
5. 阶段2：尝试访问需求文档 URL
6. 阶段3：读取 3 张截图 → 构建 OCR 容器树 → 关联到模块
7. 阶段4：48 个用例执行语义映射 → HIGH: 30, MEDIUM: 12, LOW: 6；推导出 8 组共享前置
8. 关卡3：用户确认前置条件正确，对 LOW 项手动修正
9. 阶段5：生成 5 个 spec 文件 + 3 个 mock 文件（含参数化 scenario）
10. 关卡4：2 个页面 URL + 3 个 API 路径待补充
11. 阶段6：写入文件，展示最终摘要
12. 阶段7：列出未解决问题
