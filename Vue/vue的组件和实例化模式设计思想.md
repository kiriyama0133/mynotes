---
date: 2026-08-21
tags:
  - vue
  - components
---
我在学习vue的源码的时候有时候会被Component和Instance的关系绕进去，因此编写次笔记用于学习相关的架构设计和思想，本笔记锚定的是runtime-core模块内容，总结的是组件抽象、实例化和递归渲染的内容。

## Component

官方术语里这个叫做组件定义或者组件选项对象，我一般叫他组件原型，里面保存的是静态配置，它是先从Component -> VNode -> instance的顺序在工作

```ts
export interface Component {
  setup?: (setupProps: SetupProps, setupContext: SetupContext) => any;
  render?: () => VNode;
  expose: Expose;
  data?: () => Record<string, any>;
  props?: string[] | Record<string, any>;
  mounted?: (proxy: any) => void;
}
```

vnode.type指向静态的Component，vnode.component指向的是运行时的Instance组件，instance.type又指回Component，instance.vnode指回外壳Vnode。

在 initProps 和 updateProps 中，通过 instance.type.props读取声明的字段名单，用来区分外部传入的属性属于 instance.props 还是instance.attrs

defineProps

data在setupComponent中调用Component.data获取初始化状态对象，并用reactive包装后赋给instance.data。mounted在setupComponent中个会被包装并推入instance.m生命周期队列，执行Component.setup(instance.props, setupContext)，产出响应式状态setupState或者渲染反函数render，setupComponent中赋给instance.render。


```ts
  if (Component.data) {
    instance.data = reactive(Component.data());
  }
    if (Component.mounted) {
    instance.m.push(() => {
      Component.mounted?.call(instance.proxy, instance.proxy);
    });
  }
  if (Component.setup) {
    const setupContext = {
      attrs: instance.attrs,
      slots: instance.slots,
      emit: instance.emit,
      expose(exposed = {}) {
        instance.exposed = exposed;
      },
    };
    setCurrentInstance(instance); // setup-entry
    let setupResult;
    try {
      setupResult = Component.setup(instance.props, setupContext);
    } finally {
      setCurrentInstance(null); // setup-out
    }
    if (isFunction(setupResult)) {
      instance.render = setupResult;
    } else if (isObject(setupResult)) {
      instance.setupState = proxyRefs(setupResult);
    }
  }
```

## ComponentInstance

是框架内部维护的组件的运行时的状态，一个组件从生到死的所有动态数据都挂在它身上，贯穿整个生命周期。

```ts
export interface ComponentInstance {
  vnode: VNode;
  data: any;
  slots: Slots;
  props: Record<string, any>;
  attrs: Record<string, any>;
  emit: Emit;
  exposed: Record<string, unknown>;
  proxy: any;
  update: Function | null;
  type: Component;
  setupState: Record<string, any>;
  render: Function | null;
  subTree: VNode | null;
  isMounted: boolean;
  parent: ComponentInstance | null;
  provides: Record<PropertyKey, unknown>;
  bm: LifecycleHooksArray;
  m: LifecycleHooksArray;
  bu: LifecycleHooksArray;
  u: LifecycleHooksArray;
  bum: LifecycleHooksArray;
  um: LifecycleHooksArray;
  isUnmounted: boolean;
  effect: ReactiveEffect | null;
}
```

在初始化阶段setupComponent，他参与了状态存储，保存了自身的响应式数据data、setupState，以及从父组件接受到的拆分好的props、attrs、slots。

```ts
setupResult = Component.setup(instance.props, setupContext);
```

参与渲染代理instance.proxy，通过Proxy，把setupState、data、props、以及`$emit、$attrs、$slots`统一代理到一个对象上，作为render函数的this和第一个参数。

