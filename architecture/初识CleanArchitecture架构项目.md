---
date: 2026-08-05
tags:
  - architecture
  - csharp
  - genocs
  - ddd
---
这个是个非常典型的基于DDD实践的，和基于Genocs工具包实现的一个模板文件，我们·可以基于命令简单的通过模板创建项目和部署。

我们可以在终端里用dotnet命令安装模板和指定版本：

```shell
# .NET 7/8/9/10+ 推荐命令
dotnet new install Genocs.CleanArchitecture.Template

# 如需安装指定版本（可选）
dotnet new install Genocs.CleanArchitecture.Template::9.0.0
```

>注：在较旧的 .NET 6 及更早版本中，安装语法为 dotnet new -i Genocs.CleanArchitecture.Template。

模板对应的短名称（Short Name）为 gnx-cleanarchitecture，可以通过 --help查看支持的全部配置项（如支持的数据库、消息代理选型等）：
`dotnet new gnx-cleanarchitecture --help`

```shell
[17:27:11] PS C:\Users\20742>  dotnet new gnx-cleanarchitecture --help
Genocs: Clean Architecture Template (C#)
Author: Nocco Giovanni Emanuele

Usage:
  dotnet new gnx-cleanarchitecture [options] [template options]

Options:
  -n, --name <name>       The name for the output being created. If no name is specified, the name of the output
                          directory is used.
  -o, --output <output>   Location to place the generated output.
  --dry-run               Displays a summary of what would happen if the given command line were run if it would result
                          in a template creation. [default: False]
  --force                 Forces content to be generated even if it would change existing files. [default: False]
  --no-update-check       Disables checking for the template package updates when instantiating a template. [default:
                          False]
  --project <project>     The project that should be used for context evaluation.
  -lang, --language <C#>  Specifies the template language to instantiate.
  --type <solution>       Specifies the template type to instantiate.

Template options:
  -uc, --use-cases <basic|full|readonly>   Type: choice
                                             full      Full Wallet Account Balance Management (Read/Write)
                                             basic     Allow Customer to Register and to Get Customer Details
                                           (Read/Write)
                                             readonly  Read Customer and Account Details (Read Only)
                                           Default: full
  -sb, --service-bus <choice>              Type: choice
                                             azureservicebus  Azure Service Bus (AzureServiceBus)
                                             masstransit      Mass Transit Service Bus (MassTransit)
                                             nservicebus      Particular Software Service Bus (NServiceBus)
                                             rebus            Rebus Service Bus (Rebus)
                                           Default: rebus
  -da, --database <mongodb|sqlserver|...>  Type: choice
                                             mongodb    MongoDB Database (MongoDb)
                                             sqlserver  SQL Server Database (SQLServer)
                                             inmemory   In Memory database for Debug (InMemory)
                                           Default: sqlserver

[17:27:14] PS C:\Users\20742>
```

创建：`dotnet new gnx-cleanarchitecture --name "dddotNet" -o "./dddotNet"`

更新： `dotnet new install Genocs.CleanArchitecture.Template`

## 工具链

Genocs 系列是该模板的基石组件，提供了统一的基础设施抽象、规范与工具链：

DDD的基础模型包括实体、聚合根、通用拓展与类型定义、结构化日志封装与统一配置比如Domain、Contracts文件夹里的内容由Gencos.Common工具提供。结构化日志封装与统一配置比如webapi、Worker由Gencos.Logging提供。OpenTelemetry / APM 链路追踪与指标收集由Genocs.Telemetry提供。OpenAPI / Swagger 接口文档自动化配置与增强比如webapi里的内容由Genocs.WebApi.OpenApi提供。

## 架构分层

数据持久化实现的位置默认在Infrastructure.csproj与 Host.csproj中：

• Microsoft.EntityFrameworkCore.SqlServer (10.0.8)：SQL Server 数据库持久化提供程序。
• Microsoft.EntityFrameworkCore.InMemory (10.0.8)：内存数据库（常用于本地快速开发或调试）。
• Microsoft.EntityFrameworkCore.Tools / Design (10.0.8)：EF Core 迁移与脚手架工具。

传统的架构中，持久层是直接把实体类与数据库表绑定，一个模型打到底，但在本项目中，持久化相关的职责被精细地切分在不同层级中：


```mermaid
flowchart TD
        subgraph Domain["1. 领域层 (Domain)"]
            IAccount["纯领域模型/接口<br/>(IAccount, Customer, Money)"]
            IEntityFactory["领域工厂抽象<br/>(IEntityFactory)"]
        end

        subgraph Application["2. 应用层 (Application)"]
            UseCase["业务用例 (Deposit, Register)"]
            IRepo["仓储契约 (IAccountRepository)"]
            IUoW["工作单元契约 (IUnitOfWork)"]
        end

        subgraph Infrastructure["3. 基础设施层 (Infrastructure.PersistenceLayer.SQLServer)"]
            Context["EF Core 上下文 (GenocsContext)"]
            PersistEntity["持久化专用实体 (Account 继承自 Domain.Account)"]
            EntityFactory["实体工厂实现 (EntityFactory)"]
            RepoImpl["仓储实现 (AccountRepository)"]
            UoWImpl["工作单元实现 (UnitOfWork)"]
            Migrations["数据库迁移 (Migrations)"]
        end

        UseCase --> IRepo
        UseCase --> IUoW
        UseCase --> IEntityFactory
        RepoImpl -.->|实现| IRepo
        UoWImpl -.->|实现| IUoW
        EntityFactory -.->|实现| IEntityFactory
        PersistEntity -.->|派生| IAccount
        RepoImpl --> Context
        UoWImpl --> Context
```


### 领域模型层

契约定义层（位于 Domain 与 Application）,比如Application/Repositories的IAccountRespository.cs中的：

```cs
using dddotNet.Domain.Accounts;

namespace dddotNet.Application.Repositories;

public interface IAccountRepository
{
    Task<IAccount?> GetAsync(Guid id, CancellationToken cancellationToken = default);
    Task AddAsync(IAccount account, ICredit credit, CancellationToken cancellationToken = default);
    Task UpdateAsync(IAccount account, ICredit credit, CancellationToken cancellationToken = default);
    Task UpdateAsync(IAccount account, IDebit debit, CancellationToken cancellationToken = default);
    Task DeleteAsync(IAccount account, CancellationToken cancellationToken = default);
}
```

