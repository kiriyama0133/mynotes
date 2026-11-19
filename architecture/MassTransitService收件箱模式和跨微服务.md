---
date: 2026-08-06
tags:
  - architecture
  - masstransit
  - aspnet
  - outbox
---
在真实企业项目中，通常无需手动手写发件箱轮询，我们会使用MassTransit Outbox，用于解决双写不一致问题。

我们可以打开CleanArchitecture模板的应用层的UseCase业务编排，看看Deposit.cs ：

```cs
public sealed class Deposit(
                        IEntityFactory entityFactory,
                        IOutputPort<DepositOutput> outputHandler,
                        IAccountRepository accountRepository,
                        IUnitOfWork unitOfWork,
                        IServiceBusClient serviceBus) : IUseCase<DepositInput>
{
    private readonly IEntityFactory _entityFactory = entityFactory ?? throw new ArgumentNullException(nameof(entityFactory));
    private readonly IOutputPort<DepositOutput> _outputHandler = outputHandler ?? throw new ArgumentNullException(nameof(outputHandler));
    private readonly IAccountRepository _accountRepository = accountRepository ?? throw new ArgumentNullException(nameof(accountRepository));
    private readonly IUnitOfWork _unitOfWork = unitOfWork ?? throw new ArgumentNullException(nameof(unitOfWork));
    private readonly IServiceBusClient _serviceBus = serviceBus ?? throw new ArgumentNullException(nameof(serviceBus));

    public async Task ExecuteAsync(DepositInput input, CancellationToken cancellationToken = default)
    {
        var account = await _accountRepository.GetAsync(input.AccountId, cancellationToken);
        if (account == null)
        {
            _outputHandler.Error($"The account {input.AccountId} does not exist or is already closed.");
            return;
        }

        var credit = account.Deposit(_entityFactory, input.Amount);

        await _accountRepository.UpdateAsync(account, credit, cancellationToken);

        await _serviceBus.PublishEventAsync(new Contracts.Events.DepositCompleted() { AccountId = input.AccountId, Amount = input.Amount.ToMoney().ToDecimal() }, cancellationToken);

        await _unitOfWork.SaveAsync(cancellationToken);

        DepositOutput output = new DepositOutput(
            credit,
            account.GetCurrentBalance());

        _outputHandler.Default(output);
    }
}
```

这个代码比较精彩，可以仔细看，这个代码把输入/输出端口模式使用上了，我们甚至可以直接脱离HTTP上下文，能够被API、GRPC甚至是控制台程序复用。

它依赖仓储IAccountRepository 获取聚合根，调用聚合根的业务方法 account.Deposit(...)，最后调用 IUnitOfWork.SaveAsync() 统一提交事务。

```cs
await _accountRepository.UpdateAsync(...); // 1. 更新
await _serviceBus.PublishEventAsync(...);  // 2. 发消息到 RabbitMQ/Redis
await _unitOfWork.SaveAsync(...);          // 3. 提交数据库事务
```

如果第二步消息发送成功了，但是第三步的数据库SaveAsyc因为网络抖动或者主键冲突导致失败了，或者是数据库回滚了，那么消费者就会收到存款成功但是没有落库的情况。

## 事务性收发件

那么怎么避免这种情况，那么应该使用**事务性发件箱模式**

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 900 950" width="100%" height="100%">
  <defs>
    <!-- 基础阴影 -->
    <filter id="shadow" x="-5%" y="-5%" width="110%" height="110%">
      <feDropShadow dx="0" dy="4" stdDeviation="6" flood-color="#000000" flood-opacity="0.4"/>
    </filter>
    
    <!-- 强调发光（用于事务） -->
    <filter id="glow" x="-20%" y="-20%" width="140%" height="140%">
      <feGaussianBlur stdDeviation="4" result="blur" />
      <feComposite in="SourceGraphic" in2="blur" operator="over" />
    </filter>

    <!-- 箭头标记 -->
    <marker id="arrow-blue" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#38bdf8" />
    </marker>
    <marker id="arrow-green" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#10b981" />
    </marker>
    <marker id="arrow-orange" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#f59e0b" />
    </marker>
    <marker id="arrow-purple" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#8b5cf6" />
    </marker>
    <marker id="arrow-gray" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#64748b" />
    </marker>

    <!-- 背景网格 -->
    <pattern id="grid" width="30" height="30" patternUnits="userSpaceOnUse">
      <path d="M 30 0 L 0 0 0 30" fill="none" stroke="#ffffff" stroke-opacity="0.03" stroke-width="1"/>
    </pattern>
  </defs>

  <!-- 背景 -->
  <rect width="100%" height="100%" fill="#0b1120" />
  <rect width="100%" height="100%" fill="url(#grid)" />

  <!-- ================= Subgraph: App (应用层) ================= -->
  <g transform="translate(150, 50)">
    <!-- 容器 -->
    <rect width="600" height="160" rx="16" fill="#1e293b" stroke="#334155" stroke-width="2" opacity="0.6"/>
    <text x="20" y="30" fill="#94a3b8" font-family="sans-serif" font-size="14" font-weight="bold">应用层 / UseCase</text>
    
    <!-- 业务逻辑节点 -->
    <g transform="translate(50, 60)">
      <rect width="200" height="70" rx="8" fill="#0f172a" stroke="#38bdf8" stroke-width="2" filter="url(#shadow)"/>
      <text x="100" y="30" fill="#e2e8f0" font-family="sans-serif" font-size="16" font-weight="bold" text-anchor="middle">1. 业务逻辑</text>
      <text x="100" y="50" fill="#94a3b8" font-family="sans-serif" font-size="12" text-anchor="middle">(如存款)</text>
    </g>

    <!-- 创建消息节点 -->
    <g transform="translate(350, 60)">
      <rect width="200" height="70" rx="8" fill="#0f172a" stroke="#38bdf8" stroke-width="2" filter="url(#shadow)"/>
      <text x="100" y="30" fill="#e2e8f0" font-family="sans-serif" font-size="16" font-weight="bold" text-anchor="middle">2. 创建待发事件消息</text>
      <text x="100" y="50" fill="#94a3b8" font-family="sans-serif" font-size="12" text-anchor="middle">(如 OutboxMessage)</text>
    </g>
  </g>

  <!-- ================= 连线：App -> DB ================= -->
  <!-- 业务逻辑 -> 账户表 -->
  <path d="M 300 200 Q 300 260 300 300" fill="none" stroke="#38bdf8" stroke-width="2" marker-end="url(#arrow-blue)"/>
  <rect x="235" y="235" width="90" height="24" rx="4" fill="#1e293b" />
  <text x="280" y="252" fill="#38bdf8" font-family="sans-serif" font-size="12" text-anchor="middle">更新业务数据</text>

  <!-- 创建消息 -> 发件箱表 -->
  <path d="M 600 200 Q 600 260 600 300" fill="none" stroke="#38bdf8" stroke-width="2" marker-end="url(#arrow-blue)"/>
  <rect x="545" y="235" width="90" height="24" rx="4" fill="#1e293b" />
  <text x="590" y="252" fill="#38bdf8" font-family="sans-serif" font-size="12" text-anchor="middle">插入待发消息</text>


  <!-- ================= Subgraph: Database (数据库) ================= -->
  <g transform="translate(100, 320)">
    <!-- 容器 -->
    <rect width="700" height="260" rx="16" fill="#132e24" stroke="#10b981" stroke-width="2" opacity="0.8" filter="url(#shadow)"/>
    <text x="20" y="30" fill="#34d399" font-family="sans-serif" font-size="14" font-weight="bold">同一个 SQL Server 数据库事务</text>
    
    <!-- 账户表 -->
    <g transform="translate(80, 80)">
      <rect width="220" height="80" rx="8" fill="#0f172a" stroke="#10b981" stroke-width="2" />
      <!-- 表头装饰线 -->
      <line x1="0" y1="30" x2="220" y2="30" stroke="#10b981" stroke-width="1" stroke-opacity="0.5"/>
      <text x="110" y="20" fill="#a7f3d0" font-family="monospace" font-size="14" font-weight="bold" text-anchor="middle">Account / Credit 表</text>
      <text x="110" y="55" fill="#64748b" font-family="monospace" font-size="12" text-anchor="middle">业务数据变更 (UPDATE)</text>
    </g>

    <!-- 发件箱表 -->
    <g transform="translate(400, 80)">
      <rect width="220" height="80" rx="8" fill="#0f172a" stroke="#10b981" stroke-width="2" />
      <line x1="0" y1="30" x2="220" y2="30" stroke="#10b981" stroke-width="1" stroke-opacity="0.5"/>
      <text x="110" y="20" fill="#a7f3d0" font-family="monospace" font-size="14" font-weight="bold" text-anchor="middle">OutboxMessages 表</text>
      <text x="110" y="55" fill="#64748b" font-family="monospace" font-size="12" text-anchor="middle">消息暂存 (INSERT)</text>
    </g>

    <!-- 强事务连线 -->
    <path d="M 300 440 L 400 440" fill="none" stroke="#10b981" stroke-width="3" stroke-dasharray="8,4" marker-end="url(#arrow-green)"/>
    
    <!-- 事务说明标签 -->
    <g transform="translate(150, 180)">
      <rect width="400" height="50" rx="8" fill="#064e3b" stroke="#10b981" stroke-width="2" filter="url(#glow)"/>
      <text x="200" y="20" fill="#d1fae5" font-family="sans-serif" font-size="14" font-weight="bold" text-anchor="middle">【同一个 SaveAsync() 本地事务】</text>
      <text x="200" y="38" fill="#34d399" font-family="sans-serif" font-size="12" text-anchor="middle">要么同时成功，要么同时回滚！</text>
    </g>
  </g>


  <!-- ================= 连线：DB -> Worker ================= -->
  <!-- 发件箱 -> 后台拉取 -->
  <path d="M 450 580 Q 450 640 250 670" fill="none" stroke="#64748b" stroke-width="2" marker-end="url(#arrow-gray)"/>
  <rect x="335" y="615" width="100" height="24" rx="4" fill="#1e293b" />
  <text x="385" y="632" fill="#cbd5e1" font-family="sans-serif" font-size="12" text-anchor="middle">读取未发送消息</text>
  

  <!-- ================= Subgraph: BackgroundWorker ================= -->
  <g transform="translate(50, 680)">
    <!-- 容器 -->
    <rect width="300" height="200" rx="16" fill="#2e1f11" stroke="#f59e0b" stroke-width="2" opacity="0.8" filter="url(#shadow)"/>
    <text x="20" y="30" fill="#fcd34d" font-family="sans-serif" font-size="14" font-weight="bold">后台发布进程 (Worker)</text>

    <!-- 轮询器 -->
    <g transform="translate(40, 60)">
      <rect width="220" height="50" rx="8" fill="#0f172a" stroke="#f59e0b" stroke-width="2" />
      <text x="110" y="30" fill="#fde68a" font-family="sans-serif" font-size="14" font-weight="bold" text-anchor="middle">定期拉取 / CDC</text>
    </g>

    <!-- 标记器 -->
    <g transform="translate(40, 130)">
      <rect width="220" height="50" rx="8" fill="#0f172a" stroke="#f59e0b" stroke-width="2" />
      <text x="110" y="30" fill="#fde68a" font-family="sans-serif" font-size="14" font-weight="bold" text-anchor="middle">标记消息为已发送</text>
    </g>
  </g>


  <!-- ================= Subgraph: Broker ================= -->
  <g transform="translate(550, 680)">
    <!-- 容器 -->
    <rect width="300" height="200" rx="16" fill="#2e1065" stroke="#8b5cf6" stroke-width="2" opacity="0.8" filter="url(#shadow)"/>
    <text x="20" y="30" fill="#c4b5fd" font-family="sans-serif" font-size="14" font-weight="bold">消息代理 (Broker)</text>

    <!-- RabbitMQ -->
    <g transform="translate(40, 80)">
      <rect width="220" height="70" rx="8" fill="#0f172a" stroke="#8b5cf6" stroke-width="2" />
      <text x="110" y="30" fill="#ddd6fe" font-family="sans-serif" font-size="16" font-weight="bold" text-anchor="middle">RabbitMQ</text>
      <text x="110" y="50" fill="#a78bfa" font-family="sans-serif" font-size="12" text-anchor="middle">交换机 / 队列</text>
    </g>
  </g>


  <!-- ================= Worker 与 Broker 间的交互 ================= -->
  <!-- Poller -> RabbitMQ (投递) -->
  <path d="M 310 765 L 580 765" fill="none" stroke="#f59e0b" stroke-width="2" marker-end="url(#arrow-orange)"/>
  <rect x="395" y="742" width="100" height="24" rx="4" fill="#1e293b" />
  <text x="445" y="759" fill="#fbbf24" font-family="sans-serif" font-size="12" text-anchor="middle">投递事件</text>

  <!-- RabbitMQ -> MarkSent (ACK) -->
  <path d="M 580 815 L 310 815" fill="none" stroke="#8b5cf6" stroke-width="2" stroke-dasharray="4,2" marker-end="url(#arrow-purple)"/>
  <rect x="395" y="830" width="100" height="24" rx="4" fill="#1e293b" />
  <text x="445" y="847" fill="#c4b5fd" font-family="sans-serif" font-size="12" text-anchor="middle">ACK 确认</text>

  <!-- MarkSent -> OutboxTable (更新状态) -->
  <path d="M 200 680 Q 200 620 450 580" fill="none" stroke="#f59e0b" stroke-width="2" marker-end="url(#arrow-orange)"/>
  <rect x="315" y="650" width="120" height="24" rx="4" fill="#1e293b" />
  <text x="375" y="667" fill="#fbbf24" font-family="sans-serif" font-size="12" text-anchor="middle">更新为 Processed</text>

