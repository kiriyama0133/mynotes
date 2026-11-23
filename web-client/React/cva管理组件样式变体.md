---
date: 2026-11-23
tags:
  - web-client
  - react
  - cva
---
cva的全称叫做**Class Variance Authority**，用于管理的样式变体问题，一个组件有多种外观、尺寸、状态的时候，不必自己手动拼接大量条件判断，而是通过配置对象声明不同变体对应的 Tailwind CSS 类名。

它是一个样式工厂：

![](https://pub-dd8e151a60af44cfbe36beaaf0ad54e8.r2.dev/2026/11/1795419327330-vpwyiu01.webp)

可以先利用它定义基础样式：

```ts
const buttonVariants = cva(
  "inline-flex shrink-0 items-center ...",
  {
    // 配置
  }
)
```

第一个参数是基础 className，无论 Button 是什么类型、什么尺寸，这些样式都会应用。比如说inline-flex布局、intems-center jusitfy-center内容居中、rounded-md圆角、transition-all过渡动画、disabled:opacity-50禁用的时候的透明度降低。

第二部分是变体样式，variants部分：

```ts
variants: {
  variant: {
    default: 'bg-primary text-primary-foreground hover:bg-primary/90',
    destructive: 'bg-destructive text-white hover:bg-destructive/90',
    outline: 'border bg-background shadow-xs',
    secondary: 'bg-secondary text-secondary-foreground',
    ghost: 'hover:bg-accent hover:text-accent-foreground',
    link: 'text-primary underline-offset-4 hover:underline'
  },
  size: {
    default: 'h-9 px-4 py-2',
    xs: 'h-6 gap-1 px-2 text-xs',
    sm: 'h-8 gap-1.5 px-3',
    lg: 'h-10 px-6',
    icon: 'size-9'
  }
```

我们定义两个变体维度，一个是variant，一个size，一个负责样式，一个负责尺寸，职责十分清晰。

```ts
buttonVariants({
  variant: 'destructive',
  size: 'lg'
})
```

这段代码会直接通过cva的组合之后的className返回，大致相当于：

```css
"inline-flex shrink-0 items-center justify-center ... bg-destructive text-white hover:bg-destructive/90 h-10 px-6"
```

**defaultVariants** 第三部分是默认变体，这表示调用时如果没有指定变体，就使用默认值。

注意：`cva()` 返回的是一个函数，而不是一个 className 字符串。

典型用法：

```tsx
const buttonVariants = cva(...)

function Button({
  className,
  variant,
  size,
  ...props
}: React.ComponentProps<'button'> &
  VariantProps<typeof buttonVariants>) {
  return (
    <button
      className={cn(
        buttonVariants({ variant, size }),
        className
      )}
      {...props}
    />
  )
}
```


其中： `VariantProps<typeof buttonVariants>` 是一个很重要的 TypeScript 类型工具,它可以从 `buttonVariants` 的定义中自动提取变体类型, 因此，你不必手动定义：

```ts
type ButtonVariant =
  'default' | 'destructive' | 'outline' |
  'secondary' | 'ghost' | 'link'
```

使用时就有类型提示：

```tsx
<Button variant="destructive" size="lg">
  Delete
</Button>
```