模板里订阅契约的时候直接使用的是Domain层的领域模型定义的聚合根里的内容，IEntityFactory 负责解耦领域对象与持久化对象的实例化。·

```cs
public interface IAccount : IAggregateRoot<Guid>
{
    ICredit Deposit(IEntityFactory entityFactory, PositiveMoney amountToDeposit);
    IDebit? Withdraw(IEntityFactory entityFactory, PositiveMoney amountToWithdraw);
    bool IsClosingAllowed();
    Money GetCurrentBalance();
}
...
/// <summary>
/// The <c>IEntityFactory</c> interface defines a contract for creating instances of domain entities such as customers, accounts, credits, and debits.
/// It provides methods for instantiating these entities with the necessary parameters,
/// ensuring that the creation logic is centralized and consistent across the application.
/// </summary>
public interface IEntityFactory
{
    ICustomer NewCustomer(SSN ssn, Name name);
    IAccount NewAccount(ICustomer customer);
    ICredit NewCredit(IAccount account, PositiveMoney amountToDeposit, DateTime transactionDate);
    IDebit NewDebit(IAccount account, PositiveMoney amountToWithdraw, DateTime transactionDate);
}
```

DDD里，领域层Domain必须保存纯粹，不能依赖具体的ORM，如Entity Frame Core、Dapper 或者特定数据库生成的类，如果 `IAccount` 内部直接 `new Credit(...)` 或 `new Debit(...)`，甚至直接调用持久化框架的工厂/上下文，那么`Account` 必须知道 `Credit` / `Debit` 的具体实现类（而不是接口 `ICredit` / `IDebit`）。如果 `Credit` 对象的创建过程需要依赖数据库主键生成器、全局 ID 生成算法、或是特定 ORM 跟踪的上下文，`Account` 实体就会强耦合这些底层基础设施。

引入 `IEntityFactory` 后的解耦：`Account` 只需要调用 `entityFactory.CreateCredit(...)` 或类似的抽象方法。`Account` 只依赖 `IEntityFactory` 抽象接口，而不关心 `Credit` 底层到底是用什么具体类初始化的、主键是如何生成的、或者是如何注册到 ORM 跟踪上下文里的。

这个实体工厂接口的实现在SQLServer的持久层，就是这样，领域层的 Domain.Accounts.Account 只有业务逻辑和流水集合，它根本不知道关系型数据库里需要 CustomerId这个外键字段。

### 持久化的实现层

```cs
namespace dddotNet.Infrastructure.PersistenceLayer.SQLServer;

public sealed class EntityFactory : IEntityFactory
{
    public IAccount NewAccount(ICustomer customer)
    {
        var account = new Account(customer);
        return account;
    }

    public ICredit NewCredit(IAccount account, PositiveMoney amountToDeposit, DateTime transactionDate)
        => new Credit(account, amountToDeposit, transactionDate);

    public ICustomer NewCustomer(SSN ssn, Name name)
        => new Customer(ssn, name);

    public IDebit NewDebit(IAccount account, PositiveMoney amountToWithdraw, DateTime transactionDate)
        => new Debit(account, amountToWithdraw, transactionDate);
}
```

最后，webapi中，在SQLServerInfrastructureExtensions.cs中，将SQLServer的工厂实现绑定给领域接口：

```cs
namespace dddotNet.WebApi.Extensions.SQLServer;

public static class SQLServerInfrastructureExtensions
{
    private static readonly Lock MigrationLock = new();

    public static IServiceCollection AddSQLServerPersistence(this IServiceCollection services, IConfiguration configuration)
    {
        services.AddScoped<IEntityFactory, EntityFactory>();

        services.AddDbContext<GenocsContext>(options =>
            options
                .ConfigureWarnings(w => w.Ignore(RelationalEventId.PendingModelChangesWarning))
                .UseSqlServer(
                    configuration.GetConnectionString("DefaultConnection"),
                    b =>
                    {
                        b.MigrationsAssembly("dddotNet.Infrastructure");
                        b.EnableRetryOnFailure();
                    }));

        services.AddScoped<IUnitOfWork, UnitOfWork>();

        services.AddScoped<IAccountRepository, AccountRepository>();
        services.AddScoped<ICustomerRepository, CustomerRepository>();

        return services;
    }

    public static IServiceProvider UseSqlServerPersistence(this IServiceProvider serviceProvider)
    {
        const int maxAttempts = 5;

        for (int attempt = 1; attempt <= maxAttempts; attempt++)
        {
            try
            {
                using var scope = serviceProvider.CreateScope();
                var context = scope.ServiceProvider.GetService<GenocsContext>();

                if (context?.Database.IsSqlServer() != true)
                {
                    return serviceProvider;
                }

                lock (MigrationLock)
                {
                    // CreateExecutionStrategy retries transient failures while creating/migrating a fresh database.
                    var strategy = context.Database.CreateExecutionStrategy();
                    strategy.Execute(() => context.Database.Migrate());
                }

                if (context.Database.CanConnect())
                {
                    return serviceProvider;
                }
            }
            catch (SqlException) when (attempt < maxAttempts)
            {
                Thread.Sleep(TimeSpan.FromMilliseconds(250 * attempt));
            }
        }

        // Last attempt should surface an actionable exception instead of silently starting with a broken DB.
        using (var scope = serviceProvider.CreateScope())
        {
            var context = scope.ServiceProvider.GetService<GenocsContext>();
            if (context?.Database.IsSqlServer() == true)
            {
                lock (MigrationLock)
                {
                    var strategy = context.Database.CreateExecutionStrategy();
                    strategy.Execute(() => context.Database.Migrate());
                }
            }
        }

        return serviceProvider;
    }
}
```

AddSQLServerPersistence方法中，应用层的接口都被绑定到了具体的实现上：

- `IEntityFactory` -> `EntityFactory`
    
- `IUnitOfWork` -> `UnitOfWork` （正是我们刚才讨论的**事务边界**实现）
    
- `IAccountRepository` / `ICustomerRepository` -> 具体实现。

从结果来说，应用层和领域层完全不知道ef core、sql server的存在，如果想更换为postgresql，只需要更新一个扩展方法，应用层完全不用修改，这个就叫做依赖倒置原则。

对于数据库的重试策略这里下了功夫：