</svg>


可以在数据库新建一张叫做OutboxMessage的表：

```sql
CREATE TABLE OutboxMessages (
	Id UNIQUEIDENTIFIER PRIMARY KEY,
	OccurredOn DATETIME2 NOT NULL,
	Type NVARCHAR(255) NOT NULL,
	Content NVARCHAR(MAX) NOT NULL,
	ProcessedOn DATETIME2 NULL
);
```

在用例中统一落库：

```cs
  // 业务更新
    await _accountRepository.UpdateAsync(account, credit, cancellationToken);

    // 不直接调 RabbitMQ 发网络请求，而是向本地 Outbox 表塞一条记录
    _context.OutboxMessages.Add(new OutboxMessage(
        id: Guid.NewGuid(),
        type: nameof(DepositCompleted),
        content: JsonSerializer.Serialize(new DepositCompleted { ... })
    ));

    // 关键点：业务数据和 Outbox 消息在【同一个数据库事务中原子提交】！
    // 如果报错，全部回滚，绝对不会产生“幽灵消息”！
    await _unitOfWork.SaveAsync(cancellationToken);
```

后台 Worker 异步保障投递：

• 项目中的 dddotNet.Worker 可以起一个后台任务（BackgroundService）；
• 每隔几百毫秒扫一次 OutboxMessages 表中 ProcessedOn IS NULL 的记录，投递给 RabbitMQ；
• RabbitMQ 返回确认（ACK）后，打上 ProcessedOn = DateTime.UtcNow；
• 即使宕机，下次重启会继续扫表重投（保证至少投递一次 At-Least-Once Delivery）。

在真实的开发场景中，通常是无需手写发件箱轮询的，因为.net生态有两个及其成熟的解决方案：

MassTransit Outbox（推荐）：如果该项目后续将 Rebus 升级为 MassTransit，直接开启：`cfg.AddEntityFrameworkOutbox<GenocsContext>()` 业务代码只需 await publishEndpoint.Publish(...)，框架会自动劫持并走 Outbox 事务落库与后台派发。

DotNetCore.CAP：国内广泛采用的分布式事务与 EventBus 方案，原生内置基于 EF Core 的 Outbox 实现与自动重试机制。

我们这里通过MassTransit Outbox来改造一下，后续也可以将rebus升级为masstransit。

