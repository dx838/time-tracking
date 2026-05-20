# 懒加载页面预加载执行单

## 文档信息

- [ ] 文档状态：执行前
- [ ] 文档类型：How-to 执行指南
- [ ] 创建日期：2026-05-20
- [ ] 目标读者：后续实现者、代码审查者、回归验证者
- [ ] 目标：在不破坏现有首页、托盘、挂件和页面数据体验的前提下，减少首次点击 `History / Settings / App Mapping / Data` 时的短暂加载感
- [ ] 存放位置：`docs/working/`，完成后按文档卫生规则归档到 `docs/archive/`

## 背景

当前 `Dashboard` 直接进入主包，打开主界面时通常能立即显示。

其他核心页面通过 `React.lazy` 动态加载：

- `History`
- `Data`
- `Settings`
- `About`
- `AppMapping`

这能保护首页启动速度和主包体积，但代价是首次点击某些页面时可能出现短暂 `Suspense fallback` 或页面内部 loading。

当前仓库已经有部分数据预热：

- `Settings` 使用 bootstrap cache
- `App Mapping` 使用 classification bootstrap cache
- `Dashboard` 和 `History` 使用 snapshot cache
- `Data` 已有一个专门的组件 chunk 预加载 effect，但位置分散

本执行单优先解决 chunk 首次加载造成的闪烁，不直接重写页面数据加载策略。

## 成功标准

- [ ] 冷启动停留首页约 2 秒后，首次点击 `History` 基本不出现 `加载中`
- [ ] 冷启动停留首页约 2 秒后，首次点击 `Settings` 基本不出现 `加载中`
- [ ] 冷启动停留首页约 2 秒后，首次点击 `App Mapping` 基本不出现 `加载中`
- [ ] 冷启动停留首页约 2 秒后，首次点击 `Data` 的组件级 `Suspense fallback` 不明显
- [ ] 冷启动后立刻点击其他页面时，仍允许短暂 loading，不为了消灭 loading 阻塞首页
- [ ] 首页首次显示速度不能明显变慢
- [ ] 托盘打开主界面不能重新出现菜单闪烁或窗口异常
- [ ] 挂件展开、收起、拖动、打开主窗口行为不能变化
- [ ] Settings 和 App Mapping 的未保存改动保护逻辑不能变化
- [ ] `npm run check:frontend` 通过
- [ ] `npm run check:bundle` 通过，主包不能因为静态 import 懒加载页面而明显增大

## 非目标

- [ ] 不把懒加载页面改成静态 import
- [ ] 不移除 `<Suspense fallback>`
- [ ] 不改变页面路由、导航、保存确认、脏状态判断
- [ ] 不改变 read model 的数据语义
- [ ] 不改变 SQLite 查询结果、缓存失效策略、tracking 刷新策略
- [ ] 不新增 UI 视觉样式
- [ ] 不新增 toast 或用户可见提示
- [ ] 不为了预加载失败打断用户操作
- [ ] 不在本执行单内优化 About 页面，除非验证发现它也明显影响体验

## Owner 判断

- [ ] 预加载调度 owner：`src/app/services/`
  - 理由：这是应用壳层的跨页面启动后编排，不属于某个 feature 私有逻辑
- [ ] 页面组件仍归各自 feature owner
  - `features/history/components/History`
  - `features/data/components/Data`
  - `features/settings/components/Settings`
  - `features/classification/components/AppMapping`
- [ ] 数据预热 owner 保持现状
  - `app/services/startupPrewarmService.ts` 继续编排现有启动预热
  - `features/*/services/*Cache.ts` 继续拥有具体 cache
- [ ] 禁止把预加载服务放进 `shared/*`
  - 理由：它依赖具体应用页面，不是稳定跨 feature 通用能力
- [ ] 禁止把页面数据读取直接塞进 `AppShell.tsx`
  - 理由：`AppShell` 应保持编排层，不吸收厚业务逻辑

## 风险控制原则