```cs
services.AddDbContext<GenocsContext>(options =>
	options
		.ConfigureWarnings(w => w.Ignore(RelationalEventId.PendingModelChangesWarning))
		.UseSqlServer(
			configuration.GetConnectionString("DefaultConnection"),
			b =>
			{
				b.MigrationsAssembly("dddotNet.Infrastructure");
				b.EnableRetryOnFailure();
			}));
```

 **`b.EnableRetryOnFailure()`**：告诉 EF Core，如果是由于网络抖动等瞬时故障导致的 SQL 异常，自动重试。**`context.Database.CreateExecutionStrategy()`**：这是 EF Core 官方推荐的做法。**当你开启了 `EnableRetryOnFailure` 后，如果手动调用 `Migrate()` 或者手动开启事务，必须用 `ExecutionStrategy.Execute(...)` 包起来**。否则 EF Core 会抛出异常。这段代码正确处理了这个细节，确保在连接不稳定时迁移也能安全重试。
UseSqlServerPersistence里有一个防并发启动的锁机制Lock，这里用了`MigrationLock` 锁。 这通常是为了防止在**多线程启动**（比如同一台机器上跑多个 Worker 或者服务器）时，多个实例同时尝试执行 `Database.Migrate()`，导致数据库表被锁死或迁移报错。
`IUnitOfWork` 注册为 `Scoped`（每次 HTTP 请求一个）。**在一次 HTTP 请求（即一个 Use Case 用例）中，共享同一个 UnitOfWork 和 DbContext**，从而保证事务的一致性。但要注意：如果在 `UnitOfWork` 内部去调用第三方 API（比如发邮件、调微信支付），第三方 API 成功但 `SaveChangesAsync` 失败了，就会导致数据不一致。

> **建议**：在生产环境，最好不要让应用程序在启动时自动执行迁移。更规范的做法是：把数据库迁移作为一个**独立的 CI/CD 步骤**（例如使用 EF Core CLI 的 `dotnet ef database update`），在应用部署前由流水线执行一次。

不过当前项目在 Account.Deposit(IEntityFactory, ...) 中传入工厂的设计确实“偏重”了，具有明显的过度工程（Over-engineering）痕迹。在现代 .NET / DDD 工程实践中，更推荐【由聚合根内部直接创建 Credit】配合 EF Core 的高级映射能力。Genocs 模板遵循 Uncle Bob 的严苛 Clean Architecture流派）之所以这么设计，是为了追求极致的“持久化无知”

1. 它不想让 Domain 层的 Credit 包含任何数据库外键字段（如 AccountId）。
2. 它在基础设施层写了一个 SQLServer.Credit 继承自 Domain.Credit，专门加上了 AccountId。
3. 为了让领域层不知不觉用上这个带外键的派生类，它不得不发明了 IEntityFactory，并通过 Deposit 参数暗度陈仓地传进去。

而且领域模型无需声明 public Guid AccountId { get; }，EF Core 的 Fluent API 会自动在数据库层维护外键：

```cs
// 在 GenocsContext.cs 中配置：
modelBuilder.Entity<Account>(b =>
{
	b.ToTable("Account");
	// 配置一对多关系，外键定义为影子属性 "AccountId"，完全不污染领域模型
	b.HasMany(a => a.Credits)
	 .WithOne()
	 .HasForeignKey("AccountId");
});
```

不过用于学习的话，已经足够了。


领域层只与接口交互，具体实现被彻底隐藏在 `IEntityFactory` 的具体实现类（如 `SqlEntityFactory`）之后。在真实的持久化/数据库体系中，新创建一条流水（`Credit` 或 `Debit`）往往需要遵循严格的系统规约：

- **ID 生成策略**：例如使用雪花算法（Snowflake）、UUIDv7 或者是数据库自增序列。如果直接在 `Account` 内部 `new Credit(...)`，`Account` 就不得不注入或调用 ID 生成器。
    
- **审计字段绑定**：交易时间（`transactionDate`）、创建人、版本号（用于乐观锁 `RowVersion`）。

好了，拉回来。最上方的代码里定义的是仓储契约，定义了对业务聚合的读写规范。在ddd分层里，设计用于返回聚合根接口，保证了上层代码（如应用服务 Application Service）操作的都是抽象的领域契约，而不是具体的贫血实体或数据库实体。

方法签名里不仅有 `IAccount`，还带了 `ICredit` 或 `IDebit`，例如： `Task UpdateAsync(IAccount account, ICredit credit, ...)` 这说明该仓储不仅仅是更新一个账户的主表，它在底层设计上：**“存款/取款这种领域行为发生后，不仅要更新账户的余额，还要将新产生的交易流水（Credit/Debit）一并原子化地持久化到数据库中”**。这保证了领域操作的完整性和事务的一致性。

持久化实现层（位于 Infrastructure.PersistenceLayer.SQLServer），我们可以打开 `dddotNet.Infrastructure.PersistenceLayer.SQLServer;`内的Accout看看里面的实现：


```cs
using dddotNet.Domain.Accounts;
using dddotNet.Domain.Customers;

namespace dddotNet.Infrastructure.PersistenceLayer.SQLServer;

public class Account : Domain.Accounts.Account
{
    public Guid CustomerId { get; protected set; }

    protected Account()
    {
    }

    public Account(ICustomer customer)
    {
        Id = Guid.NewGuid();
        CustomerId = customer.Id;
    }

    public void Load(IList<Credit> credits, IList<Debit> debits)
    {
        Credits = new CreditsCollection();
        Credits.Add(credits);

        Debits = new DebitsCollection();
        Debits.Add(debits);
    }
}
```

基础设施里定义了一个 `Account` 类，它**继承并扩展**了领域层的 `Domain.Accounts.Account`，领域层模型包含了核心的业务规则，比如Deposit存款、Withdraw取款，IsClosingAllowed校验、GetCurrentBalance计算余额。