核心执行时序（Outbox 怎么解决双写问题）：

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 1200 1000" width="100%" height="100%">
  <defs>
    <!-- 基础阴影 -->
    <filter id="shadow" x="-5%" y="-5%" width="110%" height="110%">
      <feDropShadow dx="0" dy="4" stdDeviation="4" flood-color="#000000" flood-opacity="0.4"/>
    </filter>

    <!-- 箭头标记 (实线) -->
    <marker id="arrow-solid" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#94a3b8" />
    </marker>
    
    <!-- 箭头标记 (虚线返回) -->
    <marker id="arrow-dashed" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#cbd5e1" />
    </marker>

    <!-- 背景网格 -->
    <pattern id="grid" width="40" height="40" patternUnits="userSpaceOnUse">
      <path d="M 40 0 L 0 0 0 40" fill="none" stroke="#ffffff" stroke-opacity="0.03" stroke-width="1"/>
    </pattern>
  </defs>

  <!-- 背景 -->
  <rect width="100%" height="100%" fill="#0f172a" />
  <rect width="100%" height="100%" fill="url(#grid)" />

  <!-- ================= Participants (顶部角色) ================= -->
  <!-- UC -->
  <g transform="translate(50, 40)">
    <rect width="150" height="40" rx="6" fill="#1e293b" stroke="#3b82f6" stroke-width="2" filter="url(#shadow)"/>
    <text x="75" y="25" fill="#60a5fa" font-family="sans-serif" font-size="14" font-weight="bold" text-anchor="middle">UseCase (Deposit)</text>
  </g>
  <!-- Repo -->
  <g transform="translate(240, 40)">
    <rect width="130" height="40" rx="6" fill="#1e293b" stroke="#06b6d4" stroke-width="2" filter="url(#shadow)"/>
    <text x="65" y="25" fill="#22d3ee" font-family="sans-serif" font-size="14" font-weight="bold" text-anchor="middle">AccountRepo</text>
  </g>
  <!-- Bus -->
  <g transform="translate(410, 40)">
    <rect width="130" height="40" rx="6" fill="#1e293b" stroke="#8b5cf6" stroke-width="2" filter="url(#shadow)"/>
    <text x="65" y="25" fill="#a78bfa" font-family="sans-serif" font-size="14" font-weight="bold" text-anchor="middle">MassTransit</text>
  </g>
  <!-- DB -->
  <g transform="translate(580, 40)">
    <rect width="130" height="40" rx="6" fill="#1e293b" stroke="#10b981" stroke-width="2" filter="url(#shadow)"/>
    <text x="65" y="25" fill="#34d399" font-family="sans-serif" font-size="14" font-weight="bold" text-anchor="middle">SQL Server</text>
  </g>
  <!-- Worker -->
  <g transform="translate(750, 40)">
    <rect width="130" height="40" rx="6" fill="#1e293b" stroke="#f59e0b" stroke-width="2" filter="url(#shadow)"/>
    <text x="65" y="25" fill="#fbbf24" font-family="sans-serif" font-size="14" font-weight="bold" text-anchor="middle">后台调度器</text>
  </g>
  <!-- MQ -->
  <g transform="translate(920, 40)">
    <rect width="130" height="40" rx="6" fill="#1e293b" stroke="#ec4899" stroke-width="2" filter="url(#shadow)"/>
    <text x="65" y="25" fill="#f472b6" font-family="sans-serif" font-size="14" font-weight="bold" text-anchor="middle">RabbitMQ</text>
  </g>

  <!-- ================= Lifelines (生命线) ================= -->
  <line x1="125" y1="80" x2="125" y2="920" stroke="#334155" stroke-width="2" stroke-dasharray="6,4" />
  <line x1="305" y1="80" x2="305" y2="920" stroke="#334155" stroke-width="2" stroke-dasharray="6,4" />
  <line x1="475" y1="80" x2="475" y2="920" stroke="#334155" stroke-width="2" stroke-dasharray="6,4" />
  <line x1="645" y1="80" x2="645" y2="920" stroke="#334155" stroke-width="2" stroke-dasharray="6,4" />
  <line x1="815" y1="80" x2="815" y2="920" stroke="#334155" stroke-width="2" stroke-dasharray="6,4" />
  <line x1="985" y1="80" x2="985" y2="920" stroke="#334155" stroke-width="2" stroke-dasharray="6,4" />

  <!-- ================= 1. UC -> Repo ================= -->
  <g transform="translate(0, 120)">
    <line x1="125" y1="0" x2="295" y2="0" stroke="#94a3b8" stroke-width="2" marker-end="url(#arrow-solid)"/>
    <circle cx="125" cy="0" r="12" fill="#3b82f6" />
    <text x="125" y="4" fill="#ffffff" font-family="sans-serif" font-size="12" font-weight="bold" text-anchor="middle">1</text>
    <text x="210" y="-10" fill="#e2e8f0" font-family="sans-serif" font-size="14" text-anchor="middle">更新业务聚合状态</text>
  </g>

  <!-- ================= Repo -> DB ================= -->
  <g transform="translate(0, 170)">
    <line x1="305" y1="0" x2="635" y2="0" stroke="#94a3b8" stroke-width="2" marker-end="url(#arrow-solid)"/>
    <text x="470" y="-10" fill="#e2e8f0" font-family="sans-serif" font-size="14" text-anchor="middle">变更挂入 EF Core ChangeTracker</text>
  </g>

  <!-- ================= 2. UC -> Bus ================= -->
  <g transform="translate(0, 240)">
    <line x1="125" y1="0" x2="465" y2="0" stroke="#94a3b8" stroke-width="2" marker-end="url(#arrow-solid)"/>
    <circle cx="125" cy="0" r="12" fill="#3b82f6" />
    <text x="125" y="4" fill="#ffffff" font-family="sans-serif" font-size="12" font-weight="bold" text-anchor="middle">2</text>
    <text x="295" y="-10" fill="#e2e8f0" font-family="sans-serif" font-size="14" text-anchor="middle">_serviceBus.PublishEventAsync(...)</text>
  </g>

  <!-- ================= Note over Bus, DB ================= -->
  <g transform="translate(400, 280)">
    <rect width="290" height="60" rx="6" fill="#2e1065" stroke="#8b5cf6" stroke-width="1" filter="url(#shadow)"/>
    <path d="M 0 10 L 15 0 L 0 10 L 0 60 Z" fill="#8b5cf6" opacity="0.5"/>
    <text x="145" y="25" fill="#ddd6fe" font-family="sans-serif" font-size="13" font-weight="bold" text-anchor="middle">【关键】：MassTransit 不直接发网络请求</text>
    <text x="145" y="45" fill="#c4b5fd" font-family="sans-serif" font-size="13" text-anchor="middle">而是将消息实体挂入同一个 DbContext！</text>
  </g>

  <!-- ================= 3. UC -> DB ================= -->
  <g transform="translate(0, 380)">
    <line x1="125" y1="0" x2="635" y2="0" stroke="#94a3b8" stroke-width="2" marker-end="url(#arrow-solid)"/>
    <circle cx="125" cy="0" r="12" fill="#3b82f6" />
    <text x="125" y="4" fill="#ffffff" font-family="sans-serif" font-size="12" font-weight="bold" text-anchor="middle">3</text>
    <text x="380" y="-10" fill="#e2e8f0" font-family="sans-serif" font-size="14" text-anchor="middle">await _unitOfWork.SaveAsync()</text>
  </g>

  <!-- ================= Note over DB ================= -->
  <g transform="translate(520, 420)">
    <rect width="250" height="90" rx="6" fill="#064e3b" stroke="#10b981" stroke-width="1" filter="url(#shadow)"/>
    <path d="M 0 10 L 15 0 L 0 10 L 0 90 Z" fill="#10b981" opacity="0.5"/>
    <text x="125" y="25" fill="#a7f3d0" font-family="sans-serif" font-size="13" font-weight="bold" text-anchor="middle">【同一个本地 ACID 事务】提交：</text>
    <text x="125" y="45" fill="#6ee7b7" font-family="sans-serif" font-size="13" text-anchor="middle">• 业务流水表更新</text>
    <text x="125" y="65" fill="#6ee7b7" font-family="sans-serif" font-size="13" text-anchor="middle">• OutboxMessage 待发消息表插入</text>
    <text x="125" y="85" fill="#fca5a5" font-family="sans-serif" font-size="12" text-anchor="middle">若任一失败，全部原子回滚！</text>
  </g>


  <!-- ================= rect: Background Worker Async ================= -->
  <g transform="translate(710, 540)">
    <rect width="440" height="320" rx="8" fill="#f59e0b" fill-opacity="0.05" stroke="#f59e0b" stroke-width="1.5" stroke-dasharray="5,3"/>
    <rect width="220" height="24" x="110" y="-12" rx="4" fill="#2e1f11" />
    <text x="220" y="4" fill="#fbbf24" font-family="sans-serif" font-size="13" font-weight="bold" text-anchor="middle">【后台异步保障投递（至少投递一次）】</text>
  </g>

  <!-- Worker -> DB (轮询) -->
  <g transform="translate(0, 600)">
    <line x1="815" y1="0" x2="655" y2="0" stroke="#94a3b8" stroke-width="2" marker-end="url(#arrow-solid)"/>
    <text x="735" y="-10" fill="#e2e8f0" font-family="sans-serif" font-size="14" text-anchor="middle">轮询检测已提交的 Outbox 消息</text>
  </g>

  <!-- Worker -> MQ (投递) -->
  <g transform="translate(0, 670)">
    <line x1="815" y1="0" x2="975" y2="0" stroke="#94a3b8" stroke-width="2" marker-end="url(#arrow-solid)"/>
    <text x="895" y="-10" fill="#e2e8f0" font-family="sans-serif" font-size="14" text-anchor="middle">投递到 RabbitMQ</text>
  </g>

  <!-- MQ -->> Worker (ACK 确认) -->
  <g transform="translate(0, 730)">
    <line x1="985" y1="0" x2="825" y2="0" stroke="#cbd5e1" stroke-width="2" stroke-dasharray="6,4" marker-end="url(#arrow-dashed)"/>
    <text x="905" y="-10" fill="#94a3b8" font-family="sans-serif" font-size="14" text-anchor="middle">ACK 确认</text>
  </g>

  <!-- Worker -> DB (标记已发送) -->
  <g transform="translate(0, 800)">
    <line x1="815" y1="0" x2="655" y2="0" stroke="#94a3b8" stroke-width="2" marker-end="url(#arrow-solid)"/>
    <text x="735" y="-10" fill="#e2e8f0" font-family="sans-serif" font-size="14" text-anchor="middle">标记消息为已发送 / 清理</text>
  </g>

</svg>

实现：

 在 Infrastructure.csproj 中引入了 MassTransit 的 EF Core 发件箱与 RabbitMQ 传输库，我们在GenocsContext.cs:18-21 的 OnModelCreating 中注入发件箱表映射：