```ts
instance.proxy = new Proxy(instance, {
    get(target, key) {
      const { setupState, data, props, attrs, slots, emit } = target;
      if (typeof key === 'symbol') {
        return;
      }
      if (hasOwn(setupState, key)) {
        return setupState[key];
      }
      if (hasOwn(data, key)) {
        return data[key];
      }
      if (hasOwn(props, key)) {
        return props[key];
      }
      if (key === '$emit') {
        return emit;
      }
      if (key === '$attrs') {
        return attrs;
      }
      if (key === '$slots') {
        return slots;
      }
    },
    set(target, key, value) {
      const { setupState, data, props } = target;
      if (typeof key === 'symbol') {
        return true;
      }
      if (hasOwn(setupState, key)) {
        setupState[key] = value;
        return true;
      }
      if (hasOwn(data, key)) {
        data[key] = value;
        return true;
      }
      if (hasOwn(props, key)) {
        console.warn('props is readonly');
        return true;
      }
      return true;
    },
  });
};
...
const setupRenderEffect = (
  instance: ComponentInstance,
  vnode: VNode,
  container: Container,
  anchor: Node | null,
  internals: ComponentRendererInternals
) => {
  const componentUpdateFn = () => {
    if (!instance.isMounted) {
      invokeLifecycleHooks(instance.bm);
      const subTree = instance.render?.call(instance.proxy, instance.proxy); // 就是这里
      instance.subTree = subTree;
      internals.patch(null, subTree, container, anchor, instance); // render
      vnode.el = subTree.el;
      instance.isMounted = true;
      invokeLifecycleHooks(instance.m);
      // instance.type.mounted?.call(instance.proxy, instance.proxy);
    } else {
      invokeLifecycleHooks(instance.bu)
      const prevTree = instance.subTree;
      const nextTree = instance.render?.call(instance.proxy, instance.proxy);
      instance.subTree = nextTree;
      internals.patch(prevTree, nextTree, container, anchor, instance); // update
      invokeLifecycleHooks(instance.u);
    }
  };

  let update: () => void;
  const reactiveEffect = new ReactiveEffect(componentUpdateFn, () => {
    queueJob(update);
  });
  instance.effect = reactiveEffect;
  update = instance.update = () => {
    if (instance.isUnmounted) {
      return;
    }
    reactiveEffect.run();
  };
  update();
};
```

这里就可以参与通信，instance.emit预先用emit.bind(null, instance)的方式，绑定好当前的实例，触发事件去instance.vnode.props里找好对应的onXXX回调。

```ts
export type Emit = (event: string, ...args: unknown[]) => void;
...

if (key === '$emit') {
return emit;
}
```

还参与了记录父组件实例，并通过Object.create(parent.provides)的形式跨层级provide/inject的原型链查找。

instance.subTree保存了当前组件render执行后生成的内部虚拟DOM树，instance.effect和instance.update保存了组件的ReactiveEffect实例和触发重绘的update函数，当自身响应式状态改变的时候，或者父组件的updateComponent中更新了Props的时候，都会调用instance.update()，拿旧的instance.subTree和新的nextTree进行patch对比。

```ts
  const componentUpdateFn = () => {
    if (!instance.isMounted) {
      invokeLifecycleHooks(instance.bm);
      const subTree = instance.render?.call(instance.proxy, instance.proxy);
      instance.subTree = subTree;
      internals.patch(null, subTree, container, anchor, instance); // render
      vnode.el = subTree.el;
      instance.isMounted = true;
      invokeLifecycleHooks(instance.m);
      // instance.type.mounted?.call(instance.proxy, instance.proxy);
    } else {
      invokeLifecycleHooks(instance.bu)
      const prevTree = instance.subTree;
      const nextTree = instance.render?.call(instance.proxy, instance.proxy);
      instance.subTree = nextTree;
      internals.patch(prevTree, nextTree, container, anchor, instance); // update
      invokeLifecycleHooks(instance.u);
    }
  };
```

## render的时期

通过 instance.render?.call(instance.proxy, instance.proxy) 调用，render方法内的this会指向instance.proxy，这样在render函数内部我们访问诸如count此类的值可以用proxy.count，或者this.count，都会经过Proxy的get拦截器，按照setupState->data->props-> `$emit $attrs $slots`的顺序查找数据并完成依赖收集。

因为 instance.render 是在 reactiveEffect.run()内部运行的，所以每次运行前都会先执行 cleanupEffect(this)。清空上一次渲染收集的依赖，再在本次 render()，执行过程中重新收集（这样如果某次渲染因为 v-if /三元表达式不再读取某个变量，该变量变化时就不会再触发重绘）。

```ts
const subTree = instance.render?.call(instance.proxy, instance.proxy);
```

通过render会产出subTree，render内部通过调用h方法返回一颗代表组件内部的结构的VNode树，称作subTree。交由internals.patch处理为真实DOM，首次挂载的时候，调用 internals.patch(null, subTree, container, anchor,instance)。因为第一个参数 n1 是 null，patch 会把 subTree 递归创建为真实 DOM节点插入 container，最后把生成出来的根 DOM 元素赋给组件外壳 vnode.el =subTree.el（方便后续移动或卸载组件时拿到真实 DOM）。