```cs
using dddotNet.Domain.ValueObjects;
using Genocs.Common.CQRS.Events;

namespace dddotNet.Domain.Accounts;

public class Account : IAccount
{
    public Guid Id { get; protected set; }
    public CreditsCollection Credits { get; protected set; }
    public DebitsCollection Debits { get; protected set; }

    public IReadOnlyCollection<IEvent> DomainEvents => throw new NotImplementedException();

    protected Account()
    {
        Credits = new CreditsCollection();
        Debits = new DebitsCollection();
    }

    public ICredit Deposit(IEntityFactory entityFactory, PositiveMoney amountToDeposit)
    {
        var credit = entityFactory.NewCredit(this, amountToDeposit, DateTime.UtcNow);
        Credits.Add(credit);
        return credit;
    }

    public IDebit? Withdraw(IEntityFactory entityFactory, PositiveMoney amountToWithdraw)
    {
        if (GetCurrentBalance().LessThan(amountToWithdraw))
            return null;

        var debit = entityFactory.NewDebit(this, amountToWithdraw, DateTime.UtcNow);
        Debits.Add(debit);
        return debit;
    }

    public bool IsClosingAllowed()
    {
        return GetCurrentBalance().IsZero();
    }

    public Money GetCurrentBalance()
    {
        var totalCredits = Credits
            .GetTotal();

        var totalDebits = Debits
            .GetTotal();

        return totalCredits
            .Subtract(totalDebits);
    }

    public bool IsTransient()
    {
        throw new NotImplementedException();
    }

    public void AddDomainEvent(IEvent @event)
    {
        throw new NotImplementedException();
    }

    public void ClearDomainEvents()
    {
        throw new NotImplementedException();
    }
}
```

这里领域模型（Domain Model）里的方法直接接收并调用了 `IEntityFactory`，确实产生了一种**依赖**。但从标准的企业级 DDD 架构来看，这种设计是**刻意且合理**的。我们知道核心原则：

>领域层不能依赖基础设施层的具体实现（如 EF Core、数据库驱动、具体的 SQL 拼接等）。

回到刚才的基础设施，持久化层，这里定义了一个继承自领域模型的持久化子类，充当领域模型和关系型数据库的适配器。**领域模型中的 `Account`**：为了保证业务逻辑的纯粹性，它可能只需要知道自己属于哪个客户（比如直接持有 `ICustomer` 引用，或者在创建时传入客户对象）。它不关心数据库里有没有 `CustomerId` 这个外键字段。**数据库/SQL Server 中的 `Account` 表**：关系型数据库是一个扁平的表格，它必须要有一个物理字段（`CustomerId` 列）来跟 `Customer` 表建立外键关联。

通过子类扩展出了 `public Guid CustomerId { get; protected set; }`，让这个类能够完美映射到 SQL Server 的表结构上。

### 高频读和写操作分离

现代企业级高并发架构中，CQRS常常使用，读写分离。 在很多严肃的商业系统中，“用 EF Core 处理写，用 Dapper 处理读” 已经成为公认的最佳实践。

就用这个样板里的用户充值存款操作里来看：

入口是AccountsController.cs，控制器非常薄，只负责接收 HTTP 请求参数，构造 DepositInput 并传递给用例。


```cs
[ApiVersion("1.0")]
[Route("api/v1/[controller]")]
[ApiController]
public sealed class AccountsController(IUseCase<DepositInput> depositUseCase, DepositPresenter presenter) : ControllerBase
{
    private readonly IUseCase<DepositInput> _depositUseCase = depositUseCase;
    private readonly DepositPresenter _presenter = presenter;

    /// <summary>
    /// Deposit on an account.
    /// </summary>
    /// <response code="200">The updated balance.</response>
    /// <response code="400">Bad request.</response>
    /// <response code="500">Error.</response>
    /// <param name="request">The request to deposit.</param>
    /// <returns>The updated balance.</returns>
    [HttpPatch("Deposit")]
    [ProducesResponseType(StatusCodes.Status200OK, Type = typeof(DepositResponse))]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    [ProducesResponseType(StatusCodes.Status500InternalServerError)]
    public async Task<IActionResult?> DepositAsync([FromBody][Required] DepositRequest request)
    {
        var depositInput = new DepositInput(request.AccountId, new PositiveMoney(request.Amount));

        await _depositUseCase.ExecuteAsync(depositInput);
        return _presenter.ViewModel;
    }
}
```

然后我们可以打开应用层的UseCase业务编排，看看Deposit.cs ：

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

那么怎么避免这种情况，那么应该使用**事务性发件箱模式**


```mermaid
flowchart TD
    subgraph App["应用层 / UseCase"]
        Business["1. 业务逻辑 (如存款)"]
        CreateOutbox["2. 创建待发事件消息"]
    end

    subgraph Database["同一个 SQL Server 数据库事务"]
        AccountTable[("Account / Credit 表")]
        OutboxTable[("OutboxMessages 表 (发件箱)")]
    end

    subgraph BackgroundWorker["后台发布进程 (Outbox Processor)"]
        Poller["定期拉取未发送消息 / CDC"]
        MarkSent["标记消息为已发送"]
    end

    subgraph Broker["消息代理 (RabbitMQ)"]
        RabbitMQ["RabbitMQ 交换机/队列"]
    end

    Business -->|更新业务数据| AccountTable
    CreateOutbox -->|插入待发消息| OutboxTable

    %% 这里加上了双引号，修复了解析错误
    AccountTable -.->|"【同一个 SaveAsync() 本地事务】<br/>要么同时成功，要么同时回滚！"| OutboxTable

    Poller -->|读取未发送消息| OutboxTable
    Poller -->|投递事件| RabbitMQ
    RabbitMQ -->|ACK 确认| MarkSent
    MarkSent -->|更新状态为 Processed| OutboxTable
```

这个在其他笔记里看，请查看[[MassTransitService收件箱模式]]。

我们继续来扯读写分离。读写分离在实际软件工程中，通常分为两个阶段演进：

1. 第一阶段：应用代码级读写分离（我们刚刚在项目中落地的模式）
2. 第二阶段：物理数据库级主从分离与异构投影（架构的高并发终局）

很多团队以为“读写分离”必须先买两台数据库服务器。其实不然，在同一个物理数据库上把代码层的读写模型彻底分开，就能解决80% 的性能瓶颈。