```cs
namespace dddotNet.Infrastructure.PersistenceLayer.SQLServer;

public sealed class GenocsContext(DbContextOptions options) : DbContext(options)
{
    public DbSet<Account> Accounts { get; set; }
    public DbSet<Customer> Customers { get; set; }
    public DbSet<Credit> Credits { get; set; }
    public DbSet<Debit> Debits { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        // MassTransit EntityFrameworkCore Outbox 实体映射
        modelBuilder.AddInboxStateEntity();
        modelBuilder.AddOutboxMessageEntity();
        modelBuilder.AddOutboxStateEntity();

        modelBuilder.Entity<Account>()
            .ToTable("Account");

        modelBuilder.Entity<Account>()
            .Ignore(p => p.Credits)
            .Ignore(p => p.Debits);

        modelBuilder.Entity<Customer>()
            .ToTable("Customer")
            .Property(b => b.SSN)
            .HasConversion(
                v => v.ToString(),
                v => new SSN(v));

        modelBuilder.Entity<Customer>()
            .ToTable("Customer")
            .Property(b => b.Name)
            .HasConversion(
                v => v.ToString(),
                v => new Name(v));

        modelBuilder.Entity<Customer>()
            .Ignore(p => p.Accounts);

        modelBuilder.Entity<Debit>()
            .ToTable("Debit")
            .Property(b => b.Amount)
            .HasConversion(
                v => v.ToMoney().ToDecimal(),
                v => new PositiveMoney(v));

        modelBuilder.Entity<Credit>()
            .ToTable("Credit")
            .Property(b => b.Amount)
            .HasConversion(
                v => v.ToMoney().ToDecimal(),
                v => new PositiveMoney(v));
    }
}

```

实现与架构抽象对齐的 ServiceBusClient，创建 MassTransitServiceBusClient.cs：

```cs
public sealed class MassTransitServiceBusClient(IPublishEndpoint publishEndpoint) : IServiceBusClient
{
    private readonly IPublishEndpoint _publishEndpoint = publishEndpoint ?? throw new ArgumentNullException(nameof(publishEndpoint));

    public async Task SendCommandAsync<T>(T command, CancellationToken cancellationToken = default)
        where T : Genocs.Common.CQRS.Commands.ICommand
    {
        await _publishEndpoint.Publish(command, cancellationToken);
    }

    public async Task PublishEventAsync<T>(T message, CancellationToken cancellationToken = default)
        where T : Genocs.Common.CQRS.Events.IEvent
    {
        await _publishEndpoint.Publish(message, cancellationToken);
    }
}

```

在 `Deposit` 用例中，调用了 `await _serviceBus.PublishEventAsync(...)`。  
在这个实现里，底层实际上是调用了 MassTransit 的 `_publishEndpoint.Publish(...)`。因为这里注入的是 `IPublishEndpoint`（而不是 `IBus` 或 `ISendEndpoint`），当你的项目配置了 MassTransit EF Core Outbox 后，这个 `Publish` 方法根本不会立刻向 RabbitMQ 发送网络请求

它会将消息实体附加到当前的 EF Core `DbContext` 中。随后你的 `_unitOfWork.SaveAsync()` 执行时，业务数据和 Outbox 消息会在**同一个本地事务**中提交。

然后我们去在 DI 容器中开启 Bus Outbox 管道，创建了配置扩展MassTransitInfrastructureExtensions.cs：


```cs
 services.AddMassTransit(x =>
    {
        // 挂载 EF Core 事务性发件箱
        x.AddEntityFrameworkOutbox<GenocsContext>(o =>
        {
            o.UseSqlServer();
            o.UseBusOutbox(); // 启用总线发件箱：所有发布事件进入同一个 DbContext 事务
        });

        // 配置 RabbitMQ 传输
        x.UsingRabbitMq((context, cfg) =>
        {
            cfg.Host(rabbitHost, virtualHost, h =>
            {
                h.Username(rabbitUser);
                h.Password(rabbitPass);
            });
            cfg.ConfigureEndpoints(context);
        });
    });

    // 将抽象接口注入为 Scoped，与当前请求的 DbContext 处于同一个事务生命周期
    services.AddScoped<IServiceBusClient, MassTransitServiceBusClient>();
```

并在 Program.cs:58 中完成了装配替换：

```cs
services.AddMassTransitWithOutbox(builder.Configuration);
```

这样即便上面的第 3 步 `_unitOfWork.SaveAsync()` 发生主键冲突或网络断连，业务表与 Outbox 表同时回滚，绝不会向 RabbitMQ发出幽灵消息。

## 回滚机制


回滚把一个原本无法回滚的网络调用，变成了一个能参与本地事务的数据表写入。

在改造前，当执行到第二步时：

```cs
    await _serviceBus.PublishEventAsync(...); // 此时直接通过 TCP 发送 AMQP 数据包给 RabbitMQ
    await _unitOfWork.SaveAsync(...);         // 此时向 SQL Server 发送 SQL
```

