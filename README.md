
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

## 高频率对象

### `react` 包

1. `ReactElement` 对象

    > 其 type 定义在`shared`包中.

    所有采用`jsx`语法书写的节点, 都会被编译器转换, 最终会以 `React.createElement(...)` 的方式创建一个 `ReactElement` 对象

    ```js
    export type ReactElement = {
      // 用于辨别ReactElement对象
      $$typeof: any,

      // 内部属性
      type: any, // 表明其种类
      key: any,
      ref: any,
      props: any,

      // ReactFiber 记录创建本对象的Fiber节点, 还未与Fiber树关联之前, 该属性为null
      _owner: any,

      // __DEV__ dev环境下的一些额外信息, 如文件路径, 文件名, 行列信息等
      _store: {validated: boolean, ...},
      _self: React$Element<any>,
      _shadowChildren: any,
      _source: Source,
    };
    ```

    需要特别注意 2 个属性:

    1. `key` 属性在 `reconciler` 阶段会用到, 目前只需要知道所有的 `ReactElement` 对象都有 `key` 属性(且其默认值是 `null`, 这点十分重要, 在`diff算法`中会使用到).

    2. `type` 属性决定了节点的种类:

      - 它的值可以是字符串(代表div,span等 dom 节点), 函数(代表function, class等节点), 或者 react 内部定义的节点类型(portal,context,fragment等)

      - 在 `reconciler` 阶段, 会根据 `type` 执行不同的逻辑(在 `fiber` 构建阶段详细解读).

          - 如 `type` 是一个字符串类型, 则直接使用.

          - 如 `type` 是一个 `ReactComponent` 类型, 则会调用其 `render` 方法获取子节点.

          - 如 `type` 是一个 `function` 类型,则会调用该方法获取子节点

[ReactElement内存结构](./ReactElement.png)

### `react-reconciler` 包

1. `Fiber` 对象

    类型的定义在 [ReactInternalTypes.js](./packages/react-reconciler/src/ReactInternalTypes.js)

    ```js
    // 一个Fiber对象代表一个即将渲染或者已经渲染的组件(ReactElement), 一个组件可能对应两个fiber(current和WorkInProgress)
    // 单个属性的解释在后文(在注释中无法添加超链接)
    export type Fiber = {|
      tag: WorkTag,
      key: null | string,
      elementType: any,
      type: any,
      stateNode: any,
      return: Fiber | null,
      child: Fiber | null,
      sibling: Fiber | null,
      index: number,
      ref:
        | null
        | (((handle: mixed) => void) & { _stringRef: ?string, ... })
        | RefObject,
      pendingProps: any, // 从`ReactElement`对象传入的 props. 用于和`fiber.memoizedProps`比较可以得出属性是否变动
      memoizedProps: any, // 上一次生成子节点时用到的属性, 生成子节点之后保持在内存中
      updateQueue: mixed, // 存储state更新的队列, 当前节点的state改动之后, 都会创建一个update对象添加到这个队列中.
      memoizedState: any, // 用于输出的state, 最终渲染所使用的state
      dependencies: Dependencies | null, // 该fiber节点所依赖的(contexts, events)等
      mode: TypeOfMode, // 二进制位Bitfield,继承至父节点,影响本fiber节点及其子树中所有节点. 与react应用的运行模式有关(有ConcurrentMode, BlockingMode, NoMode等选项).

      // Effect 副作用相关
      flags: Flags, // 标志位
      subtreeFlags: Flags, //替代16.x版本中的 firstEffect, nextEffect. 当设置了 enableNewReconciler=true才会启用
      deletions: Array<Fiber> | null, // 存储将要被删除的子节点. 当设置了 enableNewReconciler=true才会启用

      nextEffect: Fiber | null, // 单向链表, 指向下一个有副作用的fiber节点
      firstEffect: Fiber | null, // 指向副作用链表中的第一个fiber节点
      lastEffect: Fiber | null, // 指向副作用链表中的最后一个fiber节点

      // 优先级相关
      lanes: Lanes, // 本fiber节点的优先级
      childLanes: Lanes, // 子节点的优先级
      alternate: Fiber | null, // 指向内存中的另一个fiber, 每个被更新过fiber节点在内存中都是成对出现(current和workInProgress)

      // 性能统计相关(开启enableProfilerTimer后才会统计)
      // react-dev-tool会根据这些时间统计来评估性能
      actualDuration?: number, // 本次更新过程, 本节点以及子树所消耗的总时间
      actualStartTime?: number, // 标记本fiber节点开始构建的时间
      selfBaseDuration?: number, // 用于最近一次生成本fiber节点所消耗的时间
      treeBaseDuration?: number, // 生成子树所消耗的时间的总和
    |};
    ```

