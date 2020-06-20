上一节我们了解到`Reconciler`的工作可以分为“递”阶段和“归”阶段。其中“递”阶段会执行`beginWork`，“归”阶段会执行`completeWork`。这一节我们看看“递”阶段的`beginWork`方法究竟做了什么。


## 方法概览

可以从[源码这里](https://github.com/facebook/react/blob/master/packages/react-reconciler/src/ReactFiberBeginWork.old.js#L3040)看到`beginWork`的定义。整个方法大概有500行代码。

从上一节我们已经知道，`beginWork`的工作是传入当前`Fiber节点`，创建子`Fiber`节点，我们从传参来看看具体是如何做的。

### 从传参看方法执行

```js
function beginWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderExpirationTime: ExpirationTime,
): Fiber | null {
  // ...省略函数体
}
```
其中传参：
- current：当前组件对应的`Fiber`节点在上一次更新时的`Fiber`节点
- workInProgress：当前组件对应的`Fiber节点`
- renderExpirationTime：优先级，在讲解`Scheduler`时再讲解

我们知道，组件`mount`时，由于是首次渲染，是不存在当前组件对应的`Fiber`节点在上一次更新时的`Fiber`节点。所以`mount`时`current === null`。

组件`update`时，由于之前已经`mount`过，所以`current !== null`。

所以我们可以通过`current === null ?`来区分组件是`mount`还是`update`。

当一个组件在两次更新中接收的`props`和组件自身`type`都没变化，那么该组件是不需要更新的。

基于此原因，`beginWork`的工作可以分为两部分：

- 第一部分：如果`current`存在可能可以克隆`current.child`作为`workInProgress.child`，这样就能提前跳出第二部分生成新子`Fiber`节点的过程
- 第二部分：根据`tag`不同，创建不同类型的子`Fiber`节点

```js
function beginWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderExpirationTime: ExpirationTime,
): Fiber | null {

  // 第一部分：如果current存在可能存在优化路径，可以复用上一次更新的Fiber节点
  if (current !== null) {
    // ...省略

    // 优化路径，本次更新当前Fiber不需要进行后续处理
    return bailoutOnAlreadyFinishedWork(
      current,
      workInProgress,
      renderExpirationTime,
    );
  } else {
    didReceiveUpdate = false;
  }

  // 第二部分：根据tag不同，创建不同的子Fiber节点
  switch (workInProgress.tag) {
    case IndeterminateComponent: 
      // ...省略
    case LazyComponent: 
      // ...省略
    case FunctionComponent: 
      // ...省略
    case ClassComponent: 
      // ...省略
    case HostRoot:
      // ...省略
    case HostComponent:
      // ...省略
    case HostText:
      // ...省略
    // ...省略其他类型
  }
}
```

## 方法第一部分代码

我们可以看到，满足如下情况时`didReceiveUpdate === false`（即可以直接复用前一次更新的子`Fiber`，不需要新建子`Fiber`）

1. `oldProps === newProps && workInProgress.type === current.type`，即`props`与`fiber.type`不变
2. `updateExpirationTime < renderExpirationTime`，即当前`Fiber`节点优先级不够

```js
if (current !== null) {
    const oldProps = current.memoizedProps;
    const newProps = workInProgress.pendingProps;

    if (
      oldProps !== newProps ||
      hasLegacyContextChanged() ||
      (__DEV__ ? workInProgress.type !== current.type : false)
    ) {
      didReceiveUpdate = true;
    } else if (updateExpirationTime < renderExpirationTime) {
      didReceiveUpdate = false;
      switch (workInProgress.tag) {
        // 省略处理
      }
      return bailoutOnAlreadyFinishedWork(
        current,
        workInProgress,
        renderExpirationTime,
      );
    } else {
      didReceiveUpdate = false;
    }
  } else {
    didReceiveUpdate = false;
  }
```
## 方法第二部分代码

当不满足优化路径时，我们就进入第二部分，新建子`Fiber`。

我们可以看到，根据`fiber.tag`不同，进入不同类型`Fiber`的创建逻辑。

> 可以从[这里](https://github.com/facebook/react/blob/master/packages/react-reconciler/src/ReactWorkTags.js)看到`tag`对应的组件类型

```js
// 第二部分：根据tag不同，创建不同的Fiber节点
switch (workInProgress.tag) {
  case IndeterminateComponent: 
    // ...省略
  case LazyComponent: 
    // ...省略
  case FunctionComponent: 
    // ...省略
  case ClassComponent: 
    // ...省略
  case HostRoot:
    // ...省略
  case HostComponent:
    // ...省略
  case HostText:
    // ...省略
  // ...省略其他类型
}
```

对于我们常见的组件类型，如（`FunctionComponent`/`ClassComponent`/`HostComponent`），最终会进入[reconcileChildren](https://github.com/facebook/react/blob/master/packages/react-reconciler/src/ReactFiberBeginWork.old.js#L232)方法。

## reconcileChildren

从该函数名就能看出这是`Reconciler`模块的核心部分。那么他究竟做了什么呢？

- 对于`mount`的组件，他会创建新的子`Fiber`节点
- 对于`update`的组件，他会将当前组件与该组件在上次更新时对应的`Fiber`节点比较（也就是俗称的`Diff`算法），将比较的结果生成新`Fiber`节点

```js
export function reconcileChildren(
  current: Fiber | null,
  workInProgress: Fiber,
  nextChildren: any,
  renderExpirationTime: ExpirationTime,
) {
  if (current === null) {
    // 对于mount的组件
    workInProgress.child = mountChildFibers(
      workInProgress,
      null,
      nextChildren,
      renderExpirationTime,
    );
  } else {
    // 对于update的组件
    workInProgress.child = reconcileChildFibers(
      workInProgress,
      current.child,
      nextChildren,
      renderExpirationTime,
    );
  }
}
```

从代码可以看出，和`beginWork`一样，他也是通过`current === null ?`区分`mount`与`update`。

不论走哪个逻辑，最终他会生成新的子`Fiber`节点并赋值给`workInProgress.child`，作为下次`beginWork`执行时`workInProgress`的传参。

::: warning 注意
值得一提的是，`mountChildFibers`与`reconcileChildFibers`这两个方法的逻辑基本一致。唯一的区别是：`reconcileChildFibers`会为生成的`Fiber`节点带上`effectTag`属性，而`mountChildFibers`不会。
:::



## effectTag

我们知道，`Reconciler`的工作是在内存中进行，当工作结束后会通知`Renderer`需要执行的`DOM`操作。具体需要执行什么`DOM`操作就是保存在`fiber.effectTag`中。

> 你可以从[这里](https://github.com/facebook/react/blob/master/packages/react-reconciler/src/ReactSideEffectTags.js)看到`effectTag`对应的`DOM`操作

比如：

```js
// DOM需要插入到页面中
export const Placement = /*                */ 0b00000000000010;
// DOM需要更新
export const Update = /*                   */ 0b00000000000100;
// DOM需要插入到页面中并更新
export const PlacementAndUpdate = /*       */ 0b00000000000110;
// DOM需要删除
export const Deletion = /*                 */ 0b00000000001000;
```

> 通过二进制表示`effectTag`，可以方便的使用位操作为`fiber.effectTag`赋值多个`effect`。

按照这么理解，如果要通知`Renderer`将`Fiber`节点对应的`DOM`节点插入页面中，需要满足两个条件：

1. `fiber.stateNode`存在，即`Fiber`节点中保存了对应的`DOM`节点
2. `fiber.effectTag &= Placement !== 0`，即`Fiber`节点存在`Placement effectTag`

那么`mount`时，`fiber.stateNode === null`，且在`reconcileChildren`中调用的`mountChildFibers`不会为`Fiber`节点赋值`effectTag`。那么首屏渲染如何完成呢？

针对第一个问题，`fiber.stateNode`会在`compeleteWork`中创建，我们会在下一节介绍。

第二个问题的答案十分巧妙：假设`mountChildFibers`也会赋值`effectTag`，那么可以预见`mount`时整棵`Fiber`树所有节点都会有`Placement effectTag`。那么`Renderer`在执行`DOM`操作时每个节点都会执行一次插入操作，这样大量的`DOM`操作是极低效的。

为了解决这个问题，在`mount`时只有根`Fiber`节点会赋值`Placement effectTag`，在`Renderer`中只会执行一次插入操作。

::: details 根Fiber节点 Demo
借用上一节的Demo，第一个进入`beginWork`方法的`Fiber`节点就是根`Fiber`节点，你会发现他存在`current`。

所以在`reconcileChildren`时他会走`reconcileChildFibers`逻辑。

而之后通过`beginWork`创建的`Fiber`节点是不存在`current`的。

这也是为什么根`Fiber`节点会赋值`Placement effectTag的原因。

[Demo](https://code.h5jun.com/kexev/edit?html,js,console,output)
:::