```mermaid
flowchart TD
    %% 顶部的统一数据库
    DB[("同一个 SQL Server 数据库")]
    
    %% 使用不可见连线把数据库放在上方居中
    WritePath ~~~ DB
    DB ~~~ ReadPath

    %% 左侧：写路径 (Command)
    subgraph WritePath["【写路径 (Command)】"]
        direction TB
        W_Tech["技术栈：EF Core + Outbox"]
        W_Model["模型：富领域聚合根 (Account)"]
        W_Flow["流程：查聚合 -> 业务规则校验<br/>-> 产生流水 -> Commit"]
        
        W_Tech --- W_Model --- W_Flow
    end

    %% 右侧：读路径 (Query)
    subgraph ReadPath["【读路径 (Query)】"]
        direction TB
        R_Tech["技术栈：Dapper 原生 SQL"]
        R_Model["模型：扁平只读 DTO"]
        R_Flow["流程：单次复合查询 -> 格式化<br/>-> 立即返回"]
        
        R_Tech --- R_Model --- R_Flow
    end

    %% 连接线与注释
    WritePath -->|"写事务：锁行、强校验"| DB
    DB -->|"读会话：WITH NOLOCK、极速出"| ReadPath

    %% 样式美化（可选，让图表看起来更专业）
    style DB fill:#f9f9f9,stroke:#333,stroke-width:2px
    style WritePath fill:#fff3e0,stroke:#ff9800,stroke-width:2px
    style ReadPath fill:#e3f2fd,stroke:#2196f3,stroke-width:2px
```

为什么 Dapper 能做到比纯 EF Core 快数倍，AccountQueries.cs:22-42中的轮询：


```cs
/// <summary>
/// 基于 Dapper 的高性能账户只读查询实现（CQRS Read Side）.
/// 绕过领域聚合与实体追踪，直接执行原生 SQL 查询出只读 DTO.
/// </summary>
public sealed class AccountQueries(GenocsContext context) : IAccountQueries
{
    private readonly GenocsContext _context = context ?? throw new ArgumentNullException(nameof(context));

    public async Task<AccountDetailsDto?> GetAccountDetailsAsync(Guid accountId, CancellationToken cancellationToken = default)
    {
        // 关系型数据库环境（如 SQL Server）：使用 Dapper 执行原生 SQL 高速读取
        if (_context.Database.IsRelational())
        {
            const string sql = @"
                SELECT a.Id AS AccountId
                FROM Account a WITH (NOLOCK)
                WHERE a.Id = @AccountId;

                SELECT 'Credit' AS Description, Amount, TransactionDate
                FROM Credit WITH (NOLOCK)
                WHERE AccountId = @AccountId

                UNION ALL

                SELECT 'Debit' AS Description, Amount, TransactionDate
                FROM Debit WITH (NOLOCK)
                WHERE AccountId = @AccountId
                ORDER BY TransactionDate ASC;";

            var connection = _context.Database.GetDbConnection();
            if (connection.State != ConnectionState.Open)
            {
                await connection.OpenAsync(cancellationToken);
            }

            var command = new CommandDefinition(sql, new { AccountId = accountId }, cancellationToken: cancellationToken);
            using var multi = await connection.QueryMultipleAsync(command);

            var accountExists = await multi.ReadSingleOrDefaultAsync<Guid?>();
            if (accountExists == null)
            {
                return null;
            }

            var rawTransactions = (await multi.ReadAsync<RawTransactionRecord>()).ToList();

            decimal balance = 0;
            var transactions = new List<TransactionItemDto>(rawTransactions.Count);
            foreach (var item in rawTransactions)
            {
                if (string.Equals(item.Description, "Credit", StringComparison.OrdinalIgnoreCase))
                {
                    balance += item.Amount;
                }
                else if (string.Equals(item.Description, "Debit", StringComparison.OrdinalIgnoreCase))
                {
                    balance -= item.Amount;
                }

                transactions.Add(new TransactionItemDto(item.Description, item.Amount, item.TransactionDate));
            }

            return new AccountDetailsDto(accountId, balance, transactions);
        }

        // 非关系型（例如 EF Core InMemory 单元测试）环境的平滑兼容
        var account = await _context.Accounts
            .AsNoTracking()
            .FirstOrDefaultAsync(a => a.Id == accountId, cancellationToken);

        if (account == null)
        {
            return null;
        }

        var credits = await _context.Credits
            .AsNoTracking()
            .Where(c => c.AccountId == accountId)
            .ToListAsync(cancellationToken);

        var debits = await _context.Debits
            .AsNoTracking()
            .Where(d => d.AccountId == accountId)
            .ToListAsync(cancellationToken);

        var fallbackTransactions = credits
            .Select(c => new TransactionItemDto("Credit", c.Amount.ToMoney().ToDecimal(), c.TransactionDate))
            .Concat(debits.Select(d => new TransactionItemDto("Debit", d.Amount.ToMoney().ToDecimal(), d.TransactionDate)))
            .OrderBy(t => t.TransactionDate)
            .ToList();

        decimal fallbackBalance = fallbackTransactions.Sum(t => t.Description == "Credit" ? t.Amount : -t.Amount);

        return new AccountDetailsDto(accountId, fallbackBalance, fallbackTransactions);
    }

    private sealed record RawTransactionRecord(string Description, decimal Amount, DateTime TransactionDate);
}
```

QueryMultipleAsync绕过领域聚合根和仓储，专门用于高频读取，  复合查询：把查账户主表和流水表合并为一条 SQL，网络往返（Round-Trip）从 2 次减为 1 次；WITH (NOLOCK) 脏读提示：在 SQL Server 中，写事务默认会给行或页加排他锁（X 锁）。普通 SELECT 会被阻塞挂起。加上 WITH (NOLOCK)后，读请求不再等待锁释放，彻底杜绝读写互锁； 零 ChangeTracker 开销：纯 ADO.NET 流式反序列化为只读 record，不产生多余的对象状态树。

因为我们已经把接口拆成了 IAccountRepository 和 IAccountQueries，改造连接串变得轻而易举：

```
1. 配置两个独立的连接字符串（appsettings.json）：
"ConnectionStrings": {
  "MasterConnection": "Server=sql-master;Database=dddotNet;...",
  "SlaveConnection": "Server=sql-slave-readonly;Database=dddotNet;..."
}

2. 依赖注入时分别绑定：
  • GenocsContext（EF Core 写库）绑定 MasterConnection；
  • AccountQueries（Dapper 读库）连接 SlaveConnection。
3. 业务层和用例代码仍然一行都不用改！
```

架构闭环：基于 MassTransit Outbox 的“异构读模型投影”：

最极致的读写分离，甚至连关系型数据库的从库都不用，而是异构存储，写操作：由 EF Core 写 SQL Server，保证账户账目的强一致性；读模型（Read Projection）：

