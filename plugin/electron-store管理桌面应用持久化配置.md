---
date: 2026-11-23
tags:
  - plugin
  - electron
---
在Electron AppStack中，使用了electron-store作为管理桌面应用的持久化配置，它和Pinia不一样，两者工作在不同的层面。

pinia是vue御用的状态管理库，我们这个模板是react作为渲染层，因此我们会使用zustand作为状态管理库，但是说了这些其实都是和今天的主菜没有关系。

electron-store的数据存储使用的是本地的json就可以看出来和zustand or pinia完全不一样，后面两个工作在内存里，连状态都是保存在内存里的。electron-store也支持配置变更监听，ui响应的话，需要额外的通信，所以说了这么多，你应该看出来了，一个在renderer进程，一个在main进程。

在实践项目组织里，electron-store推荐这样组织文件：

```
Electron Main
│
├── config/
│   ├── config.schema.ts
│   ├── config.store.ts
│   └── config.service.ts
│
├── database/
│   ├── schema.ts
│   ├── client.ts
│   └── repositories/
│
└── trpc/
    └── routers/
        └── config.router.ts

React Renderer
│
├── stores/
│   ├── ui.store.ts
│   ├── viewport.store.ts
│   └── task.store.ts
│
└── features/
```

外观的主题、语言、字体大小等配置；窗口的大小、位置、是否最大化；通信配置；用户偏好；等等都可以存放在electron-store里，而且这个也是推荐的。

## 实践

比如来浅显地实现一个页面可以修改主题、python后端地址和端口，修改之后立即生效，并且重启之后应用任然可以保留的功能。

这次先不引入 SQLite，也不引入 Zustand，重点理解 `electron-store` 的完整使用链路

```
React 设置页面
    ↓
tRPC mutation
    ↓
Zod 输入验证
    ↓
Electron Main
    ↓
electron-store
    ↓
本地 config.json
```

我们先去建立一个schema，在config里面建立一个专门存放配置的目录，然后定义文件`config.schema.ts`：

```ts
import { z } from 'zod'

// 定义应用配置的数据结构
export const configSchema = z.object({
  // 应用主题
  theme: z.enum(['light', 'dark', 'system']).default('dark'),

  // Python 后端连接配置
  backend: z.object({
    host: z.string().default('127.0.0.1'),
    port: z.number().int().min(1).max(65535).default(50051),
  }).default({}),

})

// 从 Zod Schema 推导 TypeScript 类型
export type AppConfig = z.infer<typeof configSchema>

```

schema既是描述配置结构，也是负责运行时的验证。然后我们创建electron-store实例，在config里创建文件`config.store.ts`：

```ts
import { createTypedStore } from '../utils/typed-store'
import { configSchema } from './config.schema'

// 创建应用配置存储实例
export const configStore = createTypedStore(
  configSchema,
  {
    name: 'app-config',
  }
)

```
这里假设贴出的 `createTypedStore` 封装保存在 `src/main/utils/typed-store.ts`，如果实际路径不同，调整 import 即可。这段代码执行后，electron-store 会管理一个名为 `app-config` 的 JSON 配置文件。