依旧看这里，这里调用了注入的方法里的patch方法。
```ts
  const componentUpdateFn = () => {
    if (!instance.isMounted) {
      invokeLifecycleHooks(instance.bm);
      const subTree = instance.render?.call(instance.proxy, instance.proxy);
      instance.subTree = subTree;
      internals.patch(null, subTree, container, anchor, instance); // render
      vnode.el = subTree.el;
      instance.isMounted = true;
      invokeLifecycleHooks(instance.m);
      // instance.type.mounted?.call(instance.proxy, instance.proxy);
    } else {
      invokeLifecycleHooks(instance.bu)
      const prevTree = instance.subTree;
      const nextTree = instance.render?.call(instance.proxy, instance.proxy);
      instance.subTree = nextTree;
      internals.patch(prevTree, nextTree, container, anchor, instance); // update
      invokeLifecycleHooks(instance.u);
    }
  };
  
export type PatchFunction = (
  n1: VNode | null,
  n2: VNode,
  container: Container,
  anchor?: Node | null,
  parentComponent?: ComponentInstance | null
) => void;

  const patch = (
    n1: VNode | null,
    n2: VNode,
    container: Container,
    anchor: Node | null = null,
    parentComponent: ComponentInstance | null = null
  ) => {
    if (n1 === n2) {
      return;
    }
    if (n1 !== null) {
      if (!isSameVNode(n1, n2)) {
        // // console.log('[patch<VNode>]', n1, n2, 'unmount');
        unmount(n1);
        n1 = null; // 原节点不复用
      }
    }
    const { type, shapeFlag } = n2;
    switch (type) {
      case Text:
        processText(n1, n2, container, anchor);
        break;
      case Fragment:
        processFragment(n1, n2, container, anchor, parentComponent);
        break;
      default:
        if (shapeFlag & ShapeFlags.ELEMENT) {
          processElemet(n1, n2, container, anchor, parentComponent);
        } else if (shapeFlag & ShapeFlags.COMPONENT) {
          // // console.log('[patch]: component');
          processComponent(n1, n2, container, anchor, parentComponent, internals);
        } else if (shapeFlag & ShapeFlags.TELEPORT) {
          type.process(n1, n2, container, anchor, {
            mountChildren,
            patchChildren,
            unmountChildren,
            createText: hostCreateText,
            insert: hostInsert,
            querySelector: hostQuerySelector,
          });
        }
        break;
    }
  };
```

顶层暴露的render并不是这里的render，如果你看了example里的html，会发现最下面有渲染根组件的render调用：

```ts
render(
    h({
        type: Parent
    }),
    app
)
```

这个render函数是真个应用的挂载/卸载入口，当传入的render(vnode， container)时候，他会把上一次挂载在容器上的`container，_vnode`交给patch方法，并将本次vnode缓存到`container._vodce` 上，当传入的render(null, container)时，如果 `container._vnode` 存在，则会调用 `unmount(container._vnode)` 卸载整棵应用树。

```ts
  const render = (vnode: VNode | null, container: Container): void => {
    if (vnode === null) {
      if (container._vnode) {
        unmount(container._vnode);
      }
      container._vnode = null;
      return;
    }
    patch(container._vnode || null, vnode, container);
    container._vnode = vnode;
  };
```

## 依赖追踪的周期

我们一般有两种存储依赖的集合，Dep `dep: Set<ReactiveEffect>`。reactive对象使用的是全局的targetMap三层结构来存储的，WeakMap以key 和 Dep 为kv存放所有依赖该属性的ReactiveEffect。

```ts
const targetMap: WeakMap<object, Map<string | symbol, Set<ReactiveEffect>>> = new WeakMap();
export type Dep = Set<ReactiveEffect>;
```

ref/computed实例因为自身挂一个 `this.dep = new Set<ReactiveEffect>()` ，所以不需要使用WeakMap。

同时，每个ReactiveEffect实例上也有一个数组叫做`public deps: Dep[] = []`。这个用来反方向记录自己被哪些属性的Set各地收集过，我们知道ReactiveEffect用来执行响应式副作用的，然后Dep是一系列的响应式副作用，不会重复执行。deps显而易见，是用来反向记录“我被哪些属性的 Set 收集过”。

```ts
export let activeEffect: ReactiveEffect | undefined = undefined;
export class ReactiveEffect<T = any> {
  public executeCount = 0;
  public active = true;
  public deps: Dep[] = []; // deps from this ReactiveEffect used
  public running = 0;
  constructor(
    public fn: () => T,
    public scheduler?: () => void
  ) {}
  run(): T {
    this.executeCount++;
    if (!this.active) return this.fn() as T;
    cleanupEffect(this);
    let lastActiveEffect = activeEffect;
    try {
      this.running++;
      activeEffect = this;
      return this.fn() as T;
    } finally {
      this.running--;
      activeEffect = lastActiveEffect;
    }
  }
  stop() {
    if (this.active) {
      cleanupEffect(this);
      this.active = false;
    }
  }
}

```