• 这时，我们刚刚配置的 MassTransit Outbox 派上了巨大用场
• 当出款/存款成功后，Outbox 投递了 DepositCompleted 事件；
• 后台服务（Worker）监听到这个事件，直接把最新的账户信息更新到 Redis 或 Elasticsearch中！

这时候需要关注一个叫做主从延迟的问题：

前端读请求AccountQueries 直接从 Redis（几毫秒极速读取） 或 Elasticsearch（秒级复杂搜索） 取出组装好的 DTO 返回，主库完全零读压力。用户存了 100块钱，提交成功后页面自动刷新，一查余额，发现钱居然没多？（因为主库写完了，从库还没同步过来，用户读到了旧副本）。

可以实现写后回传，存钱接口在执行成功时，直接把最新的 CurrentBalance 放在 HTTP Response里带回前端，前端用本地数据更新页面。或者短暂强制读主库。

最后可以看出的是，领域层和聚合根都是专门为了写操作而生的，读操作不需要经过聚合根，可以直接在用例层直接调查询服务直奔数据库查出DTO。读操作不改变任何业务状态，所以也不需要做任何防御和规则校验。强行走聚合根，纯属浪费 CPU 和内存。

读用例（GetAccountDetails.cs）：

```cs
/// <summary>
/// 获取账户详情用例（CQRS 读路径）.
/// 通过 IAccountQueries 直接读取扁平只读模型，完全绕过领域聚合根与 EF Core 仓储.
/// </summary>
public sealed class GetAccountDetails(IOutputPort outputHandler, IAccountQueries accountQueries) : IUseCase<GetAccountDetailsInput>
{
    private readonly IOutputPort _outputHandler = outputHandler ?? throw new ArgumentNullException(nameof(outputHandler));
    private readonly IAccountQueries _accountQueries = accountQueries ?? throw new ArgumentNullException(nameof(accountQueries));

    public async Task ExecuteAsync(GetAccountDetailsInput input, CancellationToken cancellationToken = default)
    {
        var accountDto = await _accountQueries.GetAccountDetailsAsync(input.AccountId, cancellationToken);

        if (accountDto == null)
        {
            _outputHandler.NotFound($"The account {input.AccountId} does not exist or is not processed yet.");
            return;
        }

        var transactions = accountDto.Transactions
            .Select(t => new Transaction(t.Description, t.Amount, t.TransactionDate))
            .ToList();

        GetAccountDetailsOutput output = new GetAccountDetailsOutput(
            accountDto.AccountId,
            accountDto.CurrentBalance,
            transactions);

        _outputHandler.Default(output);
    }
}
```

读操作真的很清晰，我们来看看写操作呢。

写操作在Clean Archticture中 + DDD 中，需要绝对可信，任何一个写操作比如充值、提现、开户、转账，都需要按照6个步骤来运转：

1. 输入校验和便捷转换： Controller -> DepositInput -> 值对象；
2. 然后加载聚合根，IAccountRepository.GetAsync；
3. 聚合根业务决策与规则守护：account.Deposit / Withdraw 校验不变性；
4. 状态变更追踪：EF Core ChangeTracker 标记 Added/Modified；
5. 事件借壳入库：`_serviceBus.PublishEventAsync 进入 Outbox`；
6. 原子事务提交： `_unitOfWork.SaveAsync` 一揽子提交；
7. 视图呈现输出：Presenter 输出给前端。

我们来看当前项目的 Deposit.cs：

```cs
   public async Task ExecuteAsync(DepositInput input, CancellationToken cancellationToken = default)
    {
        // 【步骤 1 & 2：从档案库提取聚合根】
        // 仓储去数据库里把账户记录以及历史流水全部加载出来，在内存中还原出完整的富领域对象
        var account = await _accountRepository.GetAsync(input.AccountId, cancellationToken);
        if (account == null)
        {
            _outputHandler.Error($"The account {input.AccountId} does not exist.");
            return;
        }

        // 【步骤 3：聚合根内部进行业务决策（核心领域层）】
        // 注意：用例本身不做业务计算！它把钱交给 account 自己去算、去校验！
        // 如果是 Withdraw（取款），account 会在这里检查余额够不够，透支直接被拦截
        var credit = account.Deposit(_entityFactory, input.Amount);

        // 【步骤 4：挂入仓储与变更追踪】
        await _accountRepository.UpdateAsync(account, credit, cancellationToken);

        // 【步骤 5：发布事件（进入我们刚刚配置的 MassTransit Outbox）】
        // 此时不发网络请求，而是将待发事件作为一条记录放进同一个 DbContext 内存中
        await _serviceBus.PublishEventAsync(new DepositCompleted() { ... }, cancellationToken);

        // 【步骤 6：本地数据库事务原子落盘】
        // 触发 BEGIN TRANSACTION，业务流水 + Outbox 消息同时写入 SQL Server
        await _unitOfWork.SaveAsync(cancellationToken);

        // 【步骤 7：通过 Presenter 驱动返回】
        _outputHandler.Default(new DepositOutput(credit, account.GetCurrentBalance()));
    }
```

`_accountRepository` 是依赖注入进来的，实现是namespace dddotNet.Infrastructure.PersistenceLayer.SQLServer.Repositories;中的AccountRepository类：

```cs
namespace dddotNet.Infrastructure.PersistenceLayer.SQLServer.Repositories;

public sealed class AccountRepository(GenocsContext context) : IAccountRepository
{
    private readonly GenocsContext _context = context ?? throw new ArgumentNullException(nameof(context));

    public async Task AddAsync(IAccount account, ICredit credit, CancellationToken cancellationToken = default)
    {
        await _context.Accounts.AddAsync((Account)account, cancellationToken);
        await _context.Credits.AddAsync((Credit)credit, cancellationToken);
    }

    public async Task DeleteAsync(IAccount account, CancellationToken cancellationToken = default)
    {
        var accountEntity = (Account)account;

        // 纳入 EF Core 变更追踪（由 UnitOfWork.SaveAsync 原子提交事务，杜绝提前物理删除）
        var credits = await _context.Credits
            .Where(e => e.AccountId == accountEntity.Id)
            .ToListAsync(cancellationToken);

        var debits = await _context.Debits
            .Where(e => e.AccountId == accountEntity.Id)
            .ToListAsync(cancellationToken);

        _context.Credits.RemoveRange(credits);
        _context.Debits.RemoveRange(debits);
        _context.Accounts.Remove(accountEntity);
    }

    public async Task<IAccount?> GetAsync(Guid id, CancellationToken cancellationToken = default)
    {
        var account = await _context.Accounts
            .SingleOrDefaultAsync(a => a.Id == id, cancellationToken);

        if (account == null)
        {
            return null;
        }

        // 纯异步查询流水，彻底消除线程池同步阻塞，并透传 cancellationToken
        var credits = await _context.Credits
            .Where(e => e.AccountId == id)
            .ToListAsync(cancellationToken);

        var debits = await _context.Debits
            .Where(e => e.AccountId == id)
            .ToListAsync(cancellationToken);

        account.Load(credits, debits);

        return account;
    }

    public async Task UpdateAsync(IAccount account, ICredit credit, CancellationToken cancellationToken = default)
        => await _context.Credits.AddAsync((Credit)credit, cancellationToken);

    public async Task UpdateAsync(IAccount account, IDebit debit, CancellationToken cancellationToken = default)
        => await _context.Debits.AddAsync((Debit)debit, cancellationToken);
}

```