- [ ] 只预加载 chunk，不挂载页面组件
- [ ] 只在首页和基础运行时稳定后预加载
- [ ] 预加载必须可取消
- [ ] 预加载失败只记录 warning
- [ ] 预加载必须顺序执行，避免同时解析多个大 chunk
- [ ] 预加载不读写用户数据
- [ ] 数据预热如果补充，必须作为第二阶段单独验证
- [ ] 首屏体验优先于首次点击其他页面体验

## 阶段 0：实施前确认

- [ ] 确认当前文件状态没有未理解的用户改动
  - 命令：`git status --short`
- [ ] 确认当前懒加载入口仍在 `src/app/AppShell.tsx`
  - 检查 `lazy(() => import(...))`
- [ ] 确认 `Data` 仍有单独预加载 effect
  - 目标：后续替换为统一服务
- [ ] 确认现有启动预热入口
  - `src/app/services/startupPrewarmService.ts`
- [ ] 确认 Settings cache 入口
  - `src/features/settings/services/settingsBootstrapCache.ts`
  - `src/features/settings/services/settingsBootstrapService.ts`
- [ ] 确认 Classification cache 入口
  - `src/features/classification/services/classificationBootstrapCache.ts`
  - `src/features/classification/services/classificationService.ts`
- [ ] 确认 History snapshot cache 入口
  - `src/features/history/services/historySnapshotCache.ts`
- [ ] 确认 Data heatmap cache 入口
  - `src/features/data/services/dataReadModel.ts`

## 阶段 1：新增页面 chunk 预加载服务

### 目标

- [ ] 新增一个只负责懒加载页面 chunk 预加载的 app service
- [ ] 把预加载调度从 `AppShell.tsx` 中抽出来，避免壳层继续变厚
- [ ] 为调度逻辑提供可测试依赖注入

### 文件

- [ ] 新增：`src/app/services/viewChunkPreloadService.ts`
- [ ] 新增：`tests/viewChunkPreloadService.test.ts`
- [ ] 修改：`package.json`

### 服务设计

- [ ] 定义 `PreloadableView`

```ts
export type PreloadableView = "history" | "settings" | "mapping" | "data";
```

- [ ] 定义默认 view loader

```ts
const DEFAULT_VIEW_CHUNK_LOADERS: Record<PreloadableView, () => Promise<unknown>> = {
  history: () => import("../../features/history/components/History"),
  settings: () => import("../../features/settings/components/Settings"),
  mapping: () => import("../../features/classification/components/AppMapping"),
  data: () => import("../../features/data/components/Data"),
};
```

- [ ] 定义调度参数

```ts
export interface LazyViewChunkPreloadOptions {
  views?: PreloadableView[];
  initialDelayMs?: number;
  staggerMs?: number;
  idleTimeoutMs?: number;
}
```

- [ ] 默认参数建议

```ts
views: ["history", "settings", "mapping", "data"]
initialDelayMs: 1200
staggerMs: 200
idleTimeoutMs: 1500
```

### 调度行为

- [ ] 首次等待 `initialDelayMs`
- [ ] 到时间后优先使用 `requestIdleCallback`
- [ ] 浏览器不支持时 fallback 到 `window.setTimeout`
- [ ] 每次只加载一个 view chunk
- [ ] 一个 chunk 完成或失败后，再等待 `staggerMs` 进入下一个
- [ ] 任一 chunk 失败不影响后续 chunk
- [ ] 返回 cleanup 函数
- [ ] cleanup 后未执行的任务不再执行
- [ ] cleanup 后正在执行的 promise 不需要强行 abort，但完成后不能继续调度后续任务

### requestIdleCallback 类型处理

- [ ] 不直接假设 TypeScript 环境一定有 `window.requestIdleCallback`
- [ ] 使用窄类型声明或内部 helper

建议内部类型：

```ts
type IdleWindow = Window & typeof globalThis & {
  requestIdleCallback?: (callback: () => void, options?: { timeout: number }) => number;
  cancelIdleCallback?: (handle: number) => void;
};
```

### 测试要求

- [ ] 测试按配置顺序预加载
- [ ] 测试单个 loader 失败后仍继续下一个
- [ ] 测试 cancel 后后续 loader 不再执行
- [ ] 测试默认参数包含 `history / settings / mapping / data`
- [ ] 测试 warning 可注入，不把错误抛给调用方
- [ ] 测试不依赖真实浏览器 idle callback

