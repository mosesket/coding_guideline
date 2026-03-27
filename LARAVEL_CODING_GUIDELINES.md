# Laravel Coding Guidelines

**Project‑Specific Standards**

This document provides the coding standards and best practices for this Laravel project. It reflects the real structure and conventions used in the codebase and must be followed across all development.

## Table of Contents
- [Project Structure](#project-structure)
- [Naming Conventions](#naming-conventions)
- [Database & Migrations](#database--migrations)
- [Models & Eloquent](#models--eloquent)
- [API Response Standards](#api-response-standards)
- [Controllers](#controllers)
- [Services](#services)
- [Validation](#validation)
- [Security](#security)
- [Authentication & Authorization](#authentication--authorization)
- [Notifications & Mail](#notifications--mail)
- [Events & Listeners](#events--listeners)
- [Observers vs Events](#observers-vs-events)
- [Console Commands](#console-commands)
- [Custom Validation Rules](#custom-validation-rules)
- [Helpers](#helpers)
- [Enums](#enums)
- [Traits](#traits)
- [Testing](#testing)
- [Performance](#performance)
- [Code Style](#code-style)
- [Documentation](#documentation)
- [Standard Feature Workflow](#standard-feature-workflow)
- [Folder Usage Summary](#folder-usage-summary)

---

## Project Structure

### Follow the Project’s Actual Layout
```
app/
├── Channels/
├── Console/
│   └── Commands/
├── Enums/
├── Helpers/
├── Http/
│   ├── Controllers/
│   │   ├── Admin/
│   │   ├── Auth/
│   │   └── User/
│   ├── Requests/
│   │   ├── Admin/
│   │   │   ├── AdminUser/
│   │   │   ├── Catalog/
│   │   │   ├── Driver/
│   │   │   ├── Order/
│   │   │   └── Wallet/
│   │   ├── Auth/
│   │   │   ├── Admin/
│   │   │   └── User/
│   │   ├── Order/
│   │   ├── User/
│   │   └── Wallet/
│   └── Resources/
├── Mail/
│   └── Auth/
├── Models/
├── Notifications/
│   ├── Admin/
│   ├── Auth/
│   ├── Order/
│   └── Wallet/
├── Observers/
├── Providers/
├── Services/
│   ├── Admin/
│   ├── Auth/
│   ├── Catalog/
│   ├── Order/
│   ├── User/
│   └── Wallet/
└── Traits/
```

**Rule**
- Only create folders when needed. Do not create empty placeholders.

---

## Naming Conventions

### Classes
- Controllers: `UserController`, `AdminController`
- Services: `OrderService`, `WalletService`
- Requests: `StoreOrderRequest`, `UpdateWalletRequest`
- Notifications: `OrderCreatedNotification`
- Mail: `AuthWelcomeMail`

### Methods
- Controllers use HTTP verbs:
  - `index`, `store`, `show`, `update`, `destroy`
- Services use action‑based verbs:
  - `createOrder`, `processPayment`, `cancelOrder`

### Variables
- Use camelCase and descriptive names:
  - `$walletBalance`, `$orderItems`, `$validatedData`

---

## Database & Migrations

### Rules
- Use descriptive migration names
- Add indexes for search/filter columns
- Put foreign keys at the end
- Avoid breaking changes without a down strategy

### Example
```php
public function up(): void
{
    Schema::create('orders', function (Blueprint $table) {
        $table->id();
        $table->unsignedBigInteger('user_id')->index();
        $table->string('status', 50)->index();
        $table->decimal('total_amount', 12, 2);
        $table->timestamps();

        $table->foreign('user_id')->references('id')->on('users');
    });
}
```

---

## Models & Eloquent

### Rules
- Always define `$fillable`
- Define relationships explicitly
- Use casts for JSON, booleans, and dates
- Use eager loading for related data

### Example
```php
class Order extends Model
{
    protected $fillable = [
        'user_id',
        'status',
        'total_amount',
    ];

    protected $casts = [
        'total_amount' => 'decimal:2',
    ];

    public function user()
    {
        return $this->belongsTo(User::class);
    }
}
```

---

## API Response Standards

### Status Codes Always in JSON Body
**CRITICAL**: Never pass HTTP status codes as the second parameter to `response()->json()`.  
All codes must be inside the JSON body.

### We Do Not Return Server Errors
**Policy**: Clients must never receive 5xx responses.  
All errors must be converted into controlled client‑facing responses.

### Standard Response Shape
```php
return response()->json([
    'status' => 200,
    'message' => 'Records fetched successfully.',
    'data' => $data,
]);
```

### Status Codes in Use (Client‑Facing)
```
200  OK                   Successful fetch/update actions
201  Created              Successful resource creation
204  No Content           No records found
400  Bad Request          Invalid credentials / invalid input
402  Payment Required     Insufficient wallet balance
404  Not Found            Resource not found
422  Unprocessable Entity Business rule failure / invalid state
```

---

## Controllers

### Keep Controllers Thin
- Accept `FormRequest`
- Call a Service
- Return JSON
- Use `try/catch`

### Example Pattern
```php
public function index(MyFilterRequest $request): JsonResponse
{
    Log::info('Entering index');

    try {
        $records = $this->service->getAll($request->validated());

        if ($records->isEmpty()) {
            return response()->json([
                'status' => 204,
                'message' => 'No records found.',
                'data' => [],
            ]);
        }

        return response()->json([
            'status' => 200,
            'message' => 'Records fetched successfully.',
            'data' => $records,
        ]);
    } catch (\Throwable $th) {
        log_error($th);

        return response()->json([
            'status' => 400,
            'message' => 'An error occurred. Please try again later',
        ]);
    }
}
```

---

## Services

### Responsibilities
- Business rules and workflows
- Queries and filtering
- State transitions
- Transaction control
- Trigger notifications/mails

### Rules
- No response formatting
- Prefer `DB::transaction()` for multi‑step changes
- Keep methods domain‑focused

---

## Validation

### Rules
- Every endpoint must use a Form Request
- Validate relationships and ownership
- Normalize input before validation

### Failure Response Policy
- `status: 400` for invalid input
- `status: 404` for failed `exists` checks

---

## Security

### Rules
- Never use `$request->all()` for mass assignment
- Always use `$request->validated()`
- Use abilities/policies for authorization
- Never build raw SQL with user input

---

## Authentication & Authorization

### Authentication
- Sanctum tokens
- User type determined by abilities:
  - `guard:admin`
  - `guard:vendor`
  - `guard:branch`
  - `guard:head_office`
  - `guard:rider`

### Authorization
- Abilities middleware required for protected actions:
  - `abilities:admin.view`
  - `abilities:settings.edit`
  - `abilities:wallet.view`

---

## Notifications & Mail

### Notifications
- Use for user alerts
- Organized by domain
- Triggered only from Services

### Mail
- Direct transactional emails
- Also triggered from Services

---

## Events & Listeners

### Use Events When
- Multiple parts of the system must react
- You want decoupled logic
- Side effects should run async

---

## Observers vs Events

### Use Observers
- Model lifecycle hooks (`created`, `updated`, `deleted`)
- Model‑specific side effects

### Use Events
- Cross‑cutting system behaviors
- Async tasks and decoupled reactions

---

## Console Commands

### Rules
- Use `resource:action` signatures
- Keep commands focused
- Return proper exit codes

---

## Custom Validation Rules

### Rules
- Use reusable rules for shared logic
- Keep validation messages clear

---

## Helpers

### Rules
- Utilities only
- No business workflows

---

## Enums

### Rules
- Use Enums for all fixed values
- Avoid magic strings

---

## Traits

### Rules
- Reusable shared behavior
- Keep traits small and focused

---

## Testing

### Expectations
- Feature tests for endpoints
- Service tests for business logic
- Use factories for data setup

---

## Performance

### Rules
- Always eager‑load relationships
- Cache expensive queries
- Offload long operations to queues

---

## Code Style

### Standards
- Follow PSR‑12
- Use type hints everywhere possible
- Avoid backslash‑prefixed facades
- Always import classes at the top

---

## Documentation

### Rules
- Add PHPDoc to service methods
- Keep comments concise and focused
- Document complex logic clearly

---

## Standard Feature Workflow

1. Add route in correct domain group.
2. Create Form Request for validation.
3. Add controller method.
4. Implement business logic in Service.
5. Return response using project JSON standard.
6. Apply abilities middleware if needed.
7. Ensure errors are logged and returned as controlled responses.

---

## Folder Usage Summary

| Folder | Purpose |
|--------|---------|
| **Enums** | Fixed statuses and types |
| **Services** | Business logic |
| **Requests** | Validation |
| **Resources** | Response formatting |
| **Notifications** | User alerts |
| **Mail** | Transactional emails |
| **Observers** | Model lifecycle logic |
| **Console/Commands** | CLI tasks |
| **Traits** | Shared behavior |