销户操作与销户事件被牢牢锁定在同一个本地 ACID事务中，杜绝了“数据已删但事件丢失”或“事件发了但数据未删”；

在面向对象（C#）和关系型数据库（SQL Server）之间存在著名的“对象-关系阻抗失配（Object-Relational ImpedanceMismatch）”。这个 GenocsContext 做的，就是把纯洁优雅的 DDD 领域模型，无缝翻译成冰冷死板的数据库二维表。

我们在领域层使用的是强类型的值对象（Value Object），而不是裸露的基础类型：

• 客户身份证不是 string，而是 SSN；
• 客户姓名不是 string，而是 Name；
• 账户流水金额不是 decimal，而是 PositiveMoney（保证绝对非负）。

 SQL Server 根本不认识 C# 的 PositiveMoney 类，数据库列只认 DECIMAL(18,2)！上下文的作用：
 
```cs
   modelBuilder.Entity<Credit>()
        .Property(b => b.Amount)
        .HasConversion(
            v => v.ToMoney().ToDecimal(),  // 写入数据库时：把 PositiveMoney 对象转为 decimal 存入列
            v => new PositiveMoney(v));     // 从数据库读出时：把 decimal 自动 new 成 PositiveMoney 实体
```

最后要记住的是用例之间不能相互依赖，否则这种做法在架构上被严格定性为反模式（Anti-Pattern）。我们要做的是让每个用例就像一条独立的垂直业务切片，如果用例之间相调用，就是牵一发动全身了；每个用例都有自己的输入输出端口，如果用例A使用用例B，B的`_outputHandler.Default(...)` 就会直接冲垮 A 的输出流，导致web控制器拿到错误的viewModel。

我们一定要保证CQRS读写边界的纯洁。
## 消息队列与弹性通信

• Rebus.RabbitMq (10.1.1)：轻量级服务总线框架 Rebus 的 RabbitMQ 传输实现，用于事件驱动和异步解耦。
• Azure.Messaging.ServiceBus (7.20.1) / Microsoft.Azure.ServiceBus (5.2.0)：Azure 服务总线集成（在 Worker 中）。
• Microsoft.Extensions.Http.Polly (10.0.7)：提供 HTTP 请求重试、熔断与超时等弹性策略。
• Refit.HttpClientFactory (10.1.6)：类型安全的声明式 REST HTTP 客户端。

在现代的云原生和微服务架构中，服务间的通信分为两张网，一个是异步的网，（消息队列 Message Queue）：负责跨服务解耦、削峰填谷、事件驱动；一个是同步的网，HTTP远程调用，负责实时数据拉取，但必须配备弹性策略（Polly +Refit）作为保护措施，防止网络雪崩。 

这个模板里，消息队列流转覆盖了契约定义、事件发布到异步消费的完整闭环。在用例层，业务代码通过注入抽象接口 IServiceBusClient 发布事件：

```cs
await _serviceBus.PublishEventAsync(new DepositCompleted { ... }, cancellationToken);
```

底层正是由我们前面配置的 MassTransit Outbox 拦截并伴随本地数据库事务原子落盘，随后由后台线程送入 RabbitMQ。

消费者Worker时独立常驻的进程消费。打开独立进程项目 dddotNet.Worker， 查看处理器 RegistrationCompletedHandler.cs：

```cs
    public class RegistrationCompletedHandler(ILogger logger) : IHandleMessages<RegistrationCompleted>
    {
        public async Task Handle(RegistrationCompleted message)
        {
            _logger.LogInformation("用户开户成功，异步执行邮件发送与风控，CreditId: {0}", message.CreditId);
        }
    }
```

核心价值：用户的开户请求在 WebApi 几十毫秒内就响应了；发邮件、建积分档案等耗时操作全部丢给 Worker在后台慢慢消费，系统抗洪能力直接上了一个台阶。