改造和配置了 `x.AddEntityFrameworkOutbox<GenocsContext>() 和 o.UseBusOutbox()` 之后，底层的执行逻辑发生了根本性改变

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 1100 1050" width="100%" height="100%">
  <defs>
    <!-- 基础阴影 -->
    <filter id="shadow" x="-5%" y="-5%" width="110%" height="110%">
      <feDropShadow dx="0" dy="4" stdDeviation="5" flood-color="#000000" flood-opacity="0.4"/>
    </filter>

    <!-- 箭头标记 -->
    <marker id="arrow-gray" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#94a3b8" />
    </marker>
    <marker id="arrow-green" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#10b981" />
    </marker>
    <marker id="arrow-red" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#ef4444" />
    </marker>
    <marker id="arrow-orange" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#f59e0b" />
    </marker>

    <!-- 背景网格 -->
    <pattern id="grid" width="40" height="40" patternUnits="userSpaceOnUse">
      <path d="M 40 0 L 0 0 0 40" fill="none" stroke="#ffffff" stroke-opacity="0.03" stroke-width="1"/>
    </pattern>
  </defs>

  <!-- 全局背景 -->
  <rect width="100%" height="100%" fill="#0b1120" />
  <rect width="100%" height="100%" fill="url(#grid)" />

  <!-- ================= 1. 业务调用阶段 (App) ================= -->
  <g transform="translate(50, 50)">
    <rect width="1000" height="160" rx="12" fill="#1e293b" stroke="#3b82f6" stroke-width="2" opacity="0.8" filter="url(#shadow)"/>
    <text x="20" y="30" fill="#60a5fa" font-family="sans-serif" font-size="16" font-weight="bold">1. 业务调用阶段</text>
    
    <g transform="translate(150, 50)">
      <rect width="250" height="60" rx="8" fill="#0f172a" stroke="#3b82f6" stroke-width="2" />
      <text x="125" y="25" fill="#e2e8f0" font-family="sans-serif" font-size="15" font-weight="bold" text-anchor="middle">Repo.UpdateAsync</text>
      <text x="125" y="45" fill="#94a3b8" font-family="sans-serif" font-size="12" text-anchor="middle">(挂载 Credit 实体)</text>
    </g>

    <g transform="translate(600, 50)">
      <rect width="250" height="60" rx="8" fill="#0f172a" stroke="#3b82f6" stroke-width="2" />
      <text x="125" y="25" fill="#e2e8f0" font-family="sans-serif" font-size="15" font-weight="bold" text-anchor="middle">PublishEventAsync</text>
      <text x="125" y="45" fill="#fbbf24" font-family="sans-serif" font-size="12" text-anchor="middle">(被 Outbox 拦截)</text>
    </g>
  </g>

  <!-- 连线 App -> Memory -->
  <path d="M 325 210 L 325 260" fill="none" stroke="#94a3b8" stroke-width="2" marker-end="url(#arrow-gray)"/>
  <path d="M 775 210 L 775 260" fill="none" stroke="#94a3b8" stroke-width="2" marker-end="url(#arrow-gray)"/>
  
  <!-- 特殊标签：Publish 不发网络请求 -->
  <g transform="translate(820, 220)">
    <rect width="200" height="30" rx="6" fill="#2e1065" stroke="#8b5cf6" stroke-width="1" />
    <text x="100" y="20" fill="#c4b5fd" font-family="sans-serif" font-size="12" font-weight="bold" text-anchor="middle">【不发网络请求】仅转为实体</text>
  </g>

  <!-- ================= 2. EF Core 内存 ChangeTracker (Memory) ================= -->
  <g transform="translate(50, 280)">
    <rect width="1000" height="180" rx="12" fill="#1e293b" stroke="#8b5cf6" stroke-width="2" opacity="0.8" filter="url(#shadow)"/>
    <text x="20" y="30" fill="#a78bfa" font-family="sans-serif" font-size="16" font-weight="bold">EF Core 内存 ChangeTracker (变更追踪器)</text>
    
    <g transform="translate(150, 60)">
      <rect width="250" height="70" rx="8" fill="#0f172a" stroke="#8b5cf6" stroke-width="2" />
      <text x="125" y="30" fill="#e2e8f0" font-family="sans-serif" font-size="15" font-weight="bold" text-anchor="middle">待插入实体：Credit</text>
      <text x="125" y="50" fill="#94a3b8" font-family="sans-serif" font-size="12" text-anchor="middle">(State: Added)</text>
    </g>

    <g transform="translate(600, 60)">
      <rect width="250" height="70" rx="8" fill="#0f172a" stroke="#8b5cf6" stroke-width="2" />
      <text x="125" y="30" fill="#e2e8f0" font-family="sans-serif" font-size="15" font-weight="bold" text-anchor="middle">待插入实体：OutboxMessage</text>
      <text x="125" y="50" fill="#94a3b8" font-family="sans-serif" font-size="12" text-anchor="middle">(事件被序列化为数据)</text>
    </g>
  </g>

  <!-- 连线 Memory -> DB -->
  <path d="M 325 460 L 325 520 L 500 520 L 500 550" fill="none" stroke="#94a3b8" stroke-width="2" marker-end="url(#arrow-gray)"/>
  <path d="M 775 460 L 775 520 L 600 520 L 600 550" fill="none" stroke="#94a3b8" stroke-width="2" marker-end="url(#arrow-gray)"/>

  <!-- ================= 3. 数据库事务执行阶段 (DB) ================= -->
  <g transform="translate(50, 560)">
    <rect width="1000" height="320" rx="12" fill="#064e3b" stroke="#10b981" stroke-width="2" opacity="0.8" filter="url(#shadow)"/>
    <text x="20" y="30" fill="#34d399" font-family="sans-serif" font-size="16" font-weight="bold">2. 数据库事务执行阶段 (SaveAsync)</text>

    <!-- SQL 语句框 -->
    <g transform="translate(280, 50)">
      <rect width="440" height="80" rx="8" fill="#0f172a" stroke="#10b981" stroke-width="2" />
      <text x="220" y="25" fill="#a7f3d0" font-family="monospace" font-size="13" font-weight="bold" text-anchor="middle">BEGIN TRANSACTION;</text>
      <text x="220" y="45" fill="#e2e8f0" font-family="monospace" font-size="13" text-anchor="middle">INSERT INTO Credit ...;</text>
      <text x="220" y="65" fill="#e2e8f0" font-family="monospace" font-size="13" text-anchor="middle">INSERT INTO OutboxMessage ...;</text>
    </g>

    <!-- 指向 Check -->
    <path d="M 500 130 L 500 170" fill="none" stroke="#94a3b8" stroke-width="2" marker-end="url(#arrow-gray)"/>

    <!-- 条件判断 -->
    <g transform="translate(400, 180)">
      <polygon points="100,0 200,40 100,80 0,40" fill="#0f172a" stroke="#f59e0b" stroke-width="2"/>
      <text x="100" y="35" fill="#fbbf24" font-family="sans-serif" font-size="14" font-weight="bold" text-anchor="middle">执行是否异常?</text>
      <text x="100" y="55" fill="#94a3b8" font-family="sans-serif" font-size="12" text-anchor="middle">主键冲突 / 网络故障</text>
    </g>

    <!-- 分支：无异常 (Commit) -->
    <path d="M 600 220 L 700 220 L 700 250" fill="none" stroke="#10b981" stroke-width="2" marker-end="url(#arrow-green)"/>
    <text x="650" y="210" fill="#10b981" font-family="sans-serif" font-size="12" font-weight="bold">无异常</text>
    
    <g transform="translate(580, 260)">
      <rect width="240" height="60" rx="8" fill="#022c22" stroke="#10b981" stroke-width="2" />
      <text x="120" y="25" fill="#34d399" font-family="sans-serif" font-size="14" font-weight="bold" text-anchor="middle">COMMIT TRANSACTION;</text>
      <text x="120" y="45" fill="#94a3b8" font-family="sans-serif" font-size="12" text-anchor="middle">(业务表与消息表同时永久保存)</text>
    </g>

    <!-- 分支：发生异常 (Rollback) -->
    <path d="M 400 220 L 300 220 L 300 250" fill="none" stroke="#ef4444" stroke-width="2" marker-end="url(#arrow-red)"/>
    <text x="350" y="210" fill="#ef4444" font-family="sans-serif" font-size="12" font-weight="bold">发生异常</text>

    <g transform="translate(180, 260)">
      <rect width="240" height="60" rx="8" fill="#450a0a" stroke="#ef4444" stroke-width="2" />
      <text x="120" y="25" fill="#fca5a5" font-family="sans-serif" font-size="14" font-weight="bold" text-anchor="middle">ROLLBACK TRANSACTION;</text>
      <text x="120" y="45" fill="#fca5a5" font-family="sans-serif" font-size="12" text-anchor="middle">(Credit 和 Outbox 同时抹去)</text>
    </g>
  </g>

  <!-- 连线 Commit -> Worker -->
  <path d="M 750 880 L 750 940" fill="none" stroke="#10b981" stroke-width="2" marker-end="url(#arrow-green)"/>

  <!-- 特殊连线 Rollback -> Worker (虚线，表示没有消息) -->
  <path d="M 250 880 L 250 940" fill="none" stroke="#ef4444" stroke-width="2" stroke-dasharray="6,4" marker-end="url(#arrow-red)"/>
  <g transform="translate(140, 900)">
    <rect width="160" height="24" rx="4" fill="#1e293b" />
    <text x="80" y="17" fill="#fca5a5" font-family="sans-serif" font-size="12" text-anchor="middle">库里根本没有消息</text>
  </g>

  <!-- ================= 4. 后台发信阶段 (AsyncWorker) ================= -->
  <g transform="translate(50, 950)">
    <rect width="1000" height="150" rx="12" fill="#1e293b" stroke="#f59e0b" stroke-width="2" opacity="0.8" filter="url(#shadow)"/>
    <text x="20" y="30" fill="#fbbf24" font-family="sans-serif" font-size="16" font-weight="bold">3. 后台发信阶段 (Outbox Processor)</text>

    <g transform="translate(200, 60)">
      <rect width="250" height="60" rx="8" fill="#0f172a" stroke="#f59e0b" stroke-width="2" />
      <text x="125" y="25" fill="#e2e8f0" font-family="sans-serif" font-size="15" font-weight="bold" text-anchor="middle">Poller (扫描未发送)</text>
      <text x="125" y="45" fill="#94a3b8" font-family="sans-serif" font-size="12" text-anchor="middle">扫描数据库已提交的消息</text>
    </g>

    <path d="M 450 90 L 580 90" fill="none" stroke="#94a3b8" stroke-width="2" marker-end="url(#arrow-gray)"/>

    <g transform="translate(600, 60)">
      <rect width="250" height="60" rx="8" fill="#0f172a" stroke="#f59e0b" stroke-width="2" />
      <text x="125" y="25" fill="#e2e8f0" font-family="sans-serif" font-size="15" font-weight="bold" text-anchor="middle">向 RabbitMQ 发送并 ACK</text>
      <text x="125" y="45" fill="#94a3b8" font-family="sans-serif" font-size="12" text-anchor="middle">(异步保障投递)</text>
    </g>
  </g>

</svg>

当调用了 `await _serviceBus.PublishEventAsync(new DepositCompleted { ... });` 的时候，由于开启了UseBusOutbox的MassTransit根本没有向RabbitMQ发送任何网络通信，它仅仅把事件对象序列化为了json文本，创建了一个OutboxMessage数据实体，直接塞进了当前同一个GenocsContext 的内存变更追踪器中。

当紧接着调用： `await _unitOfWork.SaveAsync(cancellationToken);` 之后，在底层，EF Core 的 SaveChangesAsync 默认会自动打开一个 SQL Server本地事务。它向数据库发送的实际上是这样一段批处理：

```sql
   BEGIN TRANSACTION;

    -- ① 插入充值流水业务记录
    INSERT INTO [Credit] ([Id], [AccountId], [Amount], [TransactionDate])
    VALUES ('c1...', 'a1...', 100.00, '2026-10-25');

    -- ② 插入待发消息发件箱记录
    INSERT INTO [OutboxMessage] ([SequenceNumber], [MessageId], [Body], ...)
    VALUES (1, 'm1...', '{"AccountId":"a1...", "Amount":100}', ...);

    COMMIT TRANSACTION;
```

如果一切正常：两个 INSERT 全部成功，COMMIT TRANSACTION，数据与待发消息同时落盘，如果第 ① 步或第 ② 步有任何报错（例如主键冲突、外键约束、数据库磁盘满、网络超时中断）：
      • 数据库引擎立即触发 ROLLBACK TRANSACTION；
      • Credit 记录回滚了，OutboxMessage 待发消息记录也同步被物理撤销/回滚！
      • 内存里的 DbContext 抛出异常，整个用例中止。

在整个过程中，负责真正往 RabbitMQ 发消息的是 MassTransit 注册在后台的独立工作者：

  • 这个后台工作者只会去读取数据库中已经真正 Commit 成功的 OutboxMessage 记录。
  • 既然刚才的事务回滚了，数据库里压根就没有产生过这条 OutboxMessage 记录！
  • 后台工作者什么也查不到，因此 RabbitMQ 永远不会收到这笔错误的存款事件。

### 内存追踪变更器

 如果没有 ChangeTracker，上述的 Outbox 机制、工作单元以及整洁架构里的事务一致性，根本无法成立。这个也是ef core的关键部分，下面我们拆解一下它是如何工作的：

ef core内部，每一个被管理的对象，都会被包装为一个EntityEntry，ChangeTracker会为其维护一个状态叫做EntiryState

EntityState 的 5 种状态

|状态|含义|场景 / 如何触发|`SaveChanges()` 时的动作|
|---|---|---|---|
|**`Added`**|新增|调用了 `_dbContext.Add(entity)` 或构造函数创建的新对象被添加|生成 `INSERT` 语句|
|**`Unchanged`**|未改变|从数据库 `GetAsync` 查出来，且没做任何修改|忽略，不生成 SQL|
|**`Modified`**|已修改|查出来后，修改了实体的某个属性|生成 `UPDATE` 语句（只更新修改的字段）|
|**`Deleted`**|已删除|调用了 `_dbContext.Remove(entity)`|生成 `DELETE` 语句|
|**`Detached`**|游离|对象没有在 `ChangeTracker` 的追踪范围内（比如 `new` 出来但没 `Add`）|完全忽略，即使你改了它也不生效|