### package.json

- [ ] 新增脚本

```json
"test:preload": "node --experimental-strip-types --experimental-specifier-resolution=node tests/viewChunkPreloadService.test.ts"
```

- [ ] 把 `test:preload` 加入 `check:frontend`
  - 建议放在 `test:startup` 后、`test:ui-smoke` 前

### 阶段 1 验收

- [ ] `npm run test:preload` 通过
- [ ] `npm run check:architecture` 通过
- [ ] `npm run build` 通过

## 阶段 2：接入 AppShell

### 目标

- [ ] 用统一预加载服务替换 `AppShell.tsx` 中 Data 专用预加载 effect
- [ ] 保持 `React.lazy` 入口不变
- [ ] 保持 `Suspense fallback` 不变
- [ ] 保持当前导航、dirty confirm、toast、dialog 逻辑不变

### 文件

- [ ] 修改：`src/app/AppShell.tsx`

### 改动步骤

- [ ] 引入服务

```ts
import { scheduleLazyViewChunkPreload } from "./services/viewChunkPreloadService";
```

- [ ] 将 `didPreloadDataViewRef` 改为更准确的命名

```ts
const didPreloadLazyViewsRef = useRef(false);
```

- [ ] 删除 Data 专用 effect

```ts
void import("../features/data/components/Data");
```

- [ ] 新增统一 effect

```ts
useEffect(() => {
  if (!classificationReady || didPreloadLazyViewsRef.current) return undefined;
  didPreloadLazyViewsRef.current = true;

  return scheduleLazyViewChunkPreload({
    views: ["history", "settings", "mapping", "data"],
    initialDelayMs: 1200,
    staggerMs: 200,
    idleTimeoutMs: 1500,
  });
}, [classificationReady]);
```

### 注意事项

- [ ] 不要把 `History / Settings / AppMapping / Data` 改成静态 import
- [ ] 不要在 effect 里读取页面内部状态
- [ ] 不要在 effect 里触发 toast
- [ ] 不要把预加载状态放进 React state
- [ ] 不要让预加载结果影响导航渲染条件

### 阶段 2 验收

- [ ] `npm run test:startup` 通过
- [ ] `npm run test:ui-smoke` 通过
- [ ] `npm run build` 通过
- [ ] `npm run check:bundle` 通过

## 阶段 3：手动验证第一轮

### 冷启动后等待验证

- [ ] 完全退出应用
- [ ] 启动应用并停留 `Dashboard`
- [ ] 等待 2 秒
- [ ] 点击 `History`
  - 期望：没有明显 `加载中`
- [ ] 返回 `Dashboard`
- [ ] 点击 `Settings`
  - 期望：没有明显 `加载中`
- [ ] 返回 `Dashboard`
- [ ] 点击 `App Mapping`
  - 期望：没有明显 `加载中`
- [ ] 返回 `Dashboard`
- [ ] 点击 `Data`
  - 期望：没有组件级 `Suspense fallback`，允许局部图表数据轻微刷新

### 冷启动后立即点击验证

- [ ] 完全退出应用
- [ ] 启动应用后立即点击 `History`
  - 期望：允许短暂 loading
  - 期望：首页启动没有被预加载阻塞
- [ ] 完全退出应用
- [ ] 启动应用后立即点击 `Settings`
  - 期望：允许短暂 loading
  - 期望：页面最终正常显示

### 托盘与挂件验证

- [ ] 关闭到托盘
- [ ] 从托盘左键打开主界面
  - 期望：不出现托盘菜单矩形闪烁
- [ ] 最小化到挂件
- [ ] 展开挂件
  - 期望：不出现明显矩形阴影残留
- [ ] 从挂件打开主界面
  - 期望：主界面正常显示
- [ ] 从挂件暂停/恢复追踪
  - 期望：行为不变

### 未保存改动验证

- [ ] 进入 `Settings`
- [ ] 修改任意设置但不保存
- [ ] 导航到其他页面
  - 期望：仍出现原有未保存确认
- [ ] 取消导航
  - 期望：停留在 Settings，改动仍在