同步通信支柱：弹性通信（Polly + Refit），当微服务不得不通过 HTTP 同步调用第三方外部服务（如银行网关、第三方订单服务）时，网络可能会抖动、超时、甚至下游直接宕机。项目使用了 Refit（声明式客户端） + Polly 来保证 HTTP 调用

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 800 900" width="100%" height="100%">
  <defs>
    <!-- 箭头标记 -->
    <marker id="arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#8892b0" />
    </marker>

    <!-- 卡片阴影效果 -->
    <filter id="shadow" x="-5%" y="-5%" width="110%" height="110%">
      <feDropShadow dx="0" dy="4" stdDeviation="6" flood-color="#000000" flood-opacity="0.3"/>
    </filter>
    
    <!-- Polly 管道外框阴影 -->
    <filter id="glow" x="-5%" y="-5%" width="110%" height="110%">
      <feGaussianBlur stdDeviation="4" result="blur" />
      <feComposite in="SourceGraphic" in2="blur" operator="over" />
    </filter>

    <!-- 背景网格 -->
    <pattern id="grid" width="40" height="40" patternUnits="userSpaceOnUse">
      <path d="M 40 0 L 0 0 0 40" fill="none" stroke="#ffffff" stroke-opacity="0.03" stroke-width="1"/>
    </pattern>
  </defs>

  <!-- 背景 -->
  <rect width="100%" height="100%" fill="#0f172a" />
  <rect width="100%" height="100%" fill="url(#grid)" />

  <!-- 1. 业务代码发起请求 -->
  <g transform="translate(200, 50)">
    <rect width="400" height="100" rx="12" fill="#1e293b" stroke="#38bdf8" stroke-width="2" filter="url(#shadow)"/>
    <text x="200" y="40" fill="#e2e8f0" font-family="sans-serif" font-size="20" font-weight="bold" text-anchor="middle">业务代码发起 HTTP 请求</text>
    <text x="200" y="70" fill="#94a3b8" font-family="monospace" font-size="16" text-anchor="middle">(IOrderApi.GetOrdersAsync)</text>
  </g>

  <!-- 箭头 1 -->
  <line x1="400" y1="150" x2="400" y2="190" stroke="#8892b0" stroke-width="2" marker-end="url(#arrow)" />

  <!-- 2. Bulkhead 舱壁隔离 -->
  <g transform="translate(200, 200)">
    <rect width="400" height="100" rx="12" fill="#1e293b" stroke="#f59e0b" stroke-width="2" filter="url(#shadow)"/>
    <text x="200" y="40" fill="#fbbf24" font-family="sans-serif" font-size="20" font-weight="bold" text-anchor="middle">1. Bulkhead 舱壁隔离</text>
    <text x="200" y="70" fill="#94a3b8" font-family="sans-serif" font-size="16" text-anchor="middle">(限流隔离，防止打爆本服务线程池)</text>
  </g>

  <!-- 箭头 2 -->
  <line x1="400" y1="300" x2="400" y2="330" stroke="#8892b0" stroke-width="2" marker-end="url(#arrow)" />

  <!-- 3. Polly 弹性策略护甲管道 (外框) -->
  <g transform="translate(150, 340)">
    <!-- Polly 外框背景与边框 -->
    <rect width="500" height="310" rx="16" fill="#172033" stroke="#8b5cf6" stroke-width="2" stroke-dasharray="8,4" opacity="0.8" filter="url(#glow)"/>
    <rect width="150" height="30" x="25" y="-15" fill="#8b5cf6" rx="15" />
    <text x="100" y="5" fill="#ffffff" font-family="sans-serif" font-size="14" font-weight="bold" text-anchor="middle">Polly 护甲管道</text>

    <!-- 管道内部箭头 -->
    <line x1="250" y1="30" x2="250" y2="60" stroke="#8892b0" stroke-width="2" marker-end="url(#arrow)" />

    <!-- 3.1 AdvancedCircuitBreaker -->
    <g transform="translate(50, 70)">
      <rect width="400" height="90" rx="8" fill="#2a1b2d" stroke="#ef4444" stroke-width="2" filter="url(#shadow)"/>
      <text x="200" y="35" fill="#fca5a5" font-family="sans-serif" font-size="18" font-weight="bold" text-anchor="middle">2. AdvancedCircuitBreaker 熔断器</text>
      <text x="200" y="60" fill="#94a3b8" font-family="sans-serif" font-size="14" text-anchor="middle">(下游挂了立即熔断跳闸，快速失败)</text>
    </g>

    <!-- 管道内部箭头 -->
    <line x1="250" y1="160" x2="250" y2="190" stroke="#8892b0" stroke-width="2" marker-end="url(#arrow)" />

    <!-- 3.2 WaitAndRetry -->
    <g transform="translate(50, 200)">
      <rect width="400" height="90" rx="8" fill="#142d25" stroke="#10b981" stroke-width="2" filter="url(#shadow)"/>
      <text x="200" y="35" fill="#6ee7b7" font-family="sans-serif" font-size="18" font-weight="bold" text-anchor="middle">3. WaitAndRetry 指数退避重试</text>
      <text x="200" y="60" fill="#94a3b8" font-family="sans-serif" font-size="14" text-anchor="middle">(网络闪断自动 1s, 2s, 4s 退避重试)</text>
    </g>
  </g>

  <!-- 箭头 3 -->
  <line x1="400" y1="650" x2="400" y2="690" stroke="#8892b0" stroke-width="2" marker-end="url(#arrow)" />

  <!-- 4. 下游外部微服务 -->
  <g transform="translate(200, 700)">
    <rect width="400" height="100" rx="12" fill="#1e293b" stroke="#ec4899" stroke-width="2" filter="url(#shadow)"/>
    <text x="200" y="40" fill="#fbcfe8" font-family="sans-serif" font-size="20" font-weight="bold" text-anchor="middle">下游外部微服务</text>
    <text x="200" y="70" fill="#94a3b8" font-family="monospace" font-size="16" text-anchor="middle">(External Order WebService)</text>
  </g>

</svg>

这里用了声明式HTTP客户端Refit，传统写法需要 new HttpClient、手动拼接 URL、手动序列化 JSON、判断 StatusCode，代码极其冗长。而在本项目中，查看 IOrderApi.cs：


```cs
    [Headers("Accept: application/json", "Authorization: Bearer xxxx")]
    public interface IOrderApi
    {
        [Get("/orders")]
        Task<HttpResponseMessage> GetOrdersAsync([AliasAs("page")] int pageIndex, [AliasAs("size")] int pageSize);
    }

```

如同写本地接口一样声明远程 API，编译时自动生成高性能的 REST 客户端。


## API 与运行宿主（API Management & Host）

在 Host.csproj 中：

• Asp.Versioning.Mvc.ApiExplorer (8.1.1)：RESTful API 多版本控制与 Swagger 探索支持。
• Microsoft.FeatureManagement.AspNetCore (4.5.0)：功能开关（Feature Flags）管理。
• EPPlus (8.5.4)：Excel 报表生成与导出处理。

## 健康检查与监控（HealthChecks & Diagnostics）

• AspNetCore.HealthChecks.RabbitMQ (9.0.0) & AspNetCore.HealthChecks.Uris (9.0.0)：健康检查端点探针。
• Microsoft.ApplicationInsights.Kubernetes (8.0.1)：K8s 环境下的 Application Insights 遥测强化。

## 全局代码质量与测试（Build & Tests）



• 通过 Directory.Build.props 全局注入：
	• Roslynator.Analyzers (4.15.0)：C# 静态代码分析与优化建议。
	• StyleCop.Analyzers (1.2.0-beta.556)：代码规范与风格审查。
• 单元测试与集成测试：
	• xunit (2.9.3) & Microsoft.NET.Test.Sdk (18.5.1)