```ts
import Store from 'electron-store'
import type { z } from 'zod'

// 配置变化监听器
type Listener<T> = (
  value: T,
  previous: T
) => void

// createTypedStore 的参数类型
type TypedStoreOptions = {
  name?: string
}

// 创建带类型和 Zod 校验的 electron-store
export function createTypedStore<
  Schema extends z.ZodObject
>(
  schema: Schema,
  options: TypedStoreOptions = {}
) {
  // 从 Zod Schema 推导配置类型
  type Data = z.infer<Schema>

  // 配置键的类型
  type Key = keyof Data & string

  // 获取 Schema 中每个字段对应的验证器
  const shape = schema.shape as Record<
    Key,
    z.ZodType
  >

  // 创建 electron-store 实例
  const store = new Store<Record<string, unknown>>({
    name: options.name ?? 'config',
  })

  // 每个配置项对应一组监听器
  const listeners = new Map<
    Key,
    Set<Listener<unknown>>
  >()

  // 监听所有配置变化
  const anyListeners = new Set<
    (data: Data) => void
  >()

  // 解析默认配置
  const defaults = schema.parse({}) as Data

  // 加载和修复本地配置
  function load(): Data {
    const raw = store.store

    // 尝试验证整个配置对象
    const result = schema.safeParse(raw)

    if (result.success) {
      return result.data as Data
    }

    // 整体验证失败，从默认配置开始修复
    const repaired = { ...defaults }

    // 逐个字段验证
    for (const key of Object.keys(shape) as Key[]) {
      const value = shape[key].safeParse(raw[key])

      if (value.success) {
        repaired[key] = value.data as Data[Key]
      }
    }

    // 将修复后的配置重新保存
    store.store = repaired

    return repaired
  }

  // 应用启动时加载配置
  let data = load()

  return {
    // 读取单个配置
    get<K extends Key>(key: K): Data[K] {
      return data[key]
    },

    // 读取全部配置
    getAll(): Data {
      return data
    },

    // 修改单个配置
    set<K extends Key>(
      key: K,
      value: Data[K]
    ): void {
      const previous = data[key]

      // 运行时验证
      const parsed = shape[key].parse(value) as Data[K]

      // 更新内存中的配置
      const next = {
        ...data,
        [key]: parsed,
      }

      // 先持久化，成功后再更新内存
      store.set(key, parsed)
      data = next

      // 通知当前字段的监听器
      for (const listener of listeners.get(key) ?? []) {
        (listener as Listener<Data[K]>)(parsed, previous)
      }

      // 通知全局监听器
      for (const listener of anyListeners) {
        listener(data)
      }
    },

    // 恢复单个配置的默认值
    reset<K extends Key>(key: K): void {
      this.set(key, defaults[key])
    },

    // 监听单个配置变化
    onChange<K extends Key>(
      key: K,
      listener: Listener<Data[K]>
    ): () => void {
      let set = listeners.get(key)

      if (!set) {
        set = new Set()
        listeners.set(key, set)
      }

      set.add(listener as Listener<unknown>)

      // 返回取消监听函数
      return () => {
        set?.delete(listener as Listener<unknown>)

        if (set?.size === 0) {
          listeners.delete(key)
        }
      }
    },

    // 监听任意配置变化
    onAnyChange(
      listener: (data: Data) => void
    ): () => void {
      anyListeners.add(listener)

      return () => {
        anyListeners.delete(listener)
      }
    },

    // electron-store 配置文件路径
    path: store.path,
  }
}

```

在windows上的话，生产的json文件位置在：%APPDATA%/你的应用名称/app-config.json。配置文件的文件内容大致如下：

```ts
{
  "theme": "dark",
  "backend": {
    "host": "127.0.0.1",
    "port": 50051
  }
}
```

注意：不需要自己手动创建这个 JSON 文件，electron-store 会负责持久化。

然后通过tRPC来提供配置读写接口，Renderer 不应该直接操作 electron-store。我们让 Electron Main 暴露几个经过 Zod 验证的接口。

tRPC封装了Electron IPC的通信，因此我们不需要在业务代理里手动编写ipcMain.handle和ipcRenderer.invoke等操作。在加入了trpc之后，main process只需要定义Router文件即可，它看起来像普通的 React Hook，但实际上会通过 tRPC Client 将调用发送到 Electron Main Process。

并且有意思的是，trpc本身用于处理ts应用内部的RPC，通信协议取决于Adapter，比如AppStack模板里通过Electron IPC，接口定义实现是ts router，虽然trpc并不是天然等于ipc，因为它也可以运行在http、ws等传输层之上，我们这里使用了electron ipc。和grpc完全不一样，grpc后面可以自己了解，两者可以说完全不搭边，除了名字都带有RPC，也就是远程过程调用以外。

>RPC是远程过程调用，允许我们像调用本地函数一样调用另一个进程或者另一台机器上的函数，这里可以跨网络，也可以同一台电脑上的不同进程

但有些问题函待开发者自己去解决，比如说，trpc和elctron ipc本身不提供通用防抖机制，如果用户每触发一次Mutation，通常就产生一次ipc请求，这里推荐在renderer层实现一次防抖的机制。

编写`settings.router.ts`文件：

```ts
import { z } from 'zod'
import { router, publicProcedure } from '../trpc'
import { configStore } from '../../config/config.store'

export const settingsRouter = router({
  // 获取全部配置
  get: publicProcedure.query(() => {
    return configStore.getAll()
  }),

  // 修改主题
  updateTheme: publicProcedure
    .input(
      z.object({
        theme: z.enum(['light', 'dark', 'system']),
      })
    )
    .mutation(({ input }) => {
      configStore.set('theme', input.theme)

      return {
        success: true,
        theme: configStore.get('theme'),
      }
    }),

  // 修改 Python 后端连接配置
  updateBackend: publicProcedure
    .input(
      z.object({
        host: z.string().min(1),
        port: z.number().int().min(1).max(65535),
      })
    )
    .mutation(({ input }) => {
      configStore.set('backend', input)

      return {
        success: true,
        backend: configStore.get('backend'),
      }
    }),
})
```

这里有三个接口：

|接口|类型|作用|
|---|---|---|
|`settings.get`|Query|获取全部配置|
|`settings.updateTheme`|Mutation|修改主题|
|`settings.updateBackend`|Mutation|修改后端连接参数|
Query来读取数据，Mutation来修改数据，.input(z.object(...))用于验证从Renderer传来的数据，`configStore.set()` 内部还会再次进行字段验证，因此不是只依赖 tRPC 的输入校验。

