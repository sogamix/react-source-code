
## 基础包结构

1. `react`

  > `react` 基础包

2. `react-dom`

  > `react` 渲染器，这里指的是浏览器和 `Node` 环境的渲染器

3. `react-reconciler`

  > 管理 `react` 应用的输入、输出，将输入信号转换成输出信号传递给渲染器

  - 接受输入(`scheduleUpdateOnFiber`), 将 `fiber` 树生成逻辑封装到一个回调函数中(涉及 `fiber` 树形结构, `fiber.updateQueue` 队列, 调和算法等),

  - 把此回调函数( `performSyncWorkOnRoot` 或 `performConcurrentWorkOnRoot` )送入 `scheduler` 进行调度

  - `scheduler` 会控制回调函数执行的时机, 回调函数执行完成后得到全新的 `fiber` 树

  - 再调用渲染器(如 `react-dom` , `react-native` 等)将 fiber 树形结构最终反映到界面上

4. `scheduler`

  > 调度中心，控制由 `react-reconciler` 送入的回调函数的执行时机, 在 `concurrent` 模式下可以实现任务分片

  - 核心任务就是执行回调(回调函数由 `react-reconciler` 提供)

  - 通过控制回调函数的执行时机, 来达到任务分片的目的, 实现可中断渲染( `concurrent` 模式下才有此特性)

## 架构分层

此处分层的标准并非官方说法

### 接口层

接口层(api) `react` 包, 平时在开发过程中使用的绝大部分api均来自此包(不是所有). 在 `react` 启动之后, 正常可以改变渲染的基本操作有 3 个.

- `class` 组件中使用 `setState()`

- `function` 组件里面使用 `hook` ,并发起 `dispatchAction` 去改变 `hook` 对象

- 改变 `context` (其实也需要 `setState` 或 `dispatchAction` 的辅助才能改变)

以上 `setState` 和 `dispatchAction` 都由 `react` 包直接暴露. 所以要想 `react` 工作, 基本上是调用 `react` 包的 api 去与其他包进行交互

### 内核层

内核层(core) 整个内核部分, 由 3 部分构成:

1. 调度器 `scheduler` 包, 核心职责只有 1 个, 就是**执行回调**.

  - 把 `react-reconciler` 提供的回调函数, 包装到一个任务对象中.

  - 在内部维护一个任务队列, 优先级高的排在最前面.

  - 循环消费任务队列, 直到队列清空.

2. 构造器 `react-reconciler` 包, 有 3 个核心职责:

  - 装载渲染器, 渲染器必须实现 `HostConfig` 协议(如: `react-dom`), 保证在需要的时候, 能够正确调用渲染器的 api, 生成实际节点(如: `dom节点`).

  - 接收 `react-dom` 包(初次 `render` )和 `react` 包(后续更新 `setState` )发起的更新请求.

  - 将 `fiber` 树的构造过程包装在一个回调函数中, 并将此回调函数传入到 `scheduler` 包等待调度.

3. 渲染器 `react-dom` 包, 有 2 个核心职责:

  - 引导 `react` 应用的启动(通过 `ReactDOM.render`).

  - 实现 `HostConfig` 协议(源码在 ReactDOMHostConfig.js 中), 能够将 `react-reconciler` 包构造出来的
  
  - `fiber` 树表现出来, 生成` dom 节点`(浏览器中), 生成字符串(ssr)

[现将内核 3 个包的主要职责和调用关系, 绘制到一张概览图上:](./react.png)

## 工作循环

1. 任务调度循环

  源码位于 `Scheduler.js`, 它是 `react` 应用得以运行的保证, 它需要循环调用, 控制所有任务(task)的调度.

2. `fiber` 构造循环

  源码位于 `ReactFiberWorkLoop.js`, 控制 `fiber` 树的构造, 整个过程是一个深度优先遍历

两大循环的分工可以总结为: 大循环(任务调度循环)负责**调度task**, 小循环( `fiber` 构造循环)负责**实现task**

1. 输入: 将每一次更新(如: 新增, 删除, 修改节点之后)视为一次更新需求(目的是要更新 `DOM节点`).

2. 注册调度任务: `react-reconciler` 收到更新需求之后, 并不会立即构造 `fiber` 树, 而是去调度中心 `scheduler` 注册一个新任务 `task`, 即把更新需求转换成一个 `task`.

3. 执行调度任务(输出): 调度中心 `scheduler` 通过任务调度循环来执行 `task` ( `task` 的执行过程又回到了`react-reconciler` 包中).

    - `fiber` 构造循环是 `task` 的实现环节之一, 循环完成之后会构造出最新的 `fiber` 树.
    - `commitRoot` 是 `task` 的实现环节之二, 把最新的 `fiber` 树最终渲染到页面上, `task` 完成.