他如果知道你改了什么：

当你从数据库查出一个实体时，ChangeTracker 会在内存中存放一个快照。当调用 SaveChangesAsync() 时，它会拿当前对象的属性和底片进行比对（Dirty Checking脏检查），发现哪个属性变了，就精准生成对应字段的 UPDATE 语句。


我们来还原在用例 Deposit.cs 执行的那几毫秒内，ChangeTracker 内存里的真实演变：

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 1000 800" width="100%" height="100%">
  <defs>
    <!-- 基础阴影 -->
    <filter id="shadow" x="-5%" y="-5%" width="110%" height="110%">
      <feDropShadow dx="0" dy="4" stdDeviation="4" flood-color="#000000" flood-opacity="0.4"/>
    </filter>

    <!-- 箭头标记 -->
    <marker id="arrow-blue" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#3b82f6" />
    </marker>
    <marker id="arrow-purple" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#8b5cf6" />
    </marker>
    <marker id="arrow-green" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#10b981" />
    </marker>

    <!-- 背景网格 -->
    <pattern id="grid" width="40" height="40" patternUnits="userSpaceOnUse">
      <path d="M 40 0 L 0 0 0 40" fill="none" stroke="#ffffff" stroke-opacity="0.03" stroke-width="1"/>
    </pattern>
  </defs>

  <!-- 全局背景 -->
  <rect width="100%" height="100%" fill="#0b1120" />
  <rect width="100%" height="100%" fill="url(#grid)" />

  <!-- ================= Participants (顶部角色) ================= -->
  <!-- Code -->
  <g transform="translate(100, 40)">
    <rect width="200" height="50" rx="8" fill="#1e293b" stroke="#3b82f6" stroke-width="2" filter="url(#shadow)"/>
    <text x="100" y="30" fill="#60a5fa" font-family="sans-serif" font-size="16" font-weight="bold" text-anchor="middle">Code (UseCase 代码)</text>
  </g>

  <!-- ChangeTracker -->
  <g transform="translate(400, 40)">
    <rect width="200" height="50" rx="8" fill="#1e293b" stroke="#8b5cf6" stroke-width="2" filter="url(#shadow)"/>
    <text x="100" y="20" fill="#a78bfa" font-family="sans-serif" font-size="15" font-weight="bold" text-anchor="middle">EF Core ChangeTracker</text>
    <text x="100" y="40" fill="#c4b5fd" font-family="sans-serif" font-size="13" text-anchor="middle">(内存中)</text>
  </g>

  <!-- SQL Server -->
  <g transform="translate(700, 40)">
    <rect width="200" height="50" rx="8" fill="#1e293b" stroke="#10b981" stroke-width="2" filter="url(#shadow)"/>
    <text x="100" y="30" fill="#34d399" font-family="sans-serif" font-size="16" font-weight="bold" text-anchor="middle">SQL Server</text>
  </g>

  <!-- ================= Lifelines (生命线) ================= -->
  <line x1="200" y1="90" x2="200" y2="750" stroke="#334155" stroke-width="2" stroke-dasharray="6,4" />
  <line x1="500" y1="90" x2="500" y2="750" stroke="#334155" stroke-width="2" stroke-dasharray="6,4" />
  <line x1="800" y1="90" x2="800" y2="750" stroke="#334155" stroke-width="2" stroke-dasharray="6,4" />

  <!-- ================= 1. UpdateAsync ================= -->
  <g transform="translate(0, 140)">
    <line x1="200" y1="0" x2="490" y2="0" stroke="#3b82f6" stroke-width="2" marker-end="url(#arrow-blue)"/>
    <circle cx="200" cy="0" r="12" fill="#3b82f6" />
    <text x="200" y="4" fill="#ffffff" font-family="sans-serif" font-size="12" font-weight="bold" text-anchor="middle">1</text>
    <text x="345" y="-10" fill="#e2e8f0" font-family="sans-serif" font-size="14" font-weight="bold" text-anchor="middle">accountRepository.UpdateAsync(account, credit)</text>
  </g>

  <!-- Note 1 -->
  <g transform="translate(520, 160)">
    <rect width="340" height="60" rx="6" fill="#2e1065" stroke="#8b5cf6" stroke-width="1" filter="url(#shadow)"/>
    <path d="M 0 10 L 15 0 L 0 10 L 0 60 Z" fill="#8b5cf6" opacity="0.5"/>
    <text x="170" y="25" fill="#ddd6fe" font-family="sans-serif" font-size="13" text-anchor="middle">实际上只是在内存中把实体的状态标记为</text>
    <text x="170" y="45" fill="#c4b5fd" font-family="sans-serif" font-size="13" font-weight="bold" text-anchor="middle">Modified / Added，并未发 SQL</text>
  </g>

  <!-- ================= 2. PublishEventAsync ================= -->
  <g transform="translate(0, 280)">
    <line x1="200" y1="0" x2="490" y2="0" stroke="#3b82f6" stroke-width="2" marker-end="url(#arrow-blue)"/>
    <circle cx="200" cy="0" r="12" fill="#3b82f6" />
    <text x="200" y="4" fill="#ffffff" font-family="sans-serif" font-size="12" font-weight="bold" text-anchor="middle">2</text>
    <text x="345" y="-10" fill="#e2e8f0" font-family="sans-serif" font-size="14" font-weight="bold" text-anchor="middle">_serviceBus.PublishEventAsync(...)</text>
  </g>

  <!-- Note 2 -->
  <g transform="translate(520, 300)">
    <rect width="340" height="60" rx="6" fill="#2e1065" stroke="#8b5cf6" stroke-width="1" filter="url(#shadow)"/>
    <path d="M 0 10 L 15 0 L 0 10 L 0 60 Z" fill="#8b5cf6" opacity="0.5"/>
    <text x="170" y="25" fill="#ddd6fe" font-family="sans-serif" font-size="13" text-anchor="middle">MassTransit 拦截发布，把消息挂载到</text>
    <text x="170" y="45" fill="#c4b5fd" font-family="sans-serif" font-size="13" font-weight="bold" text-anchor="middle">ChangeTracker 的 Outbox 表中 (Added)</text>
  </g>

  <!-- ================= 3. SaveAsync ================= -->
  <g transform="translate(0, 440)">
    <line x1="200" y1="0" x2="490" y2="0" stroke="#3b82f6" stroke-width="2" marker-end="url(#arrow-blue)"/>
    <circle cx="200" cy="0" r="12" fill="#3b82f6" />
    <text x="200" y="4" fill="#ffffff" font-family="sans-serif" font-size="12" font-weight="bold" text-anchor="middle">3</text>
    <text x="345" y="-10" fill="#e2e8f0" font-family="sans-serif" font-size="14" font-weight="bold" text-anchor="middle">unitOfWork.SaveAsync() (收银结算时刻)</text>
  </g>

  <!-- ================= 4. CT -> DB ================= -->
  <g transform="translate(0, 540)">
    <line x1="500" y1="0" x2="790" y2="0" stroke="#8b5cf6" stroke-width="2" marker-end="url(#arrow-purple)"/>
    <text x="645" y="-10" fill="#e2e8f0" font-family="sans-serif" font-size="14" font-weight="bold" text-anchor="middle">发送合并的 SQL 事务批处理 (INSERT / UPDATE)</text>
  </g>

  <!-- Note 3 -->
  <g transform="translate(660, 580)">
    <rect width="280" height="60" rx="6" fill="#064e3b" stroke="#10b981" stroke-width="1" filter="url(#shadow)"/>
    <path d="M 0 10 L 15 0 L 0 10 L 0 60 Z" fill="#10b981" opacity="0.5"/>
    <text x="140" y="25" fill="#a7f3d0" font-family="sans-serif" font-size="13" font-weight="bold" text-anchor="middle">同一个本地 ACID 事务</text>
    <text x="140" y="45" fill="#6ee7b7" font-family="sans-serif" font-size="13" text-anchor="middle">要么同时成功，要么同时回滚</text>
  </g>

</svg>


这里有一个至关重要的底层设计 —— 依赖注入的生命周期（Scoped），在 ASP.NET Core 中，每次 HTTP 请求都会创建一个独立的生命周期作用域（Scope）。仓储 AccountRepository 注入的 GenocsContext 是这个 Scope 里的实例，MassTransit Outbox 拿到的也是同一个 GenocsContext 实例，因此，它们共享了同一台 ChangeTracker。

> 如果没有这套内存共享机制：仓储在自己的连接里写业务，MQ 在自己的地方发消息，两方互不知晓，就必然退化成各自为政的“双写不一致”。

正是因为有了 ChangeTracker 这个内存中转站，我们才能在业务上保持优雅解耦（业务代码只需安心调用 Update 和Publish），而在技术底层却能神不知鬼不觉地把所有的增删改和待发消息“打包捆绑”**在同一个事务里交付数据库。