追踪的全流程以effect.run到trackEffects为例。当执行run的时候会用局部变量lastActiveEffect 保存外层的activeEffect，然后把全局 activeEffect 设为当前 this，再执行 this.fn()。执行完后在 finally 块里恢复 activeEffect = lastActiveEffect（index.ts:32-40）

为什么要用 lastActiveEffect？ 为了支持父子组件嵌套渲染或 computed 嵌套：父组件render 一半去挂载子组件时，activeEffect切为子组件；子组件挂载完恢复为父组件，不会弄丢父组件的上下文。

拦截读取用track，在this.fn执行期间，读取reactive对象的属性会触发Proxy的get 拦截器，调用track(target, key)，如果当前没有activeEffect，说明只是普通代码读取，不在effect/render/computed/watch内部，直接return。

否则从 targetMap 中取出（或新建）该 `target[key]` 对应的 dep 集合，交给trackEffects(dep)。

```ts
const mutableHandlers: ProxyHandler<any> = {
  get(target, key, receiver) {
    if (key === ReactiveFlags.IS_REACTIVE) return true;
    if (key === ReactiveFlags.IS_READONLY) return false;
    if (key === ReactiveFlags.RAW) return target;

    // effect track
    track(target, key);
    let result = Reflect.get(target, key, receiver);
    if (isObject(result)) {
      result = createReactiveObject(result);
    }
    return result;
  },
  set(target, key, value, receiver) {
    let oldValue = Reflect.get(target, key, receiver);
    // 相同值不需要触发更新，但赋值本身仍然是成功的。
    if (oldValue === value) return true;
    // update
    let result = Reflect.set(target, key, value, receiver);
    if (result) {
      trigger(target, key);
    }
    return result;
  },
};

export function track(target: object, key: string | symbol) {
  if (!activeEffect) return;
  let depsMap = targetMap.get(target);
  if (!depsMap) {
    targetMap.set(target, (depsMap = new Map()));
  }
  let dep = depsMap.get(key);
  if (!dep) {
    depsMap.set(key, (dep = new Set()));
  }
  trackEffects(dep);
}
export function trackEffects(dep: Set<ReactiveEffect>) {
  if (dep.has(activeEffect!)) {
    return;
  }
  dep.add(activeEffect!);
  activeEffect!.deps.push(dep);
}
function createReactiveObject(target: object) {
  if (!isObject(target)) return target;
  if (reactiveMap.has(target)) return reactiveMap.get(target);
  let proxy = new Proxy(target, mutableHandlers);
  reactiveMap.set(target, proxy);
  return proxy;
}
```

依赖收集的过程中标，如果现在执行的副作用存在于obj的某个属性的dep里，就不管，如果不存在就收集进去，执行依靠setter钩子里的trigger触发器，当obj的depsMap里有某个属性的deps时，就拷贝一份执行，逻辑清晰明了。

```ts
export function trigger(target: object, key: string | symbol) {
  const depsMap = targetMap.get(target);
  if (!depsMap) return;
  const dep = depsMap.get(key);
  console.log('[trigger]', key, dep);
  if (!dep) return;
  // const effects = new Set(dep);
  // console.log('[execute]', effects);
  triggerEffects(dep);
}
/**
 * 触发依赖
 * @param dep 依赖集合
 */
export function triggerEffects(dep: Set<ReactiveEffect>) {
  const effects = new Set(dep);
  effects.forEach((effect) => {
    if (effect !== activeEffect) {
      if (effect.scheduler) {
        {
          if (!effect.running) {
            // prevent infinite loop
            effect.scheduler();
          }
        }
      } else {
        effect.run();
      }
    }
  });
}
```

清理方法是：

```ts
export function cleanupEffect(effect: ReactiveEffect) {
  const deps = effect.deps;
  deps.forEach((dep) => dep.delete(effect));
  deps.length = 0;
}
```

在trigger内执行effect.run之后，在正式执行委托的this.fn之前会做一次清理，cleanupEffect，然后running计数器+1， 将正在激活的effect设置为自己，最后出去的时候计数器-1，将之前我们设置的activceEffect又设置回去，执行了之后操作obj的相关属性又会添加到deps里去， `activeEffect!.deps.push(dep);` 。

计数器是为了递归保护，如果有人分在effect里或schedular内部写了自增操作，比如Obj.count++，那么就会读get 和写set，这样触发triggerEffects的时候外层的if (effect !== activeEffect) 和内部的调度器的if (!effect.running) 就会拦截这次触发，防止run没跑完，内部又在无限递归调用导致栈溢出。

