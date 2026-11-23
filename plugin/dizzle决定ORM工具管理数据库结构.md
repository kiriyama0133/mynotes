---
date: 2026-11-23
tags:
  - plugin
  - dizzle
---
drizzle指的是drizzle orm，是一个typecript orm工具，用于TypeScript/JavaScript 代码中操作关系型数据库，例如 SQLite、PostgreSQL 和 MySQL。

他不是数据库，而是一个orm工具，让我们避免编写sql代码实现schema的定义和存取查询操作等。

## drizzle介绍
drizzle工具其实包含了配套了orm和配套cli，而不是一个单一的工具：


|工具|职责|
|---|---|
|`drizzle-orm`|实际执行数据库查询|
|`drizzle-kit`|数据库结构管理、迁移生成和执行|
|`drizzle.config.ts`|配置 Drizzle Kit|
|`schema.ts`|用 TypeScript 定义数据库表|
|`drizzle/`|通常存放生成的 SQL 迁移文件|

![](https://pub-dd8e151a60af44cfbe36beaaf0ad54e8.r2.dev/2026/11/1795414318331-ib9djrzh.webp)

使用了drizzle的项目里，有一份叫做drizzle.config.ts的文件，里面定义了使用什么类型的数据库，数据库表结构定义在哪里、生成的迁移文件放在哪里、如何连接数据库，这个需要连接信息的命令。

如果我们使用脚手架工具`npx degit nick-keller/electron-appstack/template <name>`，那么这个模板里就有一个配置文件叫做drizzle.config.ts。


```ts
import { defineConfig } from 'drizzle-kit'

// Only used by `npm run db:generate` to create SQL migrations in `drizzle/`.
// Migrations are applied automatically when the app starts (see src/main/db/client.ts).
export default defineConfig({
  dialect: 'sqlite',
  schema: './src/main/db/schema.ts',
  out: './drizzle'
})

```

Electron AppStack里采用的就是默认的sqlite+drizzle orm，并且它把数据库迁移集成到了应用启动流程中。


```ts
import { sql } from 'drizzle-orm'
import { integer, sqliteTable, text } from 'drizzle-orm/sqlite-core'

// After changing this file, run `npm run db:generate` to create a migration.
// Migrations are applied automatically when the app starts.

export const todos = sqliteTable('todos', {
  id: integer().primaryKey({ autoIncrement: true }),
  title: text().notNull(),
  done: integer({ mode: 'boolean' }).notNull().default(false),
  createdAt: integer({ mode: 'timestamp_ms' }).notNull().default(sql`(unixepoch('subsec') * 1000)`)
})

```

这里是schema部分，定义了一个表里有什么字段。并且我们一般采用zod作为验证器，它和typescript的类型系统区别是ts在编译阶段检查类型，zod是运行时检查。

## zod

我们截取一段启动检查默认配置，从磁盘加载配置检查数据是否合法，修复。写入配置时再次验证并阻止非法数据进入存的的逻辑：

```ts
import Store from 'electron-store'
import type { z } from 'zod'

type Listener<T> = (value: T, previous: T) => void

/**
 * A zod-validated wrapper around electron-store.
 *
 * - Every key of the schema must have a `.default()`.
 * - Values are validated on write; invalid stored values (e.g. after a schema change) are reset
 *   to their default on load instead of crashing the app.
 * - Values are cached in memory, reads never hit the disk.
 */
export function createTypedStore<Schema extends z.ZodObject>(
  schema: Schema,
  options: { name?: string } = {}
) {
  type Data = z.infer<Schema>
  type Key = keyof Data & string

  const shape = schema.shape as Record<Key, z.ZodType>
  const store = new Store<Record<string, unknown>>({ name: options.name ?? 'config' })
  const listeners = new Map<Key, Set<Listener<never>>>()
  const anyListeners = new Set<(data: Data) => void>()

  // Throws at startup if a key has no default
  const defaults = schema.parse({}) as Data

  function load(): Data {
    const raw = store.store
    const result = schema.safeParse(raw)
    if (result.success) return result.data as Data

    // Keep valid keys, reset invalid ones to their default
    const repaired = { ...defaults }
    for (const key of Object.keys(shape) as Key[]) {
      const value = shape[key].safeParse(raw[key])
      if (value.success) repaired[key] = value.data as Data[Key]
    }
    store.store = repaired
    return repaired
  }

  let data = load()

  return {
    get<K extends Key>(key: K): Data[K] {
      return data[key]
    },

    getAll(): Data {
      return data
    },

    set<K extends Key>(key: K, value: Data[K]): void {
      const previous = data[key]
      const parsed = shape[key].parse(value) as Data[K]
      data = { ...data, [key]: parsed }
      store.set(key, parsed)
      for (const listener of listeners.get(key) ?? []) {
        ;(listener as Listener<Data[K]>)(parsed, previous)
      }
      for (const listener of anyListeners) listener(data)
    },

    reset<K extends Key>(key: K): void {
      this.set(key, defaults[key])
    },

    /** Calls `listener` every time `key` is set. Returns an unsubscribe function. */
    onChange<K extends Key>(key: K, listener: Listener<Data[K]>): () => void {
      const set = listeners.get(key) ?? new Set()
      set.add(listener as Listener<never>)
      listeners.set(key, set)
      return () => set.delete(listener as Listener<never>)
    },

    /** Calls `listener` with the whole config every time any key is set. */
    onAnyChange(listener: (data: Data) => void): () => void {
      anyListeners.add(listener)
      return () => anyListeners.delete(listener)
    },

    /** Absolute path of the JSON file. */
    path: store.path
  }
}

```

zod要参与到验证里，就需要有一个zod schema，假如项目里有一个下方这种的Schema：

```ts
import { z } from 'zod'

export const configSchema = z.object({
  robotName: z.string().default('Robot-01'),

  maxVelocity: z.number().min(0).max(2).default(1),

  enableCollisionCheck: z.boolean().default(true)
})
```

z.object定义了对象结构，zod不仅能定义类型还能定义约束，比如.numer().min(0).max(2)可以约束到必须是数字，不能小于0，不能大于2。


```ts
configSchema.parse({
  robotName: 'Robot-01',
  maxVelocity: 3,
  enableCollisionCheck: true
})
```

因此这种解析就会抛出maxVelocity的错误。

```ts
configSchema.safeParse({
  robotName: 'Robot-01',
  maxVelocity: 1,
  enableCollisionCheck: true
})
```
这种会返回一个结果对象：

```ts
{
  success: true,
  data: {
    robotName: 'Robot-01',
    maxVelocity: 1,
    enableCollisionCheck: true
  }
}
```

这里有两个核心 API：

|API|行为|
|---|---|
|`parse()`|验证成功返回数据，失败抛出异常|
|`safeParse()`|验证成功或失败都返回结果对象，不直接抛异常|

zod本身也包含了一些类型推导工具比如说从schema推导typecript类型的工具，infer，使用方法如上：

```ts
  type Data = z.infer<Schema>;
  type Key = keyof Data & string;
```

一个schema是：

```ts
const configSchema = z.object({
  robotName: z.string().default('Robot-01'),
  maxVelocity: z.number().default(1)
})
```
那么Data就会处理得到

```ts
type Data = {
  robotName: string
  maxVelocity: number
}
```

schema.shape用于获取每个字段单独的验证器，看上面的代码：

```ts
  const shape = schema.shape as Record<Key, z.ZodType>
  const store = new Store<Record<string, unknown>>({ name: options.name ?? 'config' })
  const listeners = new Map<Key, Set<Listener<never>>>()
  const anyListeners = new Set<(data: Data) => void>()
```

schema.shape这一行我们看一下，这个是整个实现里的一行，假设schema是：

```ts
const schema = z.object({
  robotName: z.string().default('Robot-01'),
  maxVelocity: z.number().min(0).max(2).default(1)
})
```

那么这里的schema.shape可以理解为：

```ts
{
  robotName: z.string().default('Robot-01'),
  maxVelocity: z.number().min(0).max(2).default(1)
}
```

也就是说，`shape` 保存了每个字段独立的 Zod 验证器。

这样甚至就可以利用shape来进行单独验证某个字段了，这个比较方便，对于后面的set和Load操作来说很重要。

```ts
shape.robotName.parse('Robot-02')
```

## 配套使用

drizzle和zod配合使用的核心思想是drizzle负责定义数据库结构，zod负责验证进入系统的数据，而drizzle-zod可以根据drizzle的表定义自动生成zod schema。

我们在src/main/db/schema.ts里定义数据库表的时候，使用drizzle-zod自动生成验证shcema，对于一个drizzle数据库表：

```ts
import {
  sqliteTable,
  text,
  real,
  integer
} from 'drizzle-orm/sqlite-core'

export const robotConfigs = sqliteTable('robot_configs', {
  id: integer('id').primaryKey({ autoIncrement: true }),

  name: text('name').notNull(),

  maxVelocity: real('max_velocity').notNull(),

  enableCollisionCheck: integer('enable_collision_check', {
    mode: 'boolean'
  }).notNull()
})
```

我们配置zod验证器：

```ts
import { createInsertSchema } from 'drizzle-zod'
import { z } from 'zod'

import { robotConfigs } from './schema'

export const insertRobotConfigSchema =
  createInsertSchema(robotConfigs, {
    name: (schema) => schema.min(1).max(50),

    maxVelocity: (schema) => schema.min(0.01).max(2)
  }).omit({
    id: true
  })
```

`createInsertSchema(robotConfigs)` 会根据 Drizzle 的表结构生成 Zod Schema。

例如：

- `name` 自动生成字符串验证。
    
- `maxVelocity` 自动生成数字验证。
    
- `enableCollisionCheck` 自动生成布尔值验证。
    
- 自增主键 `id` 不需要由调用者提供。