究其根本，它是专门为EF CORE以及各类主流ORM定制的协同组件，核心目的是将不可靠的网络请求降级为可靠的数据库表记录。我们来看看这个“事务表”到底长什么样，以及为什么同一个本地 ACID 事务能把它们牢牢绑在一起。

在前面的配置中，我们在 GenocsContext.cs:19-21 里加入了三行映射：


```cs
    modelBuilder.AddInboxStateEntity();
    modelBuilder.AddOutboxMessageEntity();
    modelBuilder.AddOutboxStateEntity();
```

MassTransit 会在数据库里生成三张协同工作的表，其中最核心、最关键的正是 OutboxMessage（发件箱消息表）。它本质上就是一张持久化到磁盘的“离线待发信件箱”，主要包含以下字段：


|字段名 (Field)|数据类型|核心作用|
|---|---|---|
|**`SequenceNumber`** (主键)|`bigint` / `long`|**自增主键**。后台 Worker 依据它来**排序和分批轮询**，保证消息按入队顺序发送。|
|**`MessageId`**|`Guid`|**消息唯一标识**。用于消费者端的**幂等去重**，确保同一条消息不会被重复处理。|
|**`EnqueueTime`**|`DateTime?`|**入队时间**。记录消息被挂载到 `ChangeTracker` 的时刻。|
|**`SentTime`**|`DateTime`|**发送时间**。记录消息成功投递到消息代理（如 RabbitMQ）的时间。|
|**`Headers`**|`text` / `json`|**消息头**。存储路由键、 CorrelationId 等元数据，影响消息的路由和追踪。|
|**`Body` / `Payload`**|`text` / `jsonb`|**消息体**。业务事件的**序列化内容**（通常是 JSON），是消费者真正需要的数据。|
|**`ContentType`**|`varchar(256)`|**内容类型**。标明 `Body` 的序列化格式（如 `application/json`）。|
|**`MessageType`**|`text`|**消息类型**。存储事件的**完整类型名称**（如 `Contracts.Events.DepositCompleted`），供消费者反序列化时识别。|
|**`DestinationAddress`**|`varchar(256)`|**目标地址**。指定消息将被发送到的交换机或队列地址。|

辅助表：OutboxState 与 InboxState

• OutboxState（发信锁与进度表）：记录当前发件箱有哪些批次正在被后台线程投递，防止多实例部署时两个 Pod抢着发同一批消息。
• InboxState（收件箱幂等表）：位于消费者端，记录已经处理过的消息 ID，防止下游因网络原因收到重复消息。

为什么本地 ACID 事务能让它们“要么同时成功，要么同时回滚”？这是由关系型数据库（如 SQL Server）底层的“预写日志机制”所决定的：


```sql

-- 当你在 C# 里调用 await _unitOfWork.SaveAsync() 时
-- SQL Server 底层真实执行的批处理指令如下：

BEGIN TRANSACTION; -- 开启本地 ACID 事务

	-- 动作 1：写入账户流水业务表
	INSERT INTO [Credit] ([Id], [AccountId], [Amount], [TransactionDate])
	VALUES ('c1-xxx', 'a1-xxx', 100.00, '2026-10-25');

	-- 动作 2：写入发件箱消息表
	INSERT INTO [OutboxMessage] ([MessageId], [MessageType], [Body], ...)
	VALUES ('m1-xxx', 'DepositCompleted', '{"AccountId":"a1-xxx","Amount":100}', ...);

-- 如果两条都成功，执行 COMMIT，两张表同时永久落盘
COMMIT TRANSACTION;

-- 如果任意一条失败（例如报错、断电、唯一索引冲突），执行 ROLLBACK：
-- SQL Server 事务管理器会依靠 UNDO 日志，将刚刚写入的两张表数据统统抹去！
ROLLBACK TRANSACTION;
```

底层回滚原理：

## 编排式saga

既然我们使用MassTransit，那么我们可以使用它内置的Transactional Outbox/inbox，不需要引入额外的第三方分布式事务中间件，masstransit原生内置了一个叫做MassTransitStateMachine的东西，它是一个编排式saga，配合着routing slip路由单，就是最成熟的跨微服务回滚调度器。

微服务中，回滚不是pg靠两阶段提交强锁所有数据库，而是采用saga补偿事务模式，正向一步步调各个微服务执行本地事务，一旦某一步失败了，中央调度器就会按照逆序发送补偿命令，比如说扣减库存的逆向就是释放预占库存，扣款的逆向就是原路退款，这样就可以实现最终一致性了。

我们之前使用了Inbox/outbox解决了本地的efcore写库+发消息的原子性，而 **Automatonymous 状态机（`MassTransitStateMachine<TInstance>`）** 就是全局调度多个微服务的中央调度器。

调度器需要自己维护一张状态表，通常用ef core持久化到pg或者存于redis，用`CorrelationId` 作为全局事务 ID，假设有一个下单的链路涉及到了订单->库存->支付。

那么收到了`OrderSubmitted` 事件之后状态机会切换到`ReservingInventory`，发送 `ReserveInventory` 命令。

收到 `InventoryReserved` 事件，状态机记录库存单号，切为 `ProcessingPayment`，发送 `ChargePayment` 命令。

如果收到支付服务发回的 `PaymentFailed`（或触发了状态机内置的 `Schedule` 超时定时器），状态机立刻识别当前已完成了库存扣减，自动发出补偿命令 `ReleaseInventory` 并通知订单服务 `MarkOrderFailed`，将状态推进至 `RolledBack`。

这样就可以复用我们配置好的efcore outbox + inbox，状态机每次切换状态并发出下一步命令的时候同样走本地数据库事务+outbox投递，下游微服务消费命令的时候走inbox幂等去重，防止回滚消息丢了或者重复退款的事情发生。

那么状态机有哪些状态，masstransit值内置了头尾两个生命周期锚点状态，其余所有业务状态均由你按照流程自由声明。初始是Initial，saga实例刚被首个事件激活创建时所处的起点，通过`Initially(When(OrderSubmitted)...)` 捕获入口事件并跳出 `Initial`。

调用 `.Finalize()` 后进入的终点，代表整个分布式事务生命周期彻底结束。如果配置了 `SetCompletedWhenFinalized()`，进入 `Final` 的那一刻，EF Core 仓储会自动执行 `DELETE` 把这行已完结的状态数据从活跃表中物理删除

我们可以自定义业务状态，比如说带回滚的订单到库存到支付这条链路为例，状态机类里声明`public State ... { get; private set; }` 属性即可。正向态中，`ReservingInventory`（等待锁库存） → `ProcessingPayment`（库存已锁，等待扣款） → `Completed`（全部成功）；逆向中：假如在 `ProcessingPayment` 收到 `PaymentFailed`，不能直接切到结束，而应切入 **`CompensatingInventory`（回滚库存中）** 并发送 `ReleaseInventory` 命令；等收到库存服务确认释放成功的 `InventoryReleased` 事件后，再安全切入 **`RolledBack`（已彻底回滚）** 或 `Final`。这样即使库存回滚中途失败，状态机也清楚知道还卡在回滚阶段，方便重试或告警介入。

### 复用efcore的本地事务

masstransit可以配合efcore使用，在消费单条消息的DI Scope生命周期里，InboxOutbox中间件、sagaRepository和outbox消息发送器

继续更新...

### 调度器的边界条件

saga调度需要关注幂等性和空回滚、防悬挂这几个情况出现。

由于网络抖动会导致补充命令重发，这个时候库存服务的`ReleaseInventory` 必须能承受重复调用而不把库存加多，masstransit inbox可以帮我们挡掉大部分的重复消费，但是业务状态字段依然要状态流转校验，这个就叫做幂等性。

正向的`ReserveInventory` 因为网络延迟还没到库存服务，调度器就因为超时发出了回滚命令 `ReleaseInventory`。此时库存服务查不到扣减记录，必须优雅返回成功并记录一条“已作废”，而不是抛 404 异常卡死回滚流。

空回滚执行了之后如果这个时候正向`ReserveInventory` 终于送达了，那么我们就应该让正向请求查查有没有已回滚，有的话直接拒接执行。

### 路由单工具

masstransit有一个回滚的工具，就是我们刚才提到的routing slip，也叫做路由单。如果我们的跨服务调用链路是动态组装的，比如不同商品类型的审核与履约步骤不同。那么写死一个巨大的状态机就比较僵硬，泛用性不足，MassTransit 还提供了 **Courier** 。

我们只需要给每个微服务实现统一的`IActivity<TArguments, TLog>` 接口，强制你成对编写两个方法：**正向执行的 `Execute()`** 和 **逆向回滚的 `Compensate()`**，`Execute()` 成功后，可以把回滚所需的快照凭证（如预扣流水号）写入 `TLog`。一旦后续任意第 N 个节点的 `Execute()` 抛异常返回 `Faulted`，MassTransit 会像剥洋葱一样，沿着路由单从第 N-1 个节点往回逐个自动调用 `Compensate(TLog)`，连中央状态机的 `if-else` 分支都不用手写。

但是在实际的生产项目里，似乎状态机saga的使用广泛程度要远高于路由单模式，占到了90%的情况，有些系统里面也会混用，比如状态机作为总调度，路由单当特定的执行器的上下级关系。