[Fiber结构](./Fiber.png)

2. `Update` 与 `UpdateQueue` 对象

    `fiber` 对象中有一个属性 `updateQueue`, 是一个链式队列(即使用链表实现的队列存储结构)

    ```js
    export type Update<State> = {|
      eventTime: number, // 发起update事件的时间(17.0.2中作为临时字段, 即将移出)
      lane: Lane, // update所属的优先级

      tag: 0 | 1 | 2 | 3, //
      payload: any, // 载荷, 根据场景可以设置成一个回调函数或者对象
      callback: (() => mixed) | null, // 回调函数

      next: Update<State> | null, // 指向链表中的下一个, 由于UpdateQueue是一个环形链表, 最后一个update.next指向第一个update对象
    |};

    // =============== UpdateQueue ==============
    type SharedQueue<State> = {|
      pending: Update<State> | null, // 指向即将输入的update队列. 在class组件中调用setState()之后, 会将新的 update 对象添加到这个队列中来
    |};

    export type UpdateQueue<State> = {|
      baseState: State, // 此队列的基础 state
      firstBaseUpdate: Update<State> | null, // 指向基础队列的队首
      lastBaseUpdate: Update<State> | null, // 指向基础队列的队尾
      shared: SharedQueue<State>, // 共享队列
      effects: Array<Update<State>> | null, // 用于保存有callback回调函数的update对象，在commit后，会依次调用这里的回调函数
    |};
    ```
    [UpdateQueue](./UpdateQueue.png)

3. `Hook` 对象

    `Hook` 用于 `function` 组件中，常用的 `api` 有 `useState` , `useEffect` , `useCallback` 等, 官方一共定义了14 种 `Hook` 类型

    这些 api 背后都会创建一个 `Hook` 对象

    ```js
    export type Hook = {|
      memoizedState: any, // 内存状态, 用于输出成最终的fiber树
      baseState: any, // 基础状态, 当Hook.queue更新过后, baseState也会更新
      baseQueue: Update<any, any> | null, // 基础状态队列, 在reconciler阶段会辅助状态合并
      queue: UpdateQueue<any, any> | null, //  指向一个Update队列
      next: Hook | null, // 指向该function组件的下一个Hook对象, 使得多个Hook之间也构成了一个链表
    |};

    type Update<S, A> = {|
      lane: Lane,
      action: A,
      eagerReducer: ((S, A) => S) | null,
      eagerState: S | null,
      next: Update<S, A>,
      priority?: ReactPriorityLevel,
    |};

    type UpdateQueue<S, A> = {|
      pending: Update<S, A> | null,
      dispatch: (A => mixed) | null,
      lastRenderedReducer: ((S, A) => S) | null,
      lastRenderedState: S | null,
    |};
    ```

    `Hook` 与 `fiber` 的关系:

      在 `fiber` 对象中有一个属性 `fiber.memoizedState` 指向 `fiber` 节点的内存状态. 在 `function` 类型的组件中, `fiber.memoizedState` 就指向 `Hook` 队列( `Hook` 队列保存了 `function` 类型的组件状态)

    [Hook](./Hook.png)

### Scheduler 包

`scheduler` 包负责调度, 在内部维护一个任务队列(`taskQueue`). 这个队列是一个**最小堆数组**(详见React 算法之堆排序), 其中存储了 `task` 对象

1. Task 对象

    ```js
    var newTask = {
      id: taskIdCounter++, // 位移标识
      callback, // task 最核心的字段, 指向react-reconciler包所提供的回调函数
      priorityLevel, // 优先级
      startTime, // 一个时间戳,代表 task 的开始时间(创建时间 + 延时时间)
      expirationTime, // 过期时间
      sortIndex: -1, // 控制 task 在队列中的次序, 值越小的越靠前
    }
    ```
    注意 `task` 中没有 `next` 属性, 它不是一个链表, 其顺序是通过堆排序来实现的(**小顶堆数组, 始终保证数组中的第一个 `task` 对象优先级最高**).

