MediatR 库是建立在 **C# 接口**和 **.NET Core 依赖注入 (DI)** 容器之上的。它利用 DI 容器的生命周期管理和类型解析能力，实现了清晰的请求路由机制，它是一个典型的中介者模式的实现。发送者和接受者都不需要直接引用彼此，消息完全是在进程内传递，无需外部依赖，所有的通信都在进程里完成。并且通过了IMediator接口来统一处理请求/响应和流式请求。

`Mediator` 类作为中介者,负责:

- 接收请求并路由到相应的处理器 Mediator.cs:45-60
- 发布通知到所有订阅的处理器 Mediator.cs:116-125
- 创建和管理流式请求 Mediator.cs:158-175



MediatR 支持以下几种主要的消息处理方式:

### 1. Request/Response (请求/响应)

这是最常见的处理方式,用于发送请求并获取响应，是单播模式，每个请求只会被一个处理器给处理从实现上可以看到,`Send` 方法会解析请求类型并获取**单个**对应的处理器: Mediator.cs:52-59

```csharp
        var handler = (RequestHandlerWrapper<TResponse>)_requestHandlers.GetOrAdd(request.GetType(), static requestType =>
        {
            var wrapperType = typeof(RequestHandlerWrapperImpl<,>).MakeGenericType(requestType, typeof(TResponse));
            var wrapper = Activator.CreateInstance(wrapperType) ?? throw new InvalidOperationException($"Could not create wrapper type for {requestType}");
            return (RequestHandlerBase)wrapper;
        });

        return handler.Handle(request, _serviceProvider, cancellationToken);
```

。 README.md:13

关键在于 `GetRequiredService<IRequestHandler<TRequest, TResponse>>()` **只会返回一个处理器实例**: RequestHandlerWrapper.cs:37-38

```csharp
 Task<TResponse> Handler(CancellationToken t = default) => serviceProvider.GetRequiredService<IRequestHandler<TRequest, TResponse>>()
            .Handle((TRequest) request, t == default ? cancellationToken : t);
```

- **带返回值的请求**: 实现 `IRequestHandler<TRequest, TResponse>` 接口 README.md:63
- **无返回值的请求**: 实现 `IRequestHandler<TRequest>` 接口 README.md:64

通过 `IMediator.Send()` 方法发送请求: Mediator.cs:45-60

### 2. Notifications (通知)

用于发布事件通知,支持多个处理器同时处理同一个通知，这个是消息总线里最常用的方式，也就是订阅者发布者模式，是一种多播模式。 README.md:13

- 实现 `INotificationHandler<TNotification>` 接口 README.md:65
- 通过 `IMediator.Publish()` 方法发布通知 Mediator.cs:116-120

### 3. Stream Requests (流式请求)

用于处理返回异步流的请求,**支持流式数据处理**，它的使用场景是需要处理大量的数据的时候，流式请求可以逐条返回结果而不是一次项加载所有数据到内存，适合用于日志、监控数据等情况。

```csharp
   public class PingStreamHandler : IStreamRequestHandler<Ping, Pong>
    {
        public async IAsyncEnumerable<Pong> Handle(Ping request, [EnumeratorCancellation]CancellationToken cancellationToken)
        {
            yield return await Task.Run(() => new Pong { Message = request.Message + " Pang" });
        }
    }
```

。 README.md:35

- 实现 `IStreamRequestHandler<TRequest, TResponse>` 接口 README.md:66
- 通过 `IMediator.CreateStream()` 方法创建流 CreateStreamTests.cs:51

### 4. Pipeline Behaviors (管道行为)

用于**实现横切关注点,如日志记录、验证、事务处理等**，它允许您在**请求处理前后插入额外的逻辑**。

```csharp
    public class OuterBehavior : IPipelineBehavior<Ping, Pong>
    {
        private readonly Logger _output;

        public OuterBehavior(Logger output)
        {
            _output = output;
        }

        public async Task<Pong> Handle(Ping request, RequestHandlerDelegate<Pong> next, CancellationToken cancellationToken)
        {
            _output.Messages.Add("Outer before");
            var response = await next();
            _output.Messages.Add("Outer after");

            return response;
        }
    }
```

 README.md:76-87

- **Pipeline Behavior**: 实现 `IPipelineBehavior<TRequest, TResponse>` 接口 Program.cs:33-37
- **Stream Pipeline Behavior**: 实现 `IStreamPipelineBehavior<TRequest, TResponse>` 接口 README.md:82

流式管道行为使用 `IAsyncEnumerable` 来处理**流数据**,可以在**流的开始和结束时添加额外的项目**，类似于一个洋葱模型。

#### 具体类型注册

```
cfg.AddBehavior<IPipelineBehavior<Ping, Pong>, OuterBehavior>();
```

MediatrServiceConfiguration.cs:151-163

#### 开放泛型注册

```
cfg.AddOpenBehavior(typeof(GenericBehavior<,>));
```

MediatrServiceConfiguration.cs:202-229

开放泛型行为会应用到所有匹配的请求类型。

#### 常见使用场景

1. **日志记录**: 记录**请求和响应**
2. **验证**: 在处理前**验证请求数据**
3. **事务管理**: 包装处理器在事务中
4. **性能监控**: 测量**处理时间**
5. **缓存**: 缓存响应结果
6. **异常处理**: 统一处理异常

#### Pre/Post Processors (前置/后置处理器)

用于在请求处理前后执行额外逻辑。 README.md:83-84

- **Pre-Processor**: 实现 `IRequestPreProcessor<TRequest>` 接口 Program.cs:38
- **Post-Processor**: 实现 `IRequestPostProcessor<TRequest, TResponse>` 接口 Program.cs:39-40

#### 内置管道行为

MediatR 提供了几个内置的管道行为:

- `RequestPreProcessorBehavior<TRequest, TResponse>`: 执行前置处理器 Program.cs:58
- `RequestPostProcessorBehavior<TRequest, TResponse>`: 执行后置处理器 RequestPostProcessorBehavior.cs:7-31
- `RequestExceptionProcessorBehavior<TRequest, TResponse>`: 处理异常 Program.cs:61
- `RequestExceptionActionProcessorBehavior<TRequest, TResponse>`: 执行异常动作 Program.cs:59-60

#### 约束泛型行为

您可以创建带有类型约束的泛型行为,使其只应用于特定类型: PipelineTests.cs:198-217

这个 `ConstrainedBehavior` 只会应用于 `TRequest` 是 `Ping` 且 `TResponse` 是 `Pong` 的请求。 PipelineTests.cs:570-635

#### Exception Handling (异常处理)

提供结构化的异常处理机制。

- **Exception Handler**: 实现 `IRequestExceptionHandler<TRequest, TResponse, TException>` 接口,可以捕获异常并提供替代响应 README.md:67
- **Exception Action**: 实现 `IRequestExceptionAction<TRequest, TException>` 接口,用于执行副作用(如日志记录),但不能阻止异常传播 README.md:68

管道行为的执行顺序很重要,先注册的行为会在外层,后注册的在内层。所有管道行为都是通过依赖注入容器自动解析的,可以在构造函数中注入其他服务。