因为masstransitStateMachine每笔业务流转在pg里都有真实的数据库记录，它天生擅长处理异步挂起等待任务、复杂条件分支和多事件并行汇聚。做运维排障的时候，直接SELECT 状态表就可以看清全网有多少笔订单卡在什么环节，非常方便。

路由单是去中心化消息包带状态，因此一旦某一步报错，就顺着消息里记录的历史路径倒着调 `Compensate()`。它只适合**纯线性、短平快、一气呵成**的批处理流水线（比如：校验参数 → 扣配额 → 写对象存储 → 发通知），无法优雅地停在半路等一个外部第三方异步回调，且出问题时很难从数据库全局统计当前在途任务。

如果状态机的一个阶段需要动态串联四个微服务做资源初始化，那么通常是由`MassTransitStateMachine` 组装并发射一张 Routing Slip，自己切入`ExecutingSlip` 状态等待；等路由单全部跑完或自动回滚完，状态机收到 `RoutingSlipCompleted` 或 `RoutingSlipFaulted` 事件后再继续推进主流程。

### 其他跨服务调度框架

Temporal.io是现代化云原生持久化的框架，Temporal 允许你直接用普通的 C# `async/await` 和 `try / catch` 顺序写代码——`catch` 块里直接按栈逆序写补偿逻辑即可。哪怕调度器容器在执行到第 3 行时断电重启，它也能通过事件溯源重放历史，精准恢复到断点继续跑或继续回滚。


## 实践

比如我们来定义个状态机和一个实例，普遍的开发也是这样做的：

这个是状态机的实例数据，它非常重要，因为它负责保存一个业务流程在执行过程中的状态和上下文数据。

```cs
public class NewsletterEmailSagaData : SagaStateMachineInstance 
{
    public Guid CorrelationId { get; set; }
    public string CurrentState { get; set; } = string.Empty;
    public Guid ProductId { get; set; }
    public string Name { get; set; } = string.Empty;
    public string Description { get; set; } = string.Empty;
    public bool NotifyProductSend { get; set; }
}
```

里面最重要的是`CorrelationId`：标识这是哪个 Saga 实例还有`CurrentState`：标识这个业务流程执行到了哪里。其他的都是我们自己可以定义的属性。

saga实例和事件的关系是，如果收到了事件，masstransit就可以创建一个saga实例，我这里写了两个事件，一个是产品创建成功的事件，一个是邮件发送成功的事件：

```cs
public record EmailSent(
    Guid Id,
    string Name,
    string Description
);

public record ProductCreated(
    Guid ProductId,
    string Name,
    string Description,
    DateTime CreatedAt
);
```

masstransit会根据事件里的关联ID来找到这个Saga实例，然后修改它的数据，最终状态可能变成completed。

我们这里是一个产品对应一个Newsletter通知流程，那么可以让它们使用相同的Guid：

```cs
Event(() => ProductCreated, e =>
    e.CorrelateById(m => m.Message.ProductId));
```

这样新建 Saga 时，`CorrelationId` 就使用 `ProductId`，后续 `EmailSent` 也必须携带相同的产品 ID：

```cs
Event(() => EmailSent, e =>
    e.CorrelateById(m => m.Message.ProductId));
```

如果未来一个产品可能触发多个独立的通知流程，那么就应该考虑使用独立的 `CorrelationId`，而不是直接使用 `ProductId`。


```cs
public class NewsletterNotifySaga : MassTransitStateMachine<NewsletterEmailSagaData>
{
    public required State Notifying { get; set; }
    public Event<ProductCreated> ProductCreated { get; set; }
    public Event<EmailSent> EmailSent { get; set; }

    public NewsletterNotifySaga()
    {
        InstanceState(x => x.CurrentState);
        Event(() => ProductCreated, e =>
            e.CorrelateById(m => m.Message.ProductId));
        Event(() => EmailSent, e =>
            e.CorrelateById(m => m.Message.ProductId));
    }
}
```


Event用于定义状态机能够接受哪些事件， `CorrelateById()`：定义如何根据事件找到对应的 Saga 实例，`Initially()`、`During()`：定义收到事件后执行什么操作，以及如何转换状态。

到这一步我们就算配置好了，我们会从初始状态开始，通过Initially方法来设置，When方法是这个状态下发生什么事件就会去执行什么，During也是，处于什么状态下，如果发生了什么事件，就去处理什么，完整代码如下：

```cs
public class NewsletterNotifySaga : MassTransitStateMachine<NewsletterEmailSagaData>
{
    public required State SendingEmailState { get; set; }
    public required State EmailSentState { get; set; }
    public Event<ProductCreated> ProductCreated { get; set; }
    public Event<EmailSent> EmailSent { get; set; }

    public NewsletterNotifySaga()
    {
        InstanceState(x => x.CurrentState);
        Event(() => ProductCreated, e =>
            e.CorrelateById(m => m.Message.ProductId));
        Event(() => EmailSent, e =>
            e.CorrelateById(m => m.Message.ProductId));
        Initially(
            When(ProductCreated)
            .Then(context =>
            {
                context.Saga.ProductId = context.Message.ProductId;
                context.Saga.Name = context.Message.Name;
                context.Saga.Description = context.Message.Description;
            })
            .TransitionTo(SendingEmailState)
            .Publish(context =>
                new CreateEmail(context.Message.ProductId, context.Message.Name, context.Message.Description)
            ));

        During(SendingEmailState, When(EmailSent)
            .Then(context =>
                context.Saga.NotifyProductSend = true)
            .TransitionTo(EmailSentState)
            );
    }
}

```

我们使用Publish命令发布事件，通知订阅者某件事情已经发生了，如果你要发送的是命令，要求某个Handler执行操作的话，就需要使用Send。

这样我们就可以将事件推送到rabbitmq中了，不过当你需要进程内事件的时候呢，因为这些事件都在mq里传播，因此如果你又要使用进程内通信的话就不方便了，我们需要配置mediator，然后在里面配置saga状态机，这样saga既可以发布mq的事件，又可以发布进程内的事件了。

```cs
public static class MassTransitExtension
{
    static public IServiceCollection AddMassTransitService(this IServiceCollection services, IConfiguration configuration)
    {
        services.AddMassTransit(x =>
        {
            //x.AddSagaStateMachine<NewsletterNotifySaga, NewsletterEmailSagaData>()
            //    .EntityFrameworkRepository(r =>
            //    {
            //        r.ExistingDbContext<AppDbContext>();
            //        r.UsePostgres();
            //    });
            x.UsingRabbitMq((context, cfg) =>
            {
                var connectionString =
                    configuration.GetConnectionString("messaging");

                if (string.IsNullOrWhiteSpace(connectionString))
                {
                    throw new InvalidOperationException(
                        $"RabbitMQ connection string is missing. " +
                        $"Environment variable exists: " +
                        $"{!string.IsNullOrWhiteSpace(
                            Environment.GetEnvironmentVariable("ConnectionStrings__messaging"))}");
                }

                cfg.Host(new Uri(connectionString));
                cfg.ConfigureEndpoints(context);
            });
        });

        services.AddMediator(cfg =>
        {
            cfg.AddConsumer<NotifyProductCreatedHandler>();
            cfg.AddConsumer<CreateProductHandler>();
            cfg.AddSagaStateMachine<NewsletterNotifySaga, NewsletterEmailSagaData>()
                .EntityFrameworkRepository(r =>
                {
                    r.ExistingDbContext<AppDbContext>();
                    r.UsePostgres();
                }
                );
        });
        return services;
    }
}

```


这个Mediator的使用体验我个人认为有点不如MediatR方便，MediatR会通过程序集自己去扫描handler并注册，这里我们还要手动去注册handler，当然我们可以自己去手动实现一个扫描程序集，然后注册进去的方法。

基于此改进，我们可以写一个反射查找，然后注册进去：

```cs
    private static bool IsConsumerInterface(Type interfaceType)
    {
        return interfaceType.IsGenericType &&
               interfaceType.GetGenericTypeDefinition() == typeof(IConsumer<>);
    }

    public static void AddMediatorConsumers(this IMediatorRegistrationConfigurator configurator, params Assembly[] assemblies)
    {
        var consumerType = assemblies
            .SelectMany(assemblies => assemblies.GetTypes())
            .Where(type => type is { IsClass: true, IsAbstract: false }
                && type.GetInterfaces().Any(IsConsumerInterface)
            )
            .Distinct()
            .ToArray();
        foreach (var consumer in consumerType)
        {
            configurator.AddConsumer(consumer);
        }
    }
```

调用的时候我们直接把本类所在的程序集传入进去就行了，这样配置好了以后你就可以在saga里发布进程内消息了，不需要自己注入`IScopedMediator`，我们如果要在saga向外部微服务发送消息，可以注入rabbitmq bus对应的发布接口。


## 推荐书籍：

《微服务架构设计模式》（Chris Richardson 著）第 4 章《使用 Saga 管理事务》**：全书最精华的一章，把“协同式 vs 编排式 Saga”、“语义锁”以及应对并发更新异常的对策讲得极其透彻。


在 YouTube 搜 `MassTransit Saga State Machine` 或直接拉取 GitHub 上的 `MassTransit/Sample-Twitch` / `Sample-Library`，里面完整演示了 `MassTransitStateMachine` + `EntityFrameworkRepository` + `Transactional Outbox` + `Courier Routing Slip`