最后再项目原有总路由文件里将`settingsRouter` 注册进去。

```ts
import { router } from './trpc'
import { settingsRouter } from './routers/settings.router'

export const appRouter = router({
  settings: settingsRouter,
})

export type AppRouter = typeof appRouter
```

这样React就可以设置页面调用接口了：

```tsx
import { useState } from 'react'
import { trpc } from '../utils/trpc'

export function SettingsPage() {
  // 获取当前配置
  const { data: config, isLoading } =
    trpc.settings.get.useQuery()

  // 获取 tRPC 缓存工具
  const utils = trpc.useUtils()

  // 修改主题
  const themeMutation =
    trpc.settings.updateTheme.useMutation({
      onSuccess: () => {
        utils.settings.get.invalidate()
      },
    })

  // 修改后端配置
  const backendMutation =
    trpc.settings.updateBackend.useMutation({
      onSuccess: () => {
        utils.settings.get.invalidate()
      },
    })

  const [host, setHost] = useState('')
  const [port, setPort] = useState('50051')

  if (isLoading || !config) {
    return <div>正在加载配置...</div>
  }

  return (
    <div>
      <h2>应用设置</h2>

      {/* 主题设置 */}
      <section>
        <h3>外观</h3>

        <select
          value={config.theme}
          onChange={(event) => {
            themeMutation.mutate({
              theme: event.target.value as
                'light' | 'dark' | 'system',
            })
          }}
        >
          <option value="light">浅色</option>
          <option value="dark">深色</option>
          <option value="system">跟随系统</option>
        </select>
      </section>

      {/* Python 后端设置 */}
      <section>
        <h3>Python 后端</h3>

        <div>
          <label>Host：</label>
          <input
            value={host || config.backend.host}
            onChange={(event) =>
              setHost(event.target.value)
            }
          />
        </div>

        <div>
          <label>Port：</label>
          <input
            value={port}
            onChange={(event) =>
              setPort(event.target.value)
            }
          />
        </div>

        <button
          disabled={backendMutation.isPending}
          onClick={() => {
            backendMutation.mutate({
              host: host || config.backend.host,
              port: Number(port),
            })
          }}
        >
          保存后端配置
        </button>
      </section>
    </div>
  )
}

```

到这里就完成了实现。

## Electron IPC Adapter


这里只放出一个文件的实现，具体实现请看相关项目的Electron IPC Adapter
```ts
// trpc/trpc.ts
import { on } from 'node:events'
import { initTRPC, TRPCError, type TRPCSubscriptionProcedure } from '@trpc/server'
import superjson from 'superjson'
import { config } from '../config'
import { getDb } from '../db/client'
import { type AppEvents, appEvents } from '../events'
import { isLicenseActive } from '../license'

export function createContext() {
  return Promise.resolve({ db: getDb(), config })
}

export type Context = Awaited<ReturnType<typeof createContext>>

const t = initTRPC.context<Context>().create({
  // superjson keeps Dates, Maps, Sets… intact between main and renderer
  transformer: superjson,
  isServer: true
})

export const router = t.router
export const publicProcedure = t.procedure

/**
 * A procedure that only runs while the license is active (or licensing is disabled). Use it for
 * the features of the app: the main window is closed when the license is refused, but this also
 * stops any other window from calling them.
 */
export const licensedProcedure = t.procedure.use(({ next }) => {
  if (!isLicenseActive()) {
    throw new TRPCError({ code: 'FORBIDDEN', message: 'The license is not active' })
  }
  return next()
})

/**
 * Wraps an async generator so it can be returned from a subscription.
 *
 * trpc-electron fails on native async iterators, which have `Symbol.asyncDispose` since
 * Node.js 24 ("Symbol.asyncDispose already exists"). A plain object iterator does not.
 */
export function ipcIterable<T>(source: AsyncIterable<T>): AsyncIterable<T> {
  return {
    [Symbol.asyncIterator]() {
      const iterator = source[Symbol.asyncIterator]()
      return {
        next: () => iterator.next(),
        return: async (value?: unknown) =>
          (await iterator.return?.(value)) ?? { done: true, value: undefined }
      }
    }
  }
}

/** A subscription that forwards every `event` emitted on `appEvents` to the renderer. */
export function appEventSubscription<E extends keyof AppEvents>(
  event: E
): TRPCSubscriptionProcedure<{
  // biome-ignore lint/suspicious/noConfusingVoidType: tRPC uses `void` for procedures without input
  input: void
  output: AsyncIterable<AppEvents[E][0], void, unknown>
  meta: object
}> {
  return publicProcedure.subscription(({ signal }) =>
    ipcIterable(
      (async function* () {
        for await (const [value] of on(appEvents, event, { signal })) {
          yield value as AppEvents[E][0]
        }
      })()
    )
  )
}

```
