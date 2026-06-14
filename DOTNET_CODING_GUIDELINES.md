# .NET Coding Guidelines

**Project-Specific Standards**

This document defines the coding standards and architecture conventions for all ASP.NET Core projects in this organisation. It is derived from our existing Laravel codebases and translated into idiomatic .NET — the patterns, discipline, and intent are the same, only the language differs.

---

## Table of Contents

1. [Project Structure](#1-project-structure)
2. [Naming Conventions](#2-naming-conventions)
3. [Database & Migrations](#3-database--migrations)
4. [Models & EF Core](#4-models--ef-core)
5. [API Response Standards](#5-api-response-standards)
6. [Controllers](#6-controllers)
7. [Services & Interfaces](#7-services--interfaces)
8. [Request Validation (DTOs)](#8-request-validation-dtos)
9. [Security](#9-security)
10. [Authentication & Authorization](#10-authentication--authorization)
11. [Enums](#11-enums)
12. [Notifications & Email](#12-notifications--email)
13. [Background Jobs](#13-background-jobs)
14. [Middleware](#14-middleware)
15. [Exceptions](#15-exceptions)
16. [Logging](#16-logging)
17. [Helpers & Extensions](#17-helpers--extensions)
18. [Testing](#18-testing)
19. [Performance](#19-performance)
20. [Code Style](#20-code-style)
21. [Standard Feature Workflow](#21-standard-feature-workflow)
22. [Folder Usage Summary](#22-folder-usage-summary)

---

## 1. Project Structure

```
src/
├── Controllers/
│   ├── Admin/
│   ├── Auth/
│   └── User/
├── Services/
│   ├── Interfaces/
│   ├── Admin/
│   ├── Auth/
│   └── Game/
├── Models/                     ← EF Core entities
├── DTOs/
│   ├── Requests/
│   │   ├── Auth/
│   │   ├── Game/
│   │   └── Admin/
│   └── Responses/
├── Enums/
├── Middleware/
├── Exceptions/
├── Helpers/
├── Extensions/
├── Notifications/
├── Jobs/                       ← Hangfire background jobs
├── Data/
│   ├── AppDbContext.cs
│   ├── AuditDbContext.cs
│   └── Migrations/
└── Configuration/
```

**Rule**
- Only create folders when needed. Do not create empty placeholder directories.
- Keep the structure flat within each folder — avoid nesting more than two levels deep.

---

## 2. Naming Conventions

### Classes

| Type | Convention | Example |
|---|---|---|
| Controller | `{Domain}Controller` | `AuthController`, `GameController` |
| Service | `{Domain}Service` | `SessionService`, `GradingService` |
| Service interface | `I{Domain}Service` | `ISessionService`, `IGradingService` |
| Request DTO | `{Action}{Domain}Request` | `CreateSubscriptionRequest`, `TapTileRequest` |
| Response DTO | `{Domain}Response` | `SessionResponse`, `GradeResultResponse` |
| Enum | `{Domain}` (singular) | `SubscriptionPlan`, `PrizeStatus` |
| Exception | `{Reason}Exception` | `SessionExpiredException`, `RoundLockedException` |
| Job | `{Action}Job` | `SendSmsJob`, `MirrorAuditRoundJob` |
| Middleware | `{Purpose}Middleware` | `BannedIpMiddleware`, `HmacWebhookMiddleware` |
| Extension | `{Type}Extensions` | `StringExtensions`, `ClaimsPrincipalExtensions` |

### Methods

- Controllers use HTTP verb names: `Index`, `Store`, `Show`, `Update`, `Destroy`
- Services use action verbs: `CreateSession`, `ProcessTap`, `GradeRound`, `GetRoundsRemaining`
- Async methods always carry the `Async` suffix: `CreateSessionAsync`, `ValidateTokenAsync`

### Variables

- Use `camelCase` for locals and parameters
- Use `PascalCase` for properties
- Avoid abbreviations: `subscriberId` not `subId`, `roundNumber` not `rndNum`
- Prefix booleans with `is`, `has`, `can`: `isExpired`, `hasActiveSubscription`

---

## 3. Database & Migrations

### Rules

- Use EF Core Migrations — never modify the database manually in production
- Every migration must have a descriptive name: `AddTokenHashToGameSessions`, `CreateRoundsTable`
- Add indexes for all columns used in `WHERE`, `ORDER BY`, or `JOIN` clauses
- Put foreign keys at the end of the entity configuration
- Always provide a `Down()` method

### Entity Configuration

Use the Fluent API in `IEntityTypeConfiguration<T>` — never use Data Annotations on models.

```csharp
// Data/Configurations/RoundConfiguration.cs
public class RoundConfiguration : IEntityTypeConfiguration<Round>
{
    public void Configure(EntityTypeBuilder<Round> builder)
    {
        builder.ToTable("rounds");

        builder.HasKey(r => r.Id);

        builder.Property(r => r.PasswordEnc)
            .IsRequired()
            .HasColumnName("password_enc");

        builder.Property(r => r.PlayerSelection)
            .HasMaxLength(6)
            .HasColumnName("player_selection");

        builder.Property(r => r.MatchCount)
            .HasColumnName("match_count");

        builder.Property(r => r.PrizeAmount)
            .HasPrecision(18, 2)
            .HasColumnName("prize_amount");

        builder.Property(r => r.CompletedAt)
            .HasColumnName("completed_at");

        builder.HasIndex(r => r.SessionId);
        builder.HasIndex(r => r.CompletedAt);

        builder.HasOne(r => r.Session)
            .WithMany(s => s.Rounds)
            .HasForeignKey(r => r.SessionId)
            .OnDelete(DeleteBehavior.Cascade);
    }
}
```

### Migration Example

```csharp
public partial class CreateRoundsTable : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.CreateTable(
            name: "rounds",
            columns: table => new
            {
                Id        = table.Column<Guid>(nullable: false, defaultValueSql: "UUID()"),
                SessionId = table.Column<Guid>(nullable: false),
                RoundNumber         = table.Column<int>(nullable: false),
                PasswordEnc         = table.Column<string>(nullable: false),
                TileLayoutAlphaEnc  = table.Column<string>(nullable: true),
                TileLayoutNumEnc    = table.Column<string>(nullable: true),
                TappedTileIds       = table.Column<string>(nullable: true),   // JSON stored as string
                PlayerSelection     = table.Column<string>(maxLength: 6, nullable: true),
                MatchCount          = table.Column<int>(nullable: true),
                PrizeAmount         = table.Column<decimal>(precision: 18, scale: 2, nullable: true),
                CompletedAt         = table.Column<DateTime>(nullable: true),
                CreatedAt           = table.Column<DateTime>(nullable: false, defaultValueSql: "UTC_TIMESTAMP()"),
            },
            constraints: table =>
            {
                table.PrimaryKey("PK_rounds", x => x.Id);
                table.ForeignKey("FK_rounds_game_sessions", x => x.SessionId,
                    principalTable: "game_sessions", principalColumn: "Id",
                    onDelete: ReferentialAction.Cascade);
            });

        migrationBuilder.CreateIndex("IX_rounds_session_id", "rounds", "SessionId");
        migrationBuilder.CreateIndex("IX_rounds_completed_at", "rounds", "CompletedAt");
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.DropTable(name: "rounds");
    }
}
```

---

## 4. Models & EF Core

### Rules

- Models are plain C# classes — no business logic, no service calls
- Navigation properties are always explicitly declared
- Use `Guid` as the primary key type — never `int` for public-facing entities
- Mark columns that must never change after creation with a private setter
- JSON columns are stored as `string` and serialised/deserialised in the service layer

### Example

```csharp
// Models/Round.cs
public class Round
{
    public Guid Id { get; private set; } = Guid.NewGuid();
    public Guid SessionId { get; set; }
    public int RoundNumber { get; set; }

    // Stored AES-256-GCM encrypted — never expose plaintext in this property
    public string PasswordEnc { get; set; } = string.Empty;
    public string? TileLayoutAlphaEnc { get; set; }
    public string? TileLayoutNumEnc { get; set; }

    // JSON array stored as string e.g. "[3,7,12]"
    public string? TappedTileIds { get; set; }

    // Server-built — client never sends this value
    public string? PlayerSelection { get; set; }

    public int? MatchCount { get; set; }
    public decimal? PrizeAmount { get; set; }
    public DateTime? CompletedAt { get; private set; }
    public DateTime CreatedAt { get; private set; } = DateTime.UtcNow;

    // Navigation
    public GameSession Session { get; set; } = null!;
    public Prize? Prize { get; set; }

    // Domain method — the only place CompletedAt can be set
    public void MarkCompleted(int matchCount, decimal? prizeAmount)
    {
        if (CompletedAt.HasValue)
            throw new RoundLockedException(Id);

        MatchCount = matchCount;
        PrizeAmount = prizeAmount;
        CompletedAt = DateTime.UtcNow;

        // Wipe encrypted tile data — no longer needed after grading
        TileLayoutAlphaEnc = null;
        TileLayoutNumEnc = null;
    }
}
```

---

## 5. API Response Standards

### CRITICAL: Status Code Always in the JSON Body

**Never** pass the HTTP status code as the second argument to `Ok()`, `BadRequest()`, etc. in a way that differs from the body. The HTTP status and the JSON body status **must match**.

Use the project's `ApiResponse<T>` wrapper for all responses.

```csharp
// DTOs/Responses/ApiResponse.cs
public class ApiResponse<T>
{
    public int Status { get; set; }
    public string Message { get; set; } = string.Empty;
    public T? Data { get; set; }
}

public class ApiResponse : ApiResponse<object> { }
```

### Status Codes in Use

```
200  OK                   Successful fetch or update
201  Created              Successful resource creation
204  No Content           Query succeeded but returned no records
400  Bad Request          Invalid input / validation failure / invalid credentials
402  Payment Required     Insufficient points / rounds exhausted
404  Not Found            Resource does not exist
409  Conflict             Duplicate tap / already used token
422  Unprocessable Entity Business rule violation / invalid state transition
```

### Response Helper — Use in Every Controller

```csharp
// Helpers/ResponseHelper.cs
public static class ResponseHelper
{
    public static IActionResult Ok<T>(T data, string message = "Records fetched successfully.")
        => new OkObjectResult(new ApiResponse<T> { Status = 200, Message = message, Data = data });

    public static IActionResult Created<T>(T data, string message = "Resource created successfully.")
        => new ObjectResult(new ApiResponse<T> { Status = 201, Message = message, Data = data }) { StatusCode = 201 };

    public static IActionResult NoContent(string message = "No records found.")
        => new OkObjectResult(new ApiResponse { Status = 204, Message = message });

    public static IActionResult BadRequest(string message)
        => new BadRequestObjectResult(new ApiResponse { Status = 400, Message = message });

    public static IActionResult PaymentRequired(string message)
        => new ObjectResult(new ApiResponse { Status = 402, Message = message }) { StatusCode = 402 };

    public static IActionResult NotFound(string message)
        => new NotFoundObjectResult(new ApiResponse { Status = 404, Message = message });

    public static IActionResult Conflict(string message)
        => new ConflictObjectResult(new ApiResponse { Status = 409, Message = message });

    public static IActionResult UnprocessableEntity(string message)
        => new UnprocessableEntityObjectResult(new ApiResponse { Status = 422, Message = message });
}
```

### Response ID Keys

Never expose raw `Id` properties directly. Always rename them descriptively in response DTOs:

```csharp
// BAD
public Guid Id { get; set; }

// GOOD
public Guid SessionId { get; set; }
public Guid RoundId { get; set; }
public Guid PrizeId { get; set; }
```

---

## 6. Controllers

### Rules

- Controllers are thin — they accept a request, call a service, return a response
- Every method must be `async` and return `Task<IActionResult>`
- Every method must be wrapped in `try/catch`
- Log entry and exit at `Information` level
- Log errors at `Error` level with full exception detail
- Never put business logic in a controller
- Never call `DbContext` directly from a controller

### Example

```csharp
// Controllers/Game/RoundController.cs
[ApiController]
[Route("api/round")]
[Authorize]
public class RoundController : ControllerBase
{
    private readonly IRoundService _roundService;
    private readonly ILogger<RoundController> _logger;

    public RoundController(IRoundService roundService, ILogger<RoundController> logger)
    {
        _roundService = roundService;
        _logger = logger;
    }

    [HttpPost("tap")]
    public async Task<IActionResult> Tap([FromBody] TapTileRequest request)
    {
        _logger.LogInformation("Entering Tap — RoundId: {RoundId}, TileId: {TileId}",
            request.RoundId, request.TileId);

        try
        {
            var subscriberId = User.GetSubscriberId();
            var result = await _roundService.TapAsync(request.RoundId, request.TileId, subscriberId);

            _logger.LogInformation("Tap completed — RoundId: {RoundId}, SlotIndex: {SlotIndex}",
                request.RoundId, result.SlotIndex);

            return ResponseHelper.Ok(result, "Tile revealed.");
        }
        catch (DuplicateTileException ex)
        {
            _logger.LogWarning("Duplicate tile tap — {Message}", ex.Message);
            return ResponseHelper.Conflict(ex.Message);
        }
        catch (RoundLockedException ex)
        {
            _logger.LogWarning("Tap on locked round — {Message}", ex.Message);
            return ResponseHelper.UnprocessableEntity(ex.Message);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unhandled error in Tap — RoundId: {RoundId}", request.RoundId);
            return ResponseHelper.BadRequest("An error occurred. Please try again later.");
        }
    }

    [HttpPost("new")]
    public async Task<IActionResult> New([FromBody] NewRoundRequest request)
    {
        _logger.LogInformation("Entering New — SessionId: {SessionId}", request.SessionId);

        try
        {
            var subscriberId = User.GetSubscriberId();
            var round = await _roundService.StartNewAsync(request.SessionId, subscriberId);

            _logger.LogInformation("New round started — RoundId: {RoundId}", round.RoundId);

            return ResponseHelper.Created(round, "New round started.");
        }
        catch (InsufficientRoundsException ex)
        {
            _logger.LogWarning("No rounds remaining — {Message}", ex.Message);
            return ResponseHelper.PaymentRequired(ex.Message);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unhandled error in New — SessionId: {SessionId}", request.SessionId);
            return ResponseHelper.BadRequest("An error occurred. Please try again later.");
        }
    }

    [HttpGet("history")]
    public async Task<IActionResult> History([FromQuery] RoundHistoryRequest request)
    {
        _logger.LogInformation("Entering History — SessionId: {SessionId}", request.SessionId);

        try
        {
            var rounds = await _roundService.GetHistoryAsync(request.SessionId, request.Limit);

            if (!rounds.Any())
                return ResponseHelper.NoContent("No completed rounds found.");

            return ResponseHelper.Ok(rounds, "Round history fetched.");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unhandled error in History — SessionId: {SessionId}", request.SessionId);
            return ResponseHelper.BadRequest("An error occurred. Please try again later.");
        }
    }
}
```

---

## 7. Services & Interfaces

### Rules

- Every service must have a matching interface in `Services/Interfaces/`
- Business logic, queries, state transitions, and transaction control all live here
- Services never return `IActionResult` — they return domain objects or throw exceptions
- Use `IDbContextFactory<T>` or injected `DbContext` — never `new DbContext()`
- Use `await using var transaction = await _db.Database.BeginTransactionAsync()` for multi-step writes
- Trigger jobs and notifications from services, never from controllers
- Keep methods domain-focused — one clear responsibility per method

### Interface

```csharp
// Services/Interfaces/IRoundService.cs
public interface IRoundService
{
    Task<TapResult> TapAsync(Guid roundId, int tileId, Guid subscriberId);
    Task<RoundStateResponse> ClearAsync(Guid roundId);
    Task<RoundResponse> StartNewAsync(Guid sessionId, Guid subscriberId);
    Task<IEnumerable<RoundHistoryItem>> GetHistoryAsync(Guid sessionId, int limit = 8);
}
```

### Implementation

```csharp
// Services/Game/RoundService.cs
public class RoundService : IRoundService
{
    private readonly AppDbContext _db;
    private readonly IGradingService _gradingService;
    private readonly ITileLayoutService _tileLayoutService;
    private readonly IPointsService _pointsService;
    private readonly IRoundAllocationService _allocationService;
    private readonly IEncryptionService _encryption;
    private readonly ILogger<RoundService> _logger;

    public RoundService(
        AppDbContext db,
        IGradingService gradingService,
        ITileLayoutService tileLayoutService,
        IPointsService pointsService,
        IRoundAllocationService allocationService,
        IEncryptionService encryption,
        ILogger<RoundService> logger)
    {
        _db            = db;
        _gradingService   = gradingService;
        _tileLayoutService = tileLayoutService;
        _pointsService    = pointsService;
        _allocationService = allocationService;
        _encryption       = encryption;
        _logger           = logger;
    }

    public async Task<TapResult> TapAsync(Guid roundId, int tileId, Guid subscriberId)
    {
        _logger.LogInformation("TapAsync started — RoundId: {RoundId}, TileId: {TileId}", roundId, tileId);

        await using var transaction = await _db.Database.BeginTransactionAsync();

        try
        {
            var round = await _db.Rounds
                .Include(r => r.Session)
                .FirstOrDefaultAsync(r => r.Id == roundId)
                ?? throw new NotFoundException($"Round {roundId} not found.");

            if (round.CompletedAt.HasValue)
                throw new RoundLockedException(roundId);

            var tappedIds = round.TappedTileIds is null
                ? new List<int>()
                : JsonSerializer.Deserialize<List<int>>(round.TappedTileIds)!;

            if (tappedIds.Contains(tileId))
                throw new DuplicateTileException(tileId);

            // Decrypt tile layout in memory — never persisted as plaintext
            var alphaLayout = JsonSerializer.Deserialize<Dictionary<int, char>>(
                _encryption.Decrypt(round.TileLayoutAlphaEnc!))!;
            var numLayout = JsonSerializer.Deserialize<Dictionary<int, char>>(
                _encryption.Decrypt(round.TileLayoutNumEnc!))!;

            var allTiles = alphaLayout.Concat(numLayout).ToDictionary(k => k.Key, v => v.Value);

            if (!allTiles.TryGetValue(tileId, out var revealedChar))
                throw new InvalidTileException(tileId);

            // Append to server-owned state — client never sends characters
            tappedIds.Add(tileId);
            round.TappedTileIds    = JsonSerializer.Serialize(tappedIds);
            round.PlayerSelection  = (round.PlayerSelection ?? string.Empty) + revealedChar;

            var slotIndex    = tappedIds.Count - 1;
            var isComplete   = round.PlayerSelection.Length == 6;
            GradeResult? grade = null;

            if (isComplete)
            {
                grade = await _gradingService.GradeAsync(round, subscriberId);
                await _pointsService.DebitAsync(subscriberId, 1, "round_completion");
            }

            await _db.SaveChangesAsync();
            await transaction.CommitAsync();

            _logger.LogInformation("TapAsync completed — RoundId: {RoundId}, SlotIndex: {SlotIndex}, IsComplete: {IsComplete}",
                roundId, slotIndex, isComplete);

            return new TapResult
            {
                RevealedChar  = revealedChar.ToString(),
                SlotIndex     = slotIndex,
                TappedCount   = tappedIds.Count,
                IsComplete    = isComplete,
                Grade         = grade,
            };
        }
        catch (Exception ex) when (ex is not AppException)
        {
            await transaction.RollbackAsync();
            _logger.LogError(ex, "TapAsync failed — RoundId: {RoundId}", roundId);
            throw;
        }
    }

    public async Task<RoundResponse> StartNewAsync(Guid sessionId, Guid subscriberId)
    {
        _logger.LogInformation("StartNewAsync — SessionId: {SessionId}", sessionId);

        var remaining = await _allocationService.GetRoundsRemainingAsync(subscriberId);

        if (remaining <= 0)
            throw new InsufficientRoundsException("No rounds remaining for today.");

        await using var transaction = await _db.Database.BeginTransactionAsync();

        try
        {
            var session = await _db.GameSessions
                .FirstOrDefaultAsync(s => s.Id == sessionId)
                ?? throw new NotFoundException($"Session {sessionId} not found.");

            var lastRound = await _db.Rounds
                .Where(r => r.SessionId == sessionId)
                .OrderByDescending(r => r.RoundNumber)
                .FirstOrDefaultAsync();

            var (alphaLayout, numLayout) = _tileLayoutService.Generate();
            var password = _passwordService.Generate();

            var round = new Round
            {
                SessionId           = sessionId,
                RoundNumber         = (lastRound?.RoundNumber ?? 0) + 1,
                PasswordEnc         = _encryption.Encrypt(password),
                TileLayoutAlphaEnc  = _encryption.Encrypt(JsonSerializer.Serialize(alphaLayout)),
                TileLayoutNumEnc    = _encryption.Encrypt(JsonSerializer.Serialize(numLayout)),
            };

            _db.Rounds.Add(round);
            await _db.SaveChangesAsync();
            await transaction.CommitAsync();

            _logger.LogInformation("New round created — RoundId: {RoundId}, Number: {Number}",
                round.Id, round.RoundNumber);

            return new RoundResponse
            {
                RoundId          = round.Id,
                RoundNumber      = round.RoundNumber,
                PasswordDisplay  = password.ToCharArray().Select(c => c.ToString()).ToList(),
                AlphaTileIds     = alphaLayout.Keys.OrderBy(_ => Guid.NewGuid()).ToList(),
                NumTileIds       = numLayout.Keys.OrderBy(_ => Guid.NewGuid()).ToList(),
                RoundsRemaining  = remaining - 1,
            };
        }
        catch (Exception ex) when (ex is not AppException)
        {
            await transaction.RollbackAsync();
            _logger.LogError(ex, "StartNewAsync failed — SessionId: {SessionId}", sessionId);
            throw;
        }
    }
}
```

### Registering Services

Register all services in a dedicated extension method, not scattered in `Program.cs`:

```csharp
// Extensions/ServiceCollectionExtensions.cs
public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddApplicationServices(this IServiceCollection services)
    {
        services.AddScoped<ISessionService,         SessionService>();
        services.AddScoped<IRoundService,           RoundService>();
        services.AddScoped<IGradingService,         GradingService>();
        services.AddScoped<ITileLayoutService,      TileLayoutService>();
        services.AddScoped<IPointsService,          PointsService>();
        services.AddScoped<IRoundAllocationService, RoundAllocationService>();
        services.AddScoped<IFraudService,           FraudService>();
        services.AddScoped<IPrizeService,           PrizeService>();
        services.AddScoped<ISmsService,             SmsService>();
        services.AddScoped<IUssdService,            UssdService>();
        services.AddSingleton<IEncryptionService,   EncryptionService>();

        return services;
    }
}
```

---

## 8. Request Validation (DTOs)

### Rules

- Every endpoint receives a typed request DTO — never accept raw `HttpRequest` or `JObject`
- Validate with FluentValidation — one validator class per request DTO
- Register validators via `AddFluentValidationAutoValidation()`
- Validation failure returns `status: 400` in the JSON body
- Normalise input in the validator's `Transform` or in a `PreProcess` step — never in the service

### Request DTO

```csharp
// DTOs/Requests/Game/TapTileRequest.cs
public class TapTileRequest
{
    public Guid RoundId { get; set; }
    public int TileId { get; set; }
}
```

### Validator

```csharp
// DTOs/Requests/Game/TapTileRequestValidator.cs
public class TapTileRequestValidator : AbstractValidator<TapTileRequest>
{
    public TapTileRequestValidator()
    {
        RuleFor(x => x.RoundId)
            .NotEmpty()
            .WithMessage("RoundId is required.");

        RuleFor(x => x.TileId)
            .GreaterThanOrEqualTo(0)
            .WithMessage("TileId must be a valid tile index.");
    }
}
```

### Validation Failure Response — Global Handler

Configure a global validation response so all 400s follow the same shape:

```csharp
// Program.cs
builder.Services.AddControllers()
    .ConfigureApiBehaviorOptions(options =>
    {
        options.InvalidModelStateResponseFactory = context =>
        {
            var errors = context.ModelState
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

### Complex Validation Example

```csharp
// DTOs/Requests/Auth/CreateSubscriptionRequestValidator.cs
public class CreateSubscriptionRequestValidator : AbstractValidator<CreateSubscriptionRequest>
{
    public CreateSubscriptionRequestValidator()
    {
        RuleFor(x => x.Phone)
            .NotEmpty().WithMessage("Phone number is required.")
            .Matches(@"^234\d{10}$").WithMessage("Phone must be in international format (234XXXXXXXXXX).")
            .Transform(x => NormalisePhone(x));

        RuleFor(x => x.Plan)
            .NotEmpty().WithMessage("Subscription plan is required.")
            .IsInEnum().WithMessage("Invalid subscription plan.");

        RuleFor(x => x.Telco)
            .NotEmpty().WithMessage("Telco is required.")
            .MaximumLength(20).WithMessage("Telco name too long.");
    }

    private static string NormalisePhone(string phone)
    {
        var digits = Regex.Replace(phone, @"[^\d]", "");

        return digits.Length == 11 && digits.StartsWith("0")
            ? "234" + digits[1..]
            : digits;
    }
}
```

---

## 9. Security

### Rules

- Never accept raw input for database queries — always use parameterised EF Core queries
- Never use `string.Format` or string interpolation to build SQL
- Never log sensitive fields: passwords, tokens, encryption keys, full card numbers
- Validate that the authenticated subscriber owns the resource before any read or write
- Tile layout data is server-side only — never include it in any response DTO
- `player_selection` is always server-built — never accept it from a client request body

### Ownership Check Pattern

```csharp
// In a service, before any operation on a round
var round = await _db.Rounds
    .Include(r => r.Session)
    .FirstOrDefaultAsync(r => r.Id == roundId);

if (round is null)
    throw new NotFoundException($"Round {roundId} not found.");

if (round.Session.SubscriberId != subscriberId)
    throw new ForbiddenException("You do not have access to this round.");
```

---

## 10. Authentication & Authorization

### JWT Setup

```csharp
// Extensions/AuthExtensions.cs
public static IServiceCollection AddJwtAuthentication(
    this IServiceCollection services, IConfiguration config)
{
    services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
        .AddJwtBearer(options =>
        {
            options.TokenValidationParameters = new TokenValidationParameters
            {
                ValidateIssuer           = true,
                ValidateAudience         = true,
                ValidateLifetime         = true,
                ValidateIssuerSigningKey = true,
                ValidIssuer              = config["Jwt:Issuer"],
                ValidAudience            = config["Jwt:Audience"],
                IssuerSigningKey         = new SymmetricSecurityKey(
                    Encoding.UTF8.GetBytes(config["Jwt:SigningKey"]!)),
                ClockSkew = TimeSpan.Zero,  // no grace period — expire exactly at midnight WAT
            };
        });

    return services;
}
```

### Claims Convention

Store subscriber identity in standard claims — never rely on custom header values:

```csharp
// Helpers/ClaimsPrincipalExtensions.cs
public static class ClaimsPrincipalExtensions
{
    public static Guid GetSubscriberId(this ClaimsPrincipal user)
    {
        var value = user.FindFirstValue(ClaimTypes.NameIdentifier)
            ?? throw new UnauthorizedException("Subscriber ID not found in token.");

        return Guid.Parse(value);
    }

    public static string GetSessionId(this ClaimsPrincipal user)
        => user.FindFirstValue("session_id")
            ?? throw new UnauthorizedException("Session ID not found in token.");
}
```

### Role-Based Authorization

Use policy-based authorization — never hardcode role strings in controllers:

```csharp
// Program.cs
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("AdminOnly",   policy => policy.RequireRole("admin"));
    options.AddPolicy("SubscriberOnly", policy => policy.RequireRole("subscriber"));
});

// Controller
[Authorize(Policy = "AdminOnly")]
[HttpGet("prizes")]
public async Task<IActionResult> GetPrizes() { ... }
```

---

## 11. Enums

### Rules

- Use enums for every fixed set of values — no magic strings anywhere
- Always define a `string` backing type for database storage
- Add a `GetDescription()` or `GetLabel()` method for display values
- Add transition/state methods directly on the enum where the logic belongs

```csharp
// Enums/PrizeStatus.cs
public enum PrizeStatus
{
    [Description("Pending")]
    Pending,

    [Description("Under Manual Review")]
    ManualReview,

    [Description("Awaiting Second Approval")]
    AwaitingSecondApproval,

    [Description("Processing")]
    Processing,

    [Description("Fulfilled")]
    Fulfilled,

    [Description("Failed")]
    Failed,
}

public static class PrizeStatusExtensions
{
    public static string GetLabel(this PrizeStatus status) =>
        status.GetType()
            .GetField(status.ToString())!
            .GetCustomAttribute<DescriptionAttribute>()!
            .Description;

    public static bool RequiresTwoApprovals(this PrizeStatus status) =>
        status == PrizeStatus.AwaitingSecondApproval;

    public static IEnumerable<PrizeStatus> AllowedTransitionsFrom(this PrizeStatus current) =>
        current switch
        {
            PrizeStatus.Pending               => [PrizeStatus.ManualReview],
            PrizeStatus.ManualReview          => [PrizeStatus.AwaitingSecondApproval, PrizeStatus.Failed],
            PrizeStatus.AwaitingSecondApproval => [PrizeStatus.Processing, PrizeStatus.Failed],
            PrizeStatus.Processing            => [PrizeStatus.Fulfilled, PrizeStatus.Failed],
            _                                 => [],
        };

    public static bool CanTransitionTo(this PrizeStatus current, PrizeStatus next) =>
        current.AllowedTransitionsFrom().Contains(next);
}
```

---

## 12. Notifications & Email

### Rules

- Trigger all notifications from Services — never from Controllers or Jobs directly
- Use Hangfire to enqueue notification delivery asynchronously
- Keep notification classes focused on a single event

```csharp
// Notifications/GameLinkSmsNotification.cs
public class GameLinkSmsNotification
{
    public string Phone { get; }
    public string RawToken { get; }
    public string GameUrl => $"https://game.cial.ng/play?t={RawToken}";

    public GameLinkSmsNotification(string phone, string rawToken)
    {
        Phone    = phone;
        RawToken = rawToken;
    }
}

// Services/Game/SessionService.cs — enqueue from service
_backgroundJobClient.Enqueue<ISmsService>(
    sms => sms.SendGameLinkAsync(new GameLinkSmsNotification(subscriber.Phone, rawToken)));
```

---

## 13. Background Jobs

### Rules

- Use Hangfire for all background and scheduled work
- Job classes are thin — they receive dependencies and call a service method
- Jobs must be idempotent — safe to retry on failure
- Log job start and completion at `Information` level
- Use `[AutomaticRetry(Attempts = 3)]` for all jobs

```csharp
// Jobs/MirrorAuditRoundJob.cs
[AutomaticRetry(Attempts = 3)]
public class MirrorAuditRoundJob
{
    private readonly IAuditService _auditService;
    private readonly ILogger<MirrorAuditRoundJob> _logger;

    public MirrorAuditRoundJob(IAuditService auditService, ILogger<MirrorAuditRoundJob> logger)
    {
        _auditService = auditService;
        _logger       = logger;
    }

    public async Task ExecuteAsync(Guid roundId)
    {
        _logger.LogInformation("MirrorAuditRoundJob started — RoundId: {RoundId}", roundId);

        await _auditService.MirrorRoundAsync(roundId);

        _logger.LogInformation("MirrorAuditRoundJob completed — RoundId: {RoundId}", roundId);
    }
}
```

### Recurring Jobs — Register at Startup

```csharp
// Program.cs
app.UseHangfireDashboard("/hangfire", new DashboardOptions
{
    Authorization = [new HangfireAdminAuthFilter()]
});

RecurringJob.AddOrUpdate<IPrizeService>(
    "prize-reconciliation",
    svc => svc.ReconcileWithPaymentSystemAsync(),
    Cron.Daily);
```

---

## 14. Middleware

### Rules

- Custom middleware goes in `Middleware/`
- Every middleware must call `_next(context)` unless it intentionally short-circuits
- Log short-circuit decisions at `Warning` level
- Return structured JSON error responses — never plain text

### Example

```csharp
// Middleware/BannedIpMiddleware.cs
public class BannedIpMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<BannedIpMiddleware> _logger;

    public BannedIpMiddleware(RequestDelegate next, ILogger<BannedIpMiddleware> logger)
    {
        _next   = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context, IBanService banService)
    {
        var ip = context.Connection.RemoteIpAddress?.ToString();

        if (ip is not null && await banService.IsIpBannedAsync(ip))
        {
            _logger.LogWarning("Banned IP blocked — IP: {Ip}", ip);

            context.Response.StatusCode  = StatusCodes.Status403Forbidden;
            context.Response.ContentType = "application/json";

            await context.Response.WriteAsJsonAsync(new ApiResponse
            {
                Status  = 403,
                Message = "Access denied.",
            });

            return;
        }

        await _next(context);
    }
}
```

### Register Middleware in Correct Order

```csharp
// Program.cs — order matters
app.UseMiddleware<BannedIpMiddleware>();
app.UseHttpsRedirection();
app.UseRateLimiter();
app.UseAuthentication();
app.UseAuthorization();
app.UseMiddleware<AdminAuditMiddleware>();
```

---

## 15. Exceptions

### Rules

- Define a base `AppException` that all domain exceptions extend
- Domain exceptions are **not** errors — they are expected states, caught individually in controllers
- Never catch `AppException` subclasses in services — let them bubble up to the controller
- Only catch `Exception` in services to rollback transactions, then re-throw

```csharp
// Exceptions/AppException.cs
public abstract class AppException : Exception
{
    public int StatusCode { get; }

    protected AppException(string message, int statusCode) : base(message)
    {
        StatusCode = statusCode;
    }
}

// Exceptions/NotFoundException.cs
public class NotFoundException : AppException
{
    public NotFoundException(string message) : base(message, 404) { }
}

// Exceptions/DuplicateTileException.cs
public class DuplicateTileException : AppException
{
    public DuplicateTileException(int tileId)
        : base($"Tile {tileId} has already been tapped this round.", 409) { }
}

// Exceptions/RoundLockedException.cs
public class RoundLockedException : AppException
{
    public RoundLockedException(Guid roundId)
        : base($"Round {roundId} is complete and cannot be modified.", 422) { }
}

// Exceptions/InsufficientRoundsException.cs
public class InsufficientRoundsException : AppException
{
    public InsufficientRoundsException(string message) : base(message, 402) { }
}

// Exceptions/SessionExpiredException.cs
public class SessionExpiredException : AppException
{
    public SessionExpiredException() : base("Session has expired.", 410) { }
}
```

### Global Exception Handler

```csharp
// Middleware/GlobalExceptionMiddleware.cs
public class GlobalExceptionMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<GlobalExceptionMiddleware> _logger;

    public GlobalExceptionMiddleware(RequestDelegate next, ILogger<GlobalExceptionMiddleware> logger)
    {
        _next   = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (AppException ex)
        {
            _logger.LogWarning("Domain exception — {Message}", ex.Message);

            context.Response.StatusCode  = ex.StatusCode;
            context.Response.ContentType = "application/json";

            await context.Response.WriteAsJsonAsync(new ApiResponse
            {
                Status  = ex.StatusCode,
                Message = ex.Message,
            });
        }
        catch (Exception ex)
        {
            // Never leak stack traces to the client
            _logger.LogError(ex, "Unhandled exception on {Path}", context.Request.Path);

            context.Response.StatusCode  = 400;
            context.Response.ContentType = "application/json";

            await context.Response.WriteAsJsonAsync(new ApiResponse
            {
                Status  = 400,
                Message = "An error occurred. Please try again later.",
            });
        }
    }
}
```

---

## 16. Logging

### Rules

- Log every controller action entry at `Information` with key identifiers
- Log every successful service outcome at `Information`
- Log expected domain failures at `Warning`
- Log unexpected exceptions at `Error` with the full exception object
- Never log: passwords, raw tokens, encryption keys, full JWT strings
- Always pass structured parameters — never interpolate strings directly into log messages

```csharp
// CORRECT — structured, queryable
_logger.LogInformation("Session validated — SubscriberId: {SubscriberId}, SessionId: {SessionId}",
    subscriberId, sessionId);

_logger.LogWarning("Token expired — TokenHash: {TokenHash}, ExpiredAt: {ExpiredAt}",
    tokenHash, expiredAt);

_logger.LogError(ex, "GradeAsync failed — RoundId: {RoundId}", roundId);

// WRONG — interpolated string loses structure
_logger.LogInformation($"Session validated for {subscriberId}");
```

---

## 17. Helpers & Extensions

### Rules

- Helpers are pure utilities — no business logic, no database calls, no service dependencies
- Extension methods go in `Extensions/` — one file per extended type
- Never add extension methods to types you do not own unless genuinely general-purpose

```csharp
// Helpers/PasswordHelper.cs
public static class PasswordHelper
{
    private static readonly char[] Letters = "ABCDEFGHIJKLMNOPQRSTUVWXYZ".ToCharArray();
    private static readonly char[] Digits  = "0123456789".ToCharArray();

    // Generates 4 unique letters + 2 unique digits, shuffled
    public static string Generate()
    {
        var letters = Letters.OrderBy(_ => RandomNumberGenerator.GetInt32(int.MaxValue))
                             .Take(4).ToArray();
        var digits  = Digits.OrderBy(_ => RandomNumberGenerator.GetInt32(int.MaxValue))
                            .Take(2).ToArray();

        return new string(letters.Concat(digits)
                                 .OrderBy(_ => RandomNumberGenerator.GetInt32(int.MaxValue))
                                 .ToArray());
    }
}

// Extensions/StringExtensions.cs
public static class StringExtensions
{
    public static string NormalisePhone(this string phone)
    {
        var digits = Regex.Replace(phone, @"[^\d]", "");

        return digits.Length == 11 && digits.StartsWith("0")
            ? "234" + digits[1..]
            : digits;
    }

    public static string MaskPhone(this string phone) =>
        phone.Length >= 7
            ? phone[..4] + "****" + phone[^3..]
            : "****";
}
```

---

## 18. Testing

### Rules

- Feature tests cover HTTP endpoints end-to-end using `WebApplicationFactory`
- Unit tests cover service logic in isolation using mocked interfaces (Moq)
- Use a dedicated in-memory or test database — never run tests against production
- Use the Arrange / Act / Assert structure in every test
- Name tests: `MethodName_Scenario_ExpectedOutcome`

```csharp
// Tests/Services/GradingServiceTests.cs
public class GradingServiceTests
{
    private readonly Mock<AppDbContext> _dbMock = new();
    private readonly Mock<IPointsService> _pointsMock = new();
    private readonly GradingService _sut;

    public GradingServiceTests()
    {
        _sut = new GradingService(_dbMock.Object, _pointsMock.Object, ...);
    }

    [Fact]
    public async Task GradeAsync_SixOfSixMatch_ReturnsPrizeTwelvePointFiveMillion()
    {
        // Arrange
        var round = new Round { PlayerSelection = "B7MQ3Z", PasswordEnc = _enc.Encrypt("B7MQ3Z") };

        // Act
        var result = await _sut.GradeAsync(round, Guid.NewGuid());

        // Assert
        Assert.Equal(6, result.MatchCount);
        Assert.Equal(12_500_000m, result.PrizeAmount);
    }

    [Fact]
    public async Task GradeAsync_FewerThanThreeMatches_CreatesNoPrizeRecord()
    {
        // Arrange
        var round = new Round { PlayerSelection = "XYZABC", PasswordEnc = _enc.Encrypt("123456") };

        // Act
        var result = await _sut.GradeAsync(round, Guid.NewGuid());

        // Assert
        Assert.Equal(0, result.MatchCount);
        Assert.Null(result.PrizeAmount);
        _dbMock.Verify(db => db.Prizes.Add(It.IsAny<Prize>()), Times.Never);
    }
}
```

---

## 19. Performance

### Rules

- Always use `async/await` on every I/O operation — never `.Result` or `.Wait()`
- Use `AsNoTracking()` for all read-only queries
- Use `Select()` to project only the columns you need — never load full entities for list views
- Offload SMS, email, and audit mirroring to Hangfire — never block the request thread
- Cache `CampaignConfig` values in `IMemoryCache` — it changes rarely and is read on every tap

```csharp
// Read-only query — always AsNoTracking
var sessions = await _db.GameSessions
    .AsNoTracking()
    .Where(s => s.SubscriberId == subscriberId)
    .Select(s => new SessionSummary { SessionId = s.Id, CreatedAt = s.CreatedAt })
    .ToListAsync();

// Cache campaign config
public async Task<int> GetMaxRoundsPerDayAsync()
{
    return await _cache.GetOrCreateAsync("config:max_rounds_per_day", async entry =>
    {
        entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10);
        var config = await _db.CampaignConfigs.FirstOrDefaultAsync(c => c.Key == "max_rounds_per_day");
        return int.Parse(config?.Value ?? "10");
    });
}
```

---

## 20. Code Style

### Rules

- Follow [Microsoft C# Coding Conventions](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/coding-style/coding-conventions)
- Use C# 12 features where available: primary constructors, collection expressions, pattern matching
- Use `var` only when the type is obvious from the right-hand side
- Prefer expression bodies for single-line properties and simple methods
- Always use explicit access modifiers — never rely on defaults
- Use `string.Empty` over `""`
- No magic numbers or magic strings — use constants or enums
- Use `nameof()` when referencing property names as strings

```csharp
// CORRECT
private const int MaxTileIndex = 35;

if (tileId > MaxTileIndex)
    throw new InvalidTileException(tileId);

// WRONG
if (tileId > 35)
    throw new Exception("bad tile");
```

---

## 21. Standard Feature Workflow

Follow this order every time a new feature is added:

1. **Route** — add endpoint to the correct controller group in the router
2. **Request DTO** — create `{Action}{Domain}Request` in `DTOs/Requests/{Domain}/`
3. **Validator** — create `{Request}Validator` alongside the DTO
4. **Response DTO** — create `{Domain}Response` in `DTOs/Responses/`
5. **Interface** — add method signature to `I{Domain}Service`
6. **Service implementation** — implement business logic, transaction control, side effects
7. **Controller method** — call service, handle exceptions, return via `ResponseHelper`
8. **Register** — add service binding in `ServiceCollectionExtensions` if new
9. **Test** — write at minimum one happy-path and one failure-path test
10. **Checklist** — verify: no tile layout in response, no `player_selection` from client, no secrets logged

---

## 22. Folder Usage Summary

| Folder | Purpose |
|---|---|
| `Controllers/` | HTTP routing, request acceptance, response formatting |
| `Services/` | All business logic, state transitions, transaction control |
| `Services/Interfaces/` | Contracts for every service — DI always binds to interface |
| `DTOs/Requests/` | Typed input objects, one per endpoint |
| `DTOs/Responses/` | Typed output objects, never expose raw EF models |
| `Models/` | EF Core entities — no business logic |
| `Data/` | DbContext, AuditDbContext, Configurations, Migrations |
| `Enums/` | All fixed value sets — no magic strings anywhere |
| `Exceptions/` | Domain exceptions extending `AppException` |
| `Middleware/` | Request pipeline components |
| `Jobs/` | Hangfire background and recurring jobs |
| `Notifications/` | Notification payload classes |
| `Helpers/` | Pure utility functions — no dependencies |
| `Extensions/` | Extension methods on existing types |
| `Configuration/` | Strongly-typed settings classes bound from `appsettings.json` |

---

*Cloud Interactive Associates Limited | RC 895353*
*info@cloudinteractive.com.ng | www.cloudinteractive.com.ng*
*Document: CIAL-DOTNET-GUIDELINES-001 | Version: 1.0 | June 2026*