- [ ] 进入 `App Mapping`
- [ ] 修改任意映射但不保存
- [ ] 导航到其他页面
  - 期望：仍出现原有未保存确认
- [ ] 取消导航
  - 期望：停留在 App Mapping，改动仍在

## 阶段 4：决定是否补 Data 热力图预热

### 进入条件

仅当阶段 1 到阶段 3 完成后，仍观察到 `Data` 首次点击存在明显体验问题，才进入本阶段。

- [ ] 确认 Data 闪烁来自热力图数据，而不是组件 chunk
- [ ] 确认 History/Data 现有 snapshot cache 已命中
- [ ] 确认补充数据预热不会明显增加启动后 SQLite 负载

### 文件

- [ ] 修改：`src/features/data/services/dataReadModel.ts`
- [ ] 修改：`src/app/services/startupPrewarmService.ts` 或新增轻量 app 编排服务
- [ ] 修改或新增对应测试

### 候选实现

- [ ] 在 `dataReadModel.ts` 中新增：

```ts
export async function prewarmRecentDataHeatmapCache(nowMs: number = Date.now()) {
  return loadDataHeatmapSnapshot("recent", nowMs);
}
```

- [ ] 在 app 层调度中安排到 chunk 预加载之后
- [ ] 必须低优先级执行
- [ ] 失败只 warning
- [ ] 不阻塞页面导航

### 阶段 4 验收

- [ ] `npm run test:data` 通过
- [ ] `npm run test:startup` 通过
- [ ] `npm run check:frontend` 通过
- [ ] 手动确认启动后 CPU/磁盘没有明显异常抬升

## 阶段 5：完整验证

### 必跑

- [ ] `npm run check:frontend`

### 需要时追加

- [ ] 如果改动触及 Rust、Tauri runtime 或窗口生命周期，追加 `npm run check:rust`
- [ ] 如果改动触及架构边界脚本或 package 脚本异常，追加 `npm run check`
- [ ] 如果准备合并到发布线，追加 `npm run check:full`

### 构建输出检查

- [ ] 查看 Vite 输出，确认页面仍是独立 chunk
- [ ] 确认 `index` 主 chunk 没有明显吃进 `History / Data / Settings / AppMapping`
- [ ] 确认 `check:bundle` 没有预算失败

## 回滚方案

### 低风险回滚

- [ ] 从 `AppShell.tsx` 移除 `scheduleLazyViewChunkPreload` effect
- [ ] 恢复原先 Data 专用预加载 effect，或暂时不做任何预加载
- [ ] 保留新服务文件但不调用，或在同一变更中删除
- [ ] 移除 `test:preload` 脚本和测试文件，如果服务也删除

### 必须回滚的信号

- [ ] 首页首次显示明显变慢
- [ ] 冷启动后 CPU/磁盘占用明显异常
- [ ] 托盘打开主界面重新出现异常闪烁
- [ ] 挂件窗口行为异常
- [ ] Settings/App Mapping 的未保存确认失效
- [ ] `check:bundle` 显示主包明显增大
- [ ] 预加载 warning 高频出现且无法解释

## 代码审查清单

- [ ] 是否保留了所有 `React.lazy`
- [ ] 是否没有新增页面静态 import
- [ ] 是否没有把 feature 业务逻辑放进 `app/services`
- [ ] 是否没有把 app 编排能力放进 `shared`
- [ ] 是否没有改变现有 cache 的失效语义
- [ ] 是否没有新增用户可见提示
- [ ] 是否没有新增全局状态或 React state 来保存预加载结果
- [ ] 是否有 cancel 能力
- [ ] 是否有失败兜底
- [ ] 是否有测试覆盖顺序、失败继续、取消
- [ ] 是否通过 `check:architecture`
- [ ] 是否通过 bundle 检查

## 最终完成标准

- [ ] 阶段 1 完成
- [ ] 阶段 2 完成
- [ ] 阶段 3 手动验收完成
- [ ] 阶段 4 已明确执行或明确跳过
- [ ] 阶段 5 验证完成
- [ ] 若进入发布或合并前状态，执行文档状态更新为“已完成”
- [ ] 完成后将本文移动到 `docs/archive/`，或保留在 `docs/working/` 直到对应实现合并

