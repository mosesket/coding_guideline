# .NET Coding Guidelines

**Organisation-Wide Standards for ASP.NET Core Projects**

This document defines the coding standards and architecture conventions for all ASP.NET Core API projects in this organisation. It is informed by our existing Laravel codebases — the patterns, discipline, and intent carry over; only the language and idioms differ. Every new project starts from these conventions; deviations must be justified and documented in the project's own README.

---

## Table of Contents

1. [Project Structure](#1-project-structure)
2. [Naming Conventions](#2-naming-conventions)
3. [Constructor Injection — Primary Constructors](#3-constructor-injection--primary-constructors)
4. [API Response Standards](#4-api-response-standards)
5. [ResponseHelper](#5-responsehelper)
6. [Controllers](#6-controllers)
7. [Services & Interfaces](#7-services--interfaces)
8. [Request DTOs & Validation](#8-request-dtos--validation)
9. [Response DTOs](#9-response-dtos)
10. [Database — EF Core & Migrations](#10-database--ef-core--migrations)
11. [Models](#11-models)
12. [Enums](#12-enums)
13. [Exceptions](#13-exceptions)
14. [Middleware](#14-middleware)
15. [Background Jobs (Hangfire)](#15-background-jobs-hangfire)
16. [Authentication & Authorization](#16-authentication--authorization)
17. [Security](#17-security)
18. [Logging](#18-logging)
19. [Helpers & Extensions](#19-helpers--extensions)
20. [Swagger / API Documentation](#20-swagger--api-documentation)
21. [Testing](#21-testing)
22. [Performance](#22-performance)
23. [Code Style](#23-code-style)
24. [Program.cs — Startup Wiring](#24-programcs--startup-wiring)
25. [Standard Feature Workflow](#25-standard-feature-workflow)
26. [Folder Usage Summary](#26-folder-usage-summary)

---

## 1. Project Structure

```
src/
├── Controllers/
│   ├── Admin/
│   ├── Auth/
│   └── {Domain}/
├── Services/
│   ├── Interfaces/
│   └── {Domain}/
├── Models/
│   └── Audit/          ← only if a separate audit DB is used
├── DTOs/
│   ├── Requests/
│   │   └── {Domain}/
│   └── Responses/
├── Data/
│   ├── AppDbContext.cs
│   └── Migrations/
├── Enums/
├── Exceptions/
├── Middleware/
├── Helpers/
├── Extensions/
├── Jobs/
├── Configuration/
│   └── SwaggerExamples/
└── Program.cs
```

**Rules:**
- Only create folders when they are needed. No empty placeholder directories.
- Keep the structure flat within each folder — avoid nesting more than two levels deep.
- Domain subfolders (e.g. `Controllers/Admin/`) are used when a domain has three or more related files.

---

## 2. Naming Conventions

### Classes

| Type | Convention | Example |
|---|---|---|
| Controller | `{Domain}Controller` | `AuthController`, `OrderController` |
| Service | `{Domain}Service` | `UserService`, `PaymentService` |
| Service interface | `I{Domain}Service` | `IUserService`, `IPaymentService` |
| Request DTO | `{Action}{Domain}Request` | `CreateUserRequest`, `UpdateOrderRequest` |
| Response DTO | `{Domain}Response` | `UserResponse`, `OrderSummaryResponse` |
| Validator | `{Request}Validator` | `CreateUserRequestValidator` |
| Enum | Singular noun | `OrderStatus`, `PaymentMethod` |
| Exception | `{Reason}Exception` | `NotFoundException`, `InsufficientFundsException` |
| Job | `{Action}Job` | `SendEmailJob`, `ReconcilePaymentsJob` |
| Middleware | `{Purpose}Middleware` | `BannedIpMiddleware`, `HmacWebhookMiddleware` |
| Swagger example | `{Scenario}Example` | `UserCreatedExample`, `ErrorNotFoundExample` |
| Extension class | `{Type}Extensions` | `StringExtensions`, `ClaimsPrincipalExtensions` |

### Methods

- Controllers: descriptive HTTP-verb-aligned names — `Index`, `Show`, `Store`, `Update`, `Destroy`, or domain-specific action names like `Approve`, `Reject`, `Validate`
- Services: action verb + noun — `CreateUserAsync`, `GetOrderHistoryAsync`, `ProcessPaymentAsync`
- All async methods **must** carry the `Async` suffix without exception

### Variables & Properties

- `camelCase` for locals and parameters
- `PascalCase` for properties and fields
- Prefix booleans with `is`, `has`, `can` — `isActive`, `hasExpired`, `canRetry`
- Never abbreviate — `subscriberId` not `subId`, `transactionReference` not `txRef`

---

## 3. Constructor Injection — Primary Constructors

Use C# 12 primary constructors throughout. The old field-assignment constructor style is not used.

```csharp
// CORRECT — primary constructor
public class OrderController(
    IOrderService orders,
    ILogger<OrderController> logger
) : ControllerBase
{
    // `orders` and `logger` are in scope for all methods — no field declarations needed
}

public class PaymentService(
    AppDbContext db,
    IEmailService email,
    ILogger<PaymentService> logger
) : IPaymentService
{
    // parameters are directly usable in all methods
}

// WRONG — do not use the old style
public class OrderController : ControllerBase
{
    private readonly IOrderService _orders;

    public OrderController(IOrderService orders)
    {
        _orders = orders;
    }
}
```

**Middleware exception — scoped services via `InvokeAsync`:**

Middleware is registered as a singleton. If it needs a scoped service (e.g. a DbContext-backed service), inject it through the `InvokeAsync` method parameter, not the constructor:

```csharp
public class AuditMiddleware(
    RequestDelegate next,
    ILogger<AuditMiddleware> logger
)
{
    // IAuditService is scoped — cannot go in the constructor
    public async Task InvokeAsync(HttpContext context, IAuditService auditService)
    {
        await next(context);
        await auditService.LogRequestAsync(context);
    }
}
```

---

## 4. API Response Standards

### The wrapper

Every endpoint returns `ApiResponse<T>`. The JSON body `Status` field and the HTTP status code **must always match**.

```csharp
// DTOs/Responses/ApiResponse.cs
public class ApiResponse<T>
{
    public int Status { get; set; }
    public string Message { get; set; } = string.Empty;
    public T? Data { get; set; }
}

// Non-generic convenience alias for responses with no data
public class ApiResponse : ApiResponse<object> { }
```

### Status code table

| HTTP | JSON `Status` | When to use |
|---|---|---|
| 200 | 200 | Successful fetch, update, or action |
| 201 | 201 | Resource successfully created |
| 200 | 204 | List query succeeded but returned zero records (HTTP 200, body `Status: 204`) |
| 400 | 400 | Invalid input, validation failure, generic fallback error |
| 401 | 401 | Not authenticated |
| 402 | 402 | Business-level insufficient balance / quota exhausted |
| 403 | 403 | Authenticated but accessing a resource you do not own |
| 404 | 404 | Resource not found |
| 409 | 409 | Duplicate — resource already exists or action already performed |
| 410 | 410 | Resource existed but has expired or been revoked |
| 422 | 422 | Invalid state transition or business rule violation |
| 429 | 429 | Rate limited (returned by ASP.NET rate limiter automatically) |
| 500 | 500 | Unhandled exception — returned by `GlobalExceptionMiddleware` only |

> **204 convention:** `NoContent` still uses HTTP 200 so the JSON body can be read by all clients. It signals "query ran, no records" — not a missing resource (that is 404).

### Response ID key naming

Never expose a bare `Id` in a response DTO. Always prefix with the entity name:

```csharp
// WRONG
public Guid Id { get; set; }

// CORRECT
public Guid UserId { get; set; }
public Guid OrderId { get; set; }
```

### Null data responses

When an action succeeds but returns no data, pass `null` explicitly:

```csharp
return ResponseHelper.Ok<object?>(null, "Record deleted.");
```

---

## 5. ResponseHelper

**Always use `ResponseHelper` — never call `Ok()`, `BadRequest()`, etc. from `ControllerBase` directly.**

```csharp
// Helpers/ResponseHelper.cs
public static class ResponseHelper
{
    public static IActionResult Ok<T>(T data, string message = "Records fetched successfully.") =>
        new OkObjectResult(new ApiResponse<T> { Status = 200, Message = message, Data = data });

    public static IActionResult Created<T>(T data, string message = "Resource created successfully.") =>
        new ObjectResult(new ApiResponse<T> { Status = 201, Message = message, Data = data })
        { StatusCode = 201 };

    public static IActionResult NoContent(string message = "No records found.") =>
        new OkObjectResult(new ApiResponse { Status = 204, Message = message });

    public static IActionResult BadRequest(string message) =>
        new BadRequestObjectResult(new ApiResponse { Status = 400, Message = message });

    public static IActionResult Unauthorized(string message) =>
        new ObjectResult(new ApiResponse { Status = 401, Message = message }) { StatusCode = 401 };

    public static IActionResult PaymentRequired(string message) =>
        new ObjectResult(new ApiResponse { Status = 402, Message = message }) { StatusCode = 402 };

    public static IActionResult Forbidden(string message) =>
        new ObjectResult(new ApiResponse { Status = 403, Message = message }) { StatusCode = 403 };

    public static IActionResult NotFound(string message) =>
        new NotFoundObjectResult(new ApiResponse { Status = 404, Message = message });

    public static IActionResult Conflict(string message) =>
        new ConflictObjectResult(new ApiResponse { Status = 409, Message = message });

    public static IActionResult Gone(string message) =>
        new ObjectResult(new ApiResponse { Status = 410, Message = message }) { StatusCode = 410 };

    public static IActionResult UnprocessableEntity(string message) =>
        new UnprocessableEntityObjectResult(new ApiResponse { Status = 422, Message = message });
}
```

Add additional methods to `ResponseHelper` as new status codes are needed — do not create inline `ObjectResult` calls in controllers.

---

## 6. Controllers

### Rules

- Controllers are thin — one service call, then a `ResponseHelper` return
- Every method: `async Task<IActionResult>`
- Every method: wrapped in `try/catch`
- Log entry at `Information` with the key identifiers for the action
- Log unexpected exceptions at `Error` with the full exception object
- Catch domain exceptions (`AppException` subclasses) individually — map each to the correct `ResponseHelper` method
- No business logic in controllers — not even a single `if` that makes a domain decision
- No `DbContext` calls in controllers
- Policy goes on the class; per-endpoint overrides go on the method

### Pattern

```csharp
[ApiController]
[Route("api/orders")]
[Authorize(Policy = "UserOnly")]
[ProducesResponseType(typeof(ApiResponse), 401)]
[ProducesResponseType(typeof(ApiResponse), 403)]
public class OrderController(
    IOrderService orders,
    ILogger<OrderController> logger
) : ControllerBase
{
    [HttpGet("{id:guid}")]
    [ProducesResponseType(typeof(ApiResponse<OrderResponse>), 200)]
    [ProducesResponseType(typeof(ApiResponse), 404)]
    [ProducesResponseType(typeof(ApiResponse), 400)]
    public async Task<IActionResult> Show(Guid id)
    {
        logger.LogInformation("Entering Show — OrderId: {Id}", id);

        try
        {
            var userId = User.GetUserId();
            var result = await orders.GetByIdAsync(id, userId);
            return ResponseHelper.Ok(result);
        }
        catch (NotFoundException ex)    { return ResponseHelper.NotFound(ex.Message); }
        catch (ForbiddenException ex)   { return ResponseHelper.Forbidden(ex.Message); }
        catch (Exception ex)
        {
            logger.LogError(ex, "Show failed — OrderId: {Id}", id);
            return ResponseHelper.BadRequest("An error occurred. Please try again later.");
        }
    }

    [HttpPost]
    [ProducesResponseType(typeof(ApiResponse<OrderResponse>), 201)]
    [ProducesResponseType(typeof(ApiResponse), 400)]
    public async Task<IActionResult> Store([FromBody] CreateOrderRequest request)
    {
        logger.LogInformation("Entering Store");

        try
        {
            var userId = User.GetUserId();
            var result = await orders.CreateAsync(request, userId);
            return ResponseHelper.Created(result, "Order created.");
        }
        catch (InsufficientFundsException ex) { return ResponseHelper.PaymentRequired(ex.Message); }
        catch (Exception ex)
        {
            logger.LogError(ex, "Store failed");
            return ResponseHelper.BadRequest("An error occurred. Please try again later.");
        }
    }
}
```

### Route constraints

Use inline type constraints to prevent invalid values reaching service code:

```csharp
[HttpGet("{id:guid}")]
[HttpPatch("{id:guid}/approve")]
[HttpGet("{page:int}")]
```

### Anonymous endpoints

Mark anonymous endpoints explicitly with `[AllowAnonymous]`, even when the controller has no class-level `[Authorize]`:

```csharp
[HttpPost("login")]
[AllowAnonymous]
public async Task<IActionResult> Login([FromBody] LoginRequest request) { ... }
```

### Empty list responses

Use the inline ternary for list results:

```csharp
var result = await service.GetAllAsync();
return result.Any() ? ResponseHelper.Ok(result) : ResponseHelper.NoContent();
```

---

## 7. Services & Interfaces

### Rules

- Every service has a matching interface in `Services/Interfaces/`
- Services return domain objects or response DTOs — never `IActionResult`
- Services throw typed `AppException` subclasses for expected failures; infrastructure exceptions bubble up unmodified
- `SaveChangesAsync` is called inside the service, never in the controller
- Transactions wrap multi-step writes — always `await using` to guarantee disposal
- Background jobs and notifications are triggered from services, never from controllers
- Read-only queries always use `AsNoTracking()`

### Interface

```csharp
// Services/Interfaces/IOrderService.cs
public interface IOrderService
{
    Task<OrderResponse> GetByIdAsync(Guid orderId, Guid userId);
    Task<OrderResponse> CreateAsync(CreateOrderRequest request, Guid userId);
    Task CancelAsync(Guid orderId, Guid userId);
    Task<IEnumerable<OrderSummaryResponse>> GetHistoryAsync(Guid userId, int limit = 20);
}
```

### Transaction pattern

```csharp
await using var transaction = await db.Database.BeginTransactionAsync();

try
{
    // multi-step writes
    db.Orders.Add(order);
    await db.SaveChangesAsync();

    await pointsService.DebitAsync(userId, cost, "order_placement");
    await db.SaveChangesAsync();

    await transaction.CommitAsync();
}
catch (Exception ex) when (ex is not AppException)
{
    await transaction.RollbackAsync();
    logger.LogError(ex, "CreateAsync failed — UserId: {UserId}", userId);
    throw;
}
```

The `when (ex is not AppException)` guard is critical — domain exceptions must bubble up to the controller unmodified. Only unexpected infrastructure failures roll back and get logged here.

### Single-step writes

No transaction needed for a single `SaveChangesAsync`:

```csharp
db.Users.Add(user);
await db.SaveChangesAsync();
```

### Enqueueing background jobs from a service

```csharp
// Inject IBackgroundJobClient — use instance, not the static BackgroundJob
jobs.Enqueue<IEmailService>(email =>
    email.SendWelcomeAsync(user.Email, user.FirstName)
);
```

### Static data lookups

When a service needs a constant lookup table (plan → price, tier → limit, etc.), define it as a `private static readonly Dictionary` on the class:

```csharp
private static readonly Dictionary<SubscriptionPlan, (decimal Price, int Credits)> PlanMap = new()
{
    [SubscriptionPlan.Basic]    = (500m, 10),
    [SubscriptionPlan.Pro]      = (2000m, 50),
    [SubscriptionPlan.Business] = (5000m, 200),
};
```

### Service registration

All bindings go in a `ServiceCollectionExtensions` class — not scattered in `Program.cs`:

```csharp
// Extensions/ServiceCollectionExtensions.cs
public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddApplicationServices(this IServiceCollection services)
    {
        // Singletons — stateless, thread-safe
        services.AddSingleton<IEncryptionService, EncryptionService>();
        services.AddSingleton<ISecretsProvider,   EnvSecretsProvider>();

        // Scoped — one instance per HTTP request (default for everything else)
        services.AddScoped<IUserService,    UserService>();
        services.AddScoped<IOrderService,   OrderService>();
        services.AddScoped<IPaymentService, PaymentService>();
        services.AddScoped<IEmailService,   EmailService>();
        services.AddScoped<IAuditService,   AuditService>();

        services.AddHttpClient("external-api");   // named client for outbound HTTP

        return services;
    }
}
```

**Lifetime rules:**
- `Singleton` — stateless and thread-safe (encryption, secrets, config readers)
- `Scoped` — everything else; one instance per request
- `Transient` — avoid unless there is a specific reason

---

## 8. Request DTOs & Validation

### Rules

- Every endpoint receives a typed DTO — never accept raw `HttpRequest`, `dynamic`, or `JObject`
- One DTO class and one validator class, placed together in the same domain folder
- Register validators automatically — never register them one by one
- Validation failures always return `Status: 400` with an `Errors` map
- Input normalisation (e.g. phone formatting) happens in the service, not the validator — the validator checks shape only

### DTO

```csharp
// DTOs/Requests/Orders/CreateOrderRequest.cs
public class CreateOrderRequest
{
    public string ProductCode { get; set; } = string.Empty;
    public int Quantity { get; set; }
    public string DeliveryAddress { get; set; } = string.Empty;
}
```

### Validator

```csharp
// DTOs/Requests/Orders/CreateOrderRequestValidator.cs
public class CreateOrderRequestValidator : AbstractValidator<CreateOrderRequest>
{
    public CreateOrderRequestValidator()
    {
        RuleFor(x => x.ProductCode)
            .NotEmpty().WithMessage("Product code is required.")
            .MaximumLength(50).WithMessage("Product code must not exceed 50 characters.");

        RuleFor(x => x.Quantity)
            .GreaterThan(0).WithMessage("Quantity must be at least 1.")
            .LessThanOrEqualTo(100).WithMessage("Maximum order quantity is 100.");

        RuleFor(x => x.DeliveryAddress)
            .NotEmpty().WithMessage("Delivery address is required.");
    }
}
```

### Registration in Program.cs

```csharp
builder.Services.AddFluentValidationAutoValidation();
builder.Services.AddValidatorsFromAssemblyContaining<Program>();
```

### Validation failure shape — configured globally

```csharp
builder.Services.AddControllers()
    .ConfigureApiBehaviorOptions(opt =>
    {
        opt.InvalidModelStateResponseFactory = ctx =>
        {
            var errors = ctx.ModelState
                .Where(e => e.Value?.Errors.Count > 0)
                .ToDictionary(
                    e => e.Key,
                    e => e.Value!.Errors.Select(x => x.ErrorMessage).ToArray()
                );

            return new BadRequestObjectResult(new
            {
                Status  = 400,
                Message = "Validation failed.",
                Errors  = errors,
            });
        };
    });
```

---

## 9. Response DTOs

### Rules

- All response DTOs live in `DTOs/Responses/`
- Never return raw EF Core models from a controller or service — always project to a response DTO
- Related small shapes may share one file (e.g. a result DTO and its nested child DTOs)
- Use `List<T> = []` (C# 12 collection expression) as the default value for collection properties
- Money amounts get a formatted display companion: `"₦1,000,000"` or `"$5.00"`

### Example — shared file for related types

```csharp
// DTOs/Responses/OrderResponse.cs
public class OrderResponse
{
    public Guid OrderId { get; set; }
    public string ProductName { get; set; } = string.Empty;
    public int Quantity { get; set; }
    public decimal TotalAmount { get; set; }
    public string TotalDisplay { get; set; } = string.Empty;   // "₦5,000"
    public OrderStatus Status { get; set; }
    public List<OrderItemResponse> Items { get; set; } = [];
    public DateTime CreatedAt { get; set; }
}

public class OrderItemResponse
{
    public string ProductCode { get; set; } = string.Empty;
    public int Quantity { get; set; }
    public decimal UnitPrice { get; set; }
}
```

---

## 10. Database — EF Core & Migrations

### Rules

- Use EF Core Migrations — never modify the database manually in production
- Every migration has a descriptive name: `AddStatusToOrders`, `CreatePaymentsTable`
- Always provide a `Down()` method
- Add indexes on every column used in `WHERE`, `ORDER BY`, or `JOIN`

### Entity configuration — inline Fluent API in OnModelCreating

Configure all entities in `DbContext.OnModelCreating` using inline lambdas:

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Order>(b =>
    {
        b.ToTable("orders");
        b.HasKey(x => x.Id);
        b.Property(x => x.Id).HasColumnName("id").HasDefaultValueSql("UUID()");
        b.Property(x => x.UserId).HasColumnName("user_id").IsRequired();
        b.Property(x => x.Status)
            .HasColumnName("status")
            .HasConversion<string>()   // store enum as string
            .IsRequired();
        b.Property(x => x.TotalAmount)
            .HasColumnName("total_amount")
            .HasPrecision(18, 2)       // always set precision for money
            .IsRequired();
        b.Property(x => x.Metadata)
            .HasColumnName("metadata")
            .HasColumnType("JSON");    // MySQL JSON column
        b.Property(x => x.CreatedAt)
            .HasColumnName("created_at")
            .HasDefaultValueSql("UTC_TIMESTAMP()");
        b.HasIndex(x => x.UserId);
        b.HasIndex(x => x.Status);
        b.HasOne(x => x.User)
            .WithMany(u => u.Orders)
            .HasForeignKey(x => x.UserId)
            .OnDelete(DeleteBehavior.Cascade);
    });
}
```

### Column naming conventions

| Rule | Detail |
|---|---|
| Table names | `snake_case`, plural — `users`, `order_items` |
| Column names | `snake_case` — always set via `.HasColumnName()` |
| Primary key | `UUID()` default — `b.Property(x => x.Id).HasDefaultValueSql("UUID()")` |
| Created timestamp | Every table has `created_at` with `.HasDefaultValueSql("UTC_TIMESTAMP()")` |
| Money columns | Always `.HasPrecision(18, 2)` |
| Enum columns | Always `.HasConversion<string>()` — readable in the DB without a lookup table |
| JSON columns | `.HasColumnType("JSON")` for MySQL JSON type; `TEXT` for large encrypted blobs |

### Seeding defaults

Seed configuration or lookup rows in `OnModelCreating` using deterministic GUIDs so re-running migrations is idempotent:

```csharp
modelBuilder.Entity<AppConfig>().HasData(
    new
    {
        Id    = Guid.Parse("00000001-0000-0000-0000-000000000001"),
        Key   = "max_daily_limit",
        Value = "10",
        UpdatedAt = new DateTime(2026, 1, 1, 0, 0, 0, DateTimeKind.Utc),
    }
);
```

### Multiple DbContexts

When the project uses a separate audit database, maintain two DbContext classes in `Data/`:

- `AppDbContext` — all live application data
- `AuditDbContext` — immutable audit mirror on a separate DB server

Each gets its own `Migrations/` subfolder. Both are migrated at application startup:

```csharp
scope.ServiceProvider.GetRequiredService<AppDbContext>().Database.Migrate();
scope.ServiceProvider.GetRequiredService<AuditDbContext>().Database.Migrate();
```

---

## 11. Models

### Rules

- Models are EF Core entities — no HTTP dependencies, no service dependencies
- Private setters on properties that must not change after creation (`Id`, `CreatedAt`, immutable state fields)
- State transitions belong on the model as a method — the method enforces the invariant
- JSON columns are stored as `string?` — deserialise in the service layer, not in the model
- Required navigation properties use `= null!`; optional ones use `?`

### Example

```csharp
// Models/Order.cs
public class Order
{
    public Guid Id { get; private set; } = Guid.NewGuid();
    public Guid UserId { get; set; }
    public OrderStatus Status { get; private set; } = OrderStatus.Pending;
    public decimal TotalAmount { get; set; }
    public string? Metadata { get; set; }           // JSON — deserialise in service
    public DateTime? FulfilledAt { get; private set; }
    public DateTime CreatedAt { get; private set; } = DateTime.UtcNow;

    public User User { get; set; } = null!;
    public List<OrderItem> Items { get; set; } = [];

    public void Fulfil()
    {
        if (Status != OrderStatus.Processing)
            throw new InvalidStateTransitionException(
                $"Cannot fulfil an order in '{Status}' status.");

        Status      = OrderStatus.Fulfilled;
        FulfilledAt = DateTime.UtcNow;
    }

    public void Cancel()
    {
        if (Status is OrderStatus.Fulfilled or OrderStatus.Cancelled)
            throw new InvalidStateTransitionException(
                $"Cannot cancel an order in '{Status}' status.");

        Status = OrderStatus.Cancelled;
    }
}
```

Domain methods are the only place private-setter properties can be mutated — this makes invariants impossible to violate accidentally.

---

## 12. Enums

### Rules

- All enums live in `Enums/`, one file per enum
- Use `[Description]` attributes for human-readable display labels
- Put extension methods in the same file as the enum, in a paired `{Enum}Extensions` static class
- Define state transitions via a switch expression in `AllowedTransitionsFrom`

### Pattern

```csharp
// Enums/OrderStatus.cs
using System.ComponentModel;

public enum OrderStatus
{
    [Description("Pending")]     Pending,
    [Description("Processing")]  Processing,
    [Description("Fulfilled")]   Fulfilled,
    [Description("Cancelled")]   Cancelled,
    [Description("Failed")]      Failed,
}

public static class OrderStatusExtensions
{
    public static string GetLabel(this OrderStatus status) =>
        status.GetType()
              .GetField(status.ToString())!
              .GetCustomAttributes(typeof(DescriptionAttribute), false)
              is DescriptionAttribute[] { Length: > 0 } attrs
            ? attrs[0].Description
            : status.ToString();

    public static IEnumerable<OrderStatus> AllowedTransitionsFrom(this OrderStatus current) =>
        current switch
        {
            OrderStatus.Pending    => [OrderStatus.Processing, OrderStatus.Cancelled],
            OrderStatus.Processing => [OrderStatus.Fulfilled, OrderStatus.Failed],
            _                      => [],
        };

    public static bool CanTransitionTo(this OrderStatus current, OrderStatus next) =>
        current.AllowedTransitionsFrom().Contains(next);
}
```

---

## 13. Exceptions

### Base class

```csharp
// Exceptions/AppException.cs
public abstract class AppException : Exception
{
    public int StatusCode { get; }

    protected AppException(string message, int statusCode) : base(message)
        => StatusCode = statusCode;
}
```

### Catalogue of standard exceptions

| Class | HTTP | When |
|---|---|---|
| `NotFoundException` | 404 | Entity not found by ID |
| `ForbiddenException` | 403 | Caller is authenticated but does not own the resource |
| `UnauthorizedException` | 401 | Missing or invalid authentication |
| `ConflictException` | 409 | Duplicate resource or action already performed |
| `InvalidStateTransitionException` | 422 | Business rule violated / invalid state change |
| `InsufficientFundsException` | 402 | Not enough credits, balance, or quota |
| `GoneException` | 410 | Resource existed but has expired |

Add project-specific exceptions in the same `Exceptions/` folder, following the same pattern.

### Where exceptions are caught

| Layer | Catches | Does |
|---|---|---|
| Controller | Individual `AppException` subclasses | Maps each to the correct `ResponseHelper` method |
| Service | `Exception when (ex is not AppException)` | Rolls back the transaction, logs at `Error`, re-throws |
| `GlobalExceptionMiddleware` | `AppException` and `Exception` | Final safety net — returns structured JSON for any uncaught exception |

Never catch `AppException` in a service. Let it bubble up untouched to the controller.

### Global exception middleware

```csharp
// Middleware/GlobalExceptionMiddleware.cs
public class GlobalExceptionMiddleware(
    RequestDelegate next,
    ILogger<GlobalExceptionMiddleware> logger
)
{
    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await next(context);
        }
        catch (AppException ex)
        {
            logger.LogWarning("Domain exception — {Message}", ex.Message);
            context.Response.StatusCode  = ex.StatusCode;
            context.Response.ContentType = "application/json";
            await context.Response.WriteAsJsonAsync(
                new ApiResponse { Status = ex.StatusCode, Message = ex.Message });
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "Unhandled exception on {Path}", context.Request.Path);
            context.Response.StatusCode  = 500;
            context.Response.ContentType = "application/json";
            await context.Response.WriteAsJsonAsync(
                new ApiResponse { Status = 500, Message = "An error occurred. Please try again later." });
        }
    }
}
```

---

## 14. Middleware

### Rules

- Custom middleware goes in `Middleware/`, using primary constructor syntax
- Scoped services must be injected via `InvokeAsync` parameters, not the constructor
- Every early return (short-circuit) writes a JSON response via `WriteAsJsonAsync` with `ApiResponse`
- Set `Content-Type = "application/json"` before every write
- Log every short-circuit decision at `Warning` level

### Middleware pipeline order

Order is not arbitrary. Register in `Program.cs` in this sequence:

```csharp
app.UseMiddleware<GlobalExceptionMiddleware>();  // 1. outermost — catch everything
app.UseMiddleware<BannedIpMiddleware>();         // 2. IP blocklist
app.UseHttpsRedirection();                       // 3. force HTTPS
app.UseMiddleware<HmacWebhookMiddleware>();      // 4. webhook signature verification
app.UseRateLimiter();                            // 5. rate limits
app.UseAuthentication();                         // 6. JWT / cookie parsing
app.UseAuthorization();                          // 7. policy enforcement
app.UseMiddleware<AuditMiddleware>();            // 8. post-auth audit logging
```

### Short-circuit pattern

```csharp
context.Response.StatusCode  = 403;
context.Response.ContentType = "application/json";
await context.Response.WriteAsJsonAsync(new ApiResponse { Status = 403, Message = "Access denied." });
return;   // must return — do not call next
```

### Path-specific middleware

When middleware only applies to certain routes, skip early using `StartsWithSegments`:

```csharp
if (!context.Request.Path.StartsWithSegments("/api/webhooks/billing"))
{
    await next(context);
    return;
}
```

### Body re-reading

When middleware needs to read the request body (e.g. HMAC signature verification), enable buffering first then rewind:

```csharp
context.Request.EnableBuffering();
var body = await new StreamReader(context.Request.Body).ReadToEndAsync();
context.Request.Body.Position = 0;   // rewind so the controller still gets the body
```

---

## 15. Background Jobs (Hangfire)

### Rules

- Use Hangfire for all background and scheduled work
- Fire-and-forget jobs use `IBackgroundJobClient` (injected) — not the static `BackgroundJob`
- Recurring jobs are registered in `Program.cs` after `app.Build()`
- Jobs must be idempotent — safe to retry on failure
- Log job start and completion at `Information` level

### Fire-and-forget (enqueued from a service)

```csharp
// Inject IBackgroundJobClient via constructor
jobs.Enqueue<IEmailService>(email =>
    email.SendOrderConfirmationAsync(order.Id)
);
```

### Recurring jobs (Program.cs)

```csharp
RecurringJob.AddOrUpdate<IPaymentService>(
    "daily-payment-reconciliation",
    svc => svc.ReconcileAsync(),
    Cron.Daily
);
```

### Hangfire dashboard

Protect the dashboard with basic auth or an admin policy — never expose it publicly:

```csharp
app.UseHangfireDashboard("/hangfire", new DashboardOptions
{
    Authorization = [new HangfireBasicAuthFilter(dashboardUser, dashboardPass)],
});
```

---

## 16. Authentication & Authorization

### JWT setup

```csharp
builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(opt =>
    {
        opt.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer           = true,
            ValidateAudience         = true,
            ValidateLifetime         = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer              = jwtIssuer,
            ValidAudience            = jwtAudience,
            IssuerSigningKey         = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(jwtSigningKey)),
            ClockSkew = TimeSpan.Zero,  // tokens expire exactly at their `exp` claim
        };
    });
```

### Policy-based authorization

Define named policies in `Program.cs` — never hardcode role strings in individual controllers:

```csharp
builder.Services.AddAuthorization(opt =>
{
    opt.AddPolicy("AdminOnly", p => p.RequireRole("admin"));
    opt.AddPolicy("UserOnly",  p => p.RequireRole("user"));
});

// Controller
[Authorize(Policy = "AdminOnly")]
```

### Claims helpers

Extract caller identity from claims using an extension method — never read claims directly inline:

```csharp
// Extensions/ClaimsPrincipalExtensions.cs
public static class ClaimsPrincipalExtensions
{
    public static Guid GetUserId(this ClaimsPrincipal user)
    {
        var value = user.FindFirstValue(ClaimTypes.NameIdentifier)
            ?? throw new UnauthorizedException("User ID not found in token.");
        return Guid.Parse(value);
    }
}

// Controller usage
var userId = User.GetUserId();
```

---

## 17. Security

### Input safety

- Never build SQL from string concatenation or interpolation — always use EF Core parameterised queries
- Never accept server-computed values from the client (the server owns the canonical state)
- Validate resource ownership in the service before any read or write

### Ownership check pattern

```csharp
var order = await db.Orders.FirstOrDefaultAsync(o => o.Id == orderId)
    ?? throw new NotFoundException($"Order {orderId} not found.");

if (order.UserId != callerUserId)
    throw new ForbiddenException("You do not have access to this order.");
```

### Token hashing

Never store raw tokens in the database. Store only the SHA-256 hash:

```csharp
private static string ComputeSha256(string input)
{
    var bytes = SHA256.HashData(Encoding.UTF8.GetBytes(input));
    return Convert.ToHexString(bytes).ToLowerInvariant();
}
```

### Constant-time comparison

Use `CryptographicOperations.FixedTimeEquals` when comparing tokens or HMAC signatures — never `==` or `string.Equals`:

```csharp
if (!CryptographicOperations.FixedTimeEquals(
    Encoding.UTF8.GetBytes(expected),
    Encoding.UTF8.GetBytes(received)))
{
    throw new UnauthorizedException("Invalid signature.");
}
```

### Secrets via environment variables

Read secrets from environment variables at startup — never from `appsettings.json`:

```csharp
static string Require(string key) =>
    Environment.GetEnvironmentVariable(key)
    ?? throw new InvalidOperationException($"Required env var '{key}' is not set.");
```

### Sensitive data — never log

Never log: raw tokens, password hashes, encryption keys, full JWT strings, payment card details, full phone numbers. Mask before logging:

```csharp
// phone — log only first 4 digits
logger.LogInformation("Processing subscription — Phone: {Phone}", phone[..4] + "****");
```

---

## 18. Logging

### Rules

- Log controller action entry at `Information` with the key identifiers for that action
- Log service method start and successful completion at `Information`
- Log domain failures (caught `AppException`) at `Warning`
- Log unexpected exceptions at `Error` with the full exception object as the first argument
- Always pass structured parameters — never interpolate strings into log messages

```csharp
// CORRECT — structured parameters, queryable in any log aggregator
logger.LogInformation("CreateAsync started — UserId: {UserId}", userId);
logger.LogInformation("CreateAsync completed — OrderId: {OrderId}", order.Id);
logger.LogWarning("Order not found — OrderId: {OrderId}", orderId);
logger.LogError(ex, "CreateAsync failed — UserId: {UserId}", userId);

// WRONG — string interpolation loses structure
logger.LogInformation($"Order {orderId} not found");
logger.LogError($"Failed: {ex.Message}");
```

---

## 19. Helpers & Extensions

### Helpers

Static utility classes that do not depend on any injected service. Place in `Helpers/`:

```csharp
// Helpers/RandomHelper.cs
public static class RandomHelper
{
    public static string GenerateSecureToken(int byteLength = 32)
    {
        var bytes = new byte[byteLength];
        RandomNumberGenerator.Fill(bytes);
        return Convert.ToBase64String(bytes);
    }
}
```

### Extensions

Extension methods on existing types. Place in `Extensions/`, one file per extended type:

```csharp
// Extensions/StringExtensions.cs
public static class StringExtensions
{
    public static string ToTitleCase(this string value) =>
        string.IsNullOrEmpty(value)
            ? value
            : char.ToUpper(value[0]) + value[1..].ToLower();

    public static string MaskMiddle(this string value, int visibleStart = 4, int visibleEnd = 3) =>
        value.Length > visibleStart + visibleEnd
            ? value[..visibleStart] + "****" + value[^visibleEnd..]
            : "****";
}

// Extensions/ClaimsPrincipalExtensions.cs
public static class ClaimsPrincipalExtensions
{
    public static Guid GetUserId(this ClaimsPrincipal user)
    {
        var value = user.FindFirstValue(ClaimTypes.NameIdentifier)
            ?? throw new UnauthorizedException("User ID not found in token.");
        return Guid.Parse(value);
    }
}
```

**Helpers vs Extensions:**
- `Helpers/` — standalone static classes that don't extend an existing type
- `Extensions/` — extension methods that attach to an existing type (`string`, `ClaimsPrincipal`, `IServiceCollection`)

---

## 20. Swagger / API Documentation

### Rules

- Every endpoint declares `[ProducesResponseType]` for every status code it can return
- Every response shape has a matching `IExamplesProvider<T>` class in `Configuration/SwaggerExamples/`
- Common responses (401, 403) go on the controller class; endpoint-specific ones go on the method
- Example classes are grouped by domain — `OrderExamples.cs`, `UserExamples.cs`
- Swagger is only enabled in the Development environment

### Example class pattern

```csharp
// Configuration/SwaggerExamples/OrderExamples.cs
public class OrderCreatedExample : IExamplesProvider<ApiResponse<OrderResponse>>
{
    public ApiResponse<OrderResponse> GetExamples() => new()
    {
        Status  = 201,
        Message = "Order created.",
        Data    = new OrderResponse
        {
            OrderId     = Guid.Parse("aabbccdd-0000-0000-0000-000000000001"),
            ProductName = "Premium Plan",
            TotalAmount = 5000m,
            TotalDisplay = "₦5,000",
            Status      = OrderStatus.Pending,
            CreatedAt   = new DateTime(2026, 6, 15, 10, 0, 0, DateTimeKind.Utc),
        },
    };
}
```

### Wiring to an endpoint

```csharp
[HttpPost]
[SwaggerResponseExample(201, typeof(OrderCreatedExample))]
[SwaggerResponseExample(400, typeof(ErrorBadRequestExample))]
[ProducesResponseType(typeof(ApiResponse<OrderResponse>), 201)]
[ProducesResponseType(typeof(ApiResponse), 400)]
public async Task<IActionResult> Store([FromBody] CreateOrderRequest request) { ... }
```

### Swagger in Program.cs

```csharp
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI(c =>
    {
        c.SwaggerEndpoint("/swagger/v1/swagger.json", "My API v1");
        c.RoutePrefix = "swagger";
    });
}
```

---

## 21. Testing

### Rules

- Integration tests cover HTTP endpoints end-to-end via `WebApplicationFactory<Program>`
- Unit tests cover service logic in isolation with mocked interfaces (Moq)
- Use a dedicated test database — never run tests against production or staging
- Use the Arrange / Act / Assert pattern in every test method
- Test method naming: `MethodName_Scenario_ExpectedOutcome`

### Unit test example

```csharp
// Tests/Services/OrderServiceTests.cs
public class OrderServiceTests
{
    private readonly Mock<AppDbContext> _dbMock = new();
    private readonly Mock<IPaymentService> _paymentMock = new();
    private readonly OrderService _sut;

    public OrderServiceTests()
    {
        _sut = new OrderService(_dbMock.Object, _paymentMock.Object, ...);
    }

    [Fact]
    public async Task CreateAsync_ValidRequest_ReturnsCreatedOrder()
    {
        // Arrange
        var request = new CreateOrderRequest { ProductCode = "PRO", Quantity = 1 };

        // Act
        var result = await _sut.CreateAsync(request, userId: Guid.NewGuid());

        // Assert
        Assert.NotNull(result);
        Assert.Equal("PRO", result.ProductCode);
    }

    [Fact]
    public async Task CreateAsync_InsufficientFunds_ThrowsInsufficientFundsException()
    {
        // Arrange
        _paymentMock.Setup(p => p.GetBalanceAsync(It.IsAny<Guid>())).ReturnsAsync(0m);

        // Act & Assert
        await Assert.ThrowsAsync<InsufficientFundsException>(
            () => _sut.CreateAsync(new CreateOrderRequest { ProductCode = "PRO", Quantity = 1 },
                userId: Guid.NewGuid())
        );
    }
}
```

### Integration test example

```csharp
public class OrderControllerTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;

    public OrderControllerTests(WebApplicationFactory<Program> factory)
    {
        _client = factory.WithWebHostBuilder(builder =>
        {
            builder.ConfigureServices(services =>
            {
                // replace real DB with test DB
                services.RemoveAll<AppDbContext>();
                services.AddDbContext<AppDbContext>(opt =>
                    opt.UseInMemoryDatabase("TestDb"));
            });
        }).CreateClient();
    }

    [Fact]
    public async Task Post_ValidOrder_Returns201()
    {
        var response = await _client.PostAsJsonAsync("/api/orders",
            new { ProductCode = "PRO", Quantity = 1 });

        Assert.Equal(HttpStatusCode.Created, response.StatusCode);
    }
}
```

---

## 22. Performance

### Rules

- All I/O is `async/await` — never `.Result` or `.Wait()`
- Every read-only query uses `.AsNoTracking()`
- Project to DTOs in the query using `.Select()` — never load full entities for list endpoints
- Cache values that are read frequently but change rarely (`IMemoryCache` with a sensible TTL)
- Offload email, SMS, and any side-effect work to Hangfire — never block the request thread

```csharp
// Read-only list query
var orders = await db.Orders
    .AsNoTracking()
    .Where(o => o.UserId == userId)
    .Select(o => new OrderSummaryResponse
    {
        OrderId     = o.Id,
        ProductName = o.ProductName,
        TotalAmount = o.TotalAmount,
        Status      = o.Status,
        CreatedAt   = o.CreatedAt,
    })
    .OrderByDescending(o => o.CreatedAt)
    .Take(limit)
    .ToListAsync();

// Cached config value
var limit = await cache.GetOrCreateAsync("config:max_daily_orders", async entry =>
{
    entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10);
    var config = await db.AppConfigs.AsNoTracking()
        .FirstOrDefaultAsync(c => c.Key == "max_daily_orders");
    return int.Parse(config?.Value ?? "5");
});
```

---

## 23. Code Style

Follow [Microsoft C# Coding Conventions](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/coding-style/coding-conventions) as the baseline. Project-specific additions:

| Rule | Detail |
|---|---|
| C# 12 features | Use primary constructors, collection expressions `[]`, pattern matching, switch expressions everywhere |
| `var` | Use when the type is obvious from the right-hand side; explicit type otherwise |
| Expression bodies | Use for single-expression methods and properties |
| Access modifiers | Always explicit — never rely on the C# default (`private`) |
| Empty strings | `string.Empty` instead of `""` |
| Magic values | No magic numbers or strings — use named constants or enum values |
| Property names as strings | `nameof(MyProperty)` — never a string literal |
| `null!` | For required navigation properties that EF Core guarantees to populate |
| Collection initialisation | `= []` — C# 12 collection expression |
| List spreading | `[.. someList]` — collection spread expression |

```csharp
// Primary constructor
public class PaymentService(AppDbContext db, ILogger<PaymentService> logger) : IPaymentService

// Collection expression
public List<string> Tags { get; set; } = [];

// Switch expression
var label = status switch
{
    OrderStatus.Pending    => "Waiting",
    OrderStatus.Fulfilled  => "Complete",
    _                      => "Unknown",
};

// Pattern matching
is DescriptionAttribute[] { Length: > 0 } attrs ? attrs[0].Description : status.ToString()
```

---

## 24. Program.cs — Startup Wiring

`Program.cs` contains wiring only — no business logic. Standard sequence:

```
1. Load .env (development only)
2. Read all required env vars with Require() — fail fast at startup if any are missing
3. builder.Services.AddApplicationServices()   ← all domain services (one call)
4. Add DbContext(s), Redis, MemoryCache
5. Add Hangfire
6. Add Authentication (JWT)
7. Add Authorization (named policies)
8. Add RateLimiter (named policies)
9. Add Controllers + FluentValidation + Swagger
10. var app = builder.Build()
11. Migrate both DbContexts
12. app.Use* middleware — in the correct order
13. Hangfire dashboard + recurring jobs
14. Swagger (Development only)
15. app.MapControllers().RequireRateLimiting("global")
16. app.Run()
```

**Env var pattern — fail at startup:**

```csharp
static string Require(string key) =>
    Environment.GetEnvironmentVariable(key)
    ?? throw new InvalidOperationException($"Required env var '{key}' is not set.");

var dbConn    = Require("DB_CONNECTION_STRING");
var jwtKey    = Require("JWT_SIGNING_KEY");
var redisConn = Require("REDIS_CONNECTION");
```

Everything else goes into `AddApplicationServices()`. Keep `Program.cs` under 200 lines.

---

## 25. Standard Feature Workflow

Follow this order every time a new feature or endpoint is added:

1. **Route** — add the endpoint to the correct controller with the right `[Authorize]` policy
2. **Request DTO** — create `{Action}{Domain}Request` in `DTOs/Requests/{Domain}/`
3. **Validator** — create `{Request}Validator` alongside the DTO
4. **Response DTO** — create or extend the appropriate response class in `DTOs/Responses/`
5. **Interface** — add the method signature to `I{Domain}Service`
6. **Service** — implement the logic with transaction control, ownership checks, job enqueuing
7. **Controller method** — call the service, catch individual exceptions, return via `ResponseHelper`
8. **Register** — add to `AddApplicationServices()` if it is a new service
9. **Swagger examples** — add an `IExamplesProvider` class and wire with `[SwaggerResponseExample]`
10. **Tests** — at minimum one happy-path test and one failure-path test
11. **Security review** — confirm: no sensitive fields in responses, no server-computed values from client, ownership verified, nothing sensitive logged

---

## 26. Folder Usage Summary

| Folder | What goes here |
|---|---|
| `Controllers/` | HTTP routing, DTO acceptance, `ResponseHelper` calls — nothing else |
| `Services/` | All business logic, `SaveChangesAsync`, transaction control, job enqueuing |
| `Services/Interfaces/` | One interface per service — DI always binds to the interface |
| `DTOs/Requests/` | One DTO class + one validator per endpoint, grouped by domain |
| `DTOs/Responses/` | All response shapes; related small types may share a file |
| `Models/` | EF Core entities + domain methods; no HTTP or service dependencies |
| `Models/Audit/` | Audit DB entity classes (if a separate audit DB is used) |
| `Data/` | `AppDbContext`, `AuditDbContext`, `Migrations/` |
| `Enums/` | All fixed value sets; extension class in the same file |
| `Exceptions/` | `AppException` subclasses only |
| `Middleware/` | Request pipeline components |
| `Jobs/` | Hangfire job wrappers |
| `Configuration/SwaggerExamples/` | `IExamplesProvider` classes, grouped by domain |
| `Extensions/` | Extension methods on existing types, one file per type |
| `Helpers/` | Pure static utilities (`ResponseHelper`, `RandomHelper`) |

---

*Cloud Interactive Associates Limited | RC 895353*
*info@cloudinteractive.com.ng | www.cloudinteractive.com.ng*
*Document: CIAL-DOTNET-GUIDELINES-001 | Version: 2.0 | June 2026*
