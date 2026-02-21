# Laravel Coding Guidelines

**Generic Best Practices for All Laravel Projects**

This document provides coding standards and best practices for Laravel applications. These guidelines are framework-agnostic and should be followed across all Laravel projects.

## Table of Contents
- [Project Structure](#project-structure)
- [Naming Conventions](#naming-conventions)
- [Database & Migrations](#database--migrations)
- [Models & Eloquent](#models--eloquent)
- [Controllers](#controllers)
- [Services](#services)
- [Validation](#validation)
- [Security](#security)
- [Testing](#testing)
- [Performance](#performance)

## Project Structure

### Follow Laravel's Default Structure
```
app/
├── Console/
│   └── Commands/        # Artisan commands
├── Enums/               # PHP 8.1+ Enum classes (PostStatus, UserRole, etc.)
├── Events/              # Application events
├── Exceptions/          # Custom exception classes
├── Helpers/             # Utility functions and helper classes
├── Http/
│   ├── Controllers/     # controllers (organized by feature/domain)
│   │   └── folders/     # API controllers
│   ├── Middleware/      # Custom middleware
│   ├── Requests/        # Form validation classes
│   └── Resources/       # API response transformers
├── Jobs/                # Background/queued jobs
├── Listeners/           # Event listeners
├── Mail/                # Mailable classes
├── Models/              # Eloquent models
├── Notifications/       # Notification classes
├── Observers/           # Model observers
├── Policies/            # Authorization policies
├── Providers/           # Service providers
├── Rules/               # Custom validation rules
├── Scopes/              # Global and local query scopes
├── Services/            # Business logic and external integrations
│   └── [Domain]/        # Organize services by domain/feature
└── Traits/              # Reusable traits
```

**Note**: Only create folders when you need them. Don't create empty folders "just in case".

### Organize Routes Logically
```php
// routes/api.php
Route::prefix('v1')->group(function () {
    require __DIR__.'/api/auth.php';
    require __DIR__.'/api/posts.php';
    require __DIR__.'/api/campaigns.php';
});
```

## Naming Conventions

### Classes
- **Controllers**: Use singular noun + `Controller` suffix
  ```php
  class PostController  // NOT PostsController
  class UserController
  ```

- **Models**: Use singular, PascalCase
  ```php
  class Post
  class SocialAccount
  ```

- **Jobs**: Use verb-noun pattern
  ```php
  class SendEmailNotification
  class PublishPostJob
  ```

### Methods
- **Controllers**: Use HTTP verbs
  ```php
  public function index()   // GET /posts
  public function store()   // POST /posts
  public function show($id) // GET /posts/{id}
  public function update()  // PUT/PATCH /posts/{id}
  public function destroy() // DELETE /posts/{id}
  ```

- **Services**: Use descriptive verbs
  ```php
  public function createPost(array $data): Post
  public function publishPost(Post $post): bool
  public function schedulePost(Post $post, Carbon $time): void
  ```

### Variables & Properties
- Use camelCase
- Be descriptive
  ```php
  $userPosts       // NOT $up
  $scheduledAt     // NOT $sa
  $socialAccounts  // NOT $sas
  ```

## Database & Migrations

### Migration Naming
```bash
# Creating tables
php artisan make:migration create_posts_table

# Modifying tables
php artisan make:migration add_status_to_posts_table

# Adding columns
php artisan make:migration add_published_at_to_posts_table
```

### Migration Best Practices
```php
// ✅ GOOD
public function up(): void
{
    Schema::create('posts', function (Blueprint $table) {
        $table->uuid('id')->primary();
        $table->uuid('organization_id');
        $table->string('status', 50)->index();
        $table->timestamp('scheduled_at')->nullable();
        $table->timestamps();
        $table->softDeletes();

        // Foreign keys at the end
        $table->foreign('organization_id')
              ->references('id')
              ->on('organizations')
              ->onDelete('cascade');
    });
}

// ❌ BAD
public function up(): void
{
    Schema::create('posts', function (Blueprint $table) {
        $table->id(); // Should use UUID for consistency
        $table->string('org_id'); // Unclear naming
        $table->text('status'); // Wrong data type
        // Missing indexes, foreign keys
    });
}
```

### Use UUIDs for Public IDs
```php
use App\Traits\HasUuid;

class Post extends Model
{
    use HasUuid;

    public $incrementing = false;
    protected $keyType = 'string';
}
```

## Models & Eloquent

### Model Best Practices
```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Database\Eloquent\Factories\HasFactory;

class Post extends Model
{
    use HasFactory, SoftDeletes, HasUuid;

    // Always define fillable
    protected $fillable = [
        'organization_id',
        'status',
        'scheduled_at',
    ];

    // Define casts
    protected $casts = [
        'scheduled_at' => 'datetime',
        'published_at' => 'datetime',
        'is_recurring' => 'boolean',
        'recurrence_rule' => 'array',
    ];

    // Relationships
    public function organization(): BelongsTo
    {
        return $this->belongsTo(Organization::class);
    }

    public function variants(): HasMany
    {
        return $this->hasMany(PostVariant::class);
    }

    // Scopes
    public function scopeScheduled($query)
    {
        return $query->where('status', 'scheduled');
    }

    // Accessors & Mutators
    public function getIsScheduledAttribute(): bool
    {
        return $this->status === 'scheduled';
    }
}
```

### Avoid N+1 Queries
```php
// ✅ GOOD
$posts = Post::with(['variants', 'organization'])->get();

// ❌ BAD
$posts = Post::all();
foreach ($posts as $post) {
    $post->variants; // N+1 query
}
```

## API Response Standards

### Always Return Status in JSON Body
**CRITICAL**: Never use HTTP status codes as the second parameter in `response()->json()`. Always return status codes in the JSON body only to prevent server errors from propagating to clients.

```php
// ✅ GOOD - Status in JSON body only
return response()->json([
    'status' => 200,
    'message' => 'Success',
    'data' => $data
]);

return response()->json([
    'status' => 401,
    'message' => 'Unauthenticated'
]);

return response()->json([
    'status' => 400,
    'message' => 'Validation failed',
    'errors' => $errors
]);

// ❌ BAD - Using HTTP status code parameter
return response()->json([
    'status' => 400,
    'message' => 'Failed'
], 400); // DON'T DO THIS - Can cause server errors

return response()->json($data, 201); // DON'T DO THIS
```

### Standard Response Formats

```php
// Success with data
return response()->json([
    'status' => 200,
    'message' => 'Operation successful',
    'data' => $resource
]);

// Success with pagination
return response()->json([
    'status' => 200,
    'data' => $items,
    'meta' => [
        'current_page' => $page,
        'last_page' => $lastPage,
        'per_page' => $perPage,
        'total' => $total
    ]
]);

// Validation error
return response()->json([
    'status' => 400,
    'message' => 'Validation failed',
    'errors' => $validator->errors()
]);

// Unauthorized
return response()->json([
    'status' => 401,
    'message' => 'Unauthenticated'
]);

// Not found
return response()->json([
    'status' => 404,
    'message' => 'Resource not found'
]);

// Server error
return response()->json([
    'status' => 500,
    'message' => 'Internal server error'
]);
```

### Authentication Exceptions
Configure exception handling in `bootstrap/app.php`:

```php
->withExceptions(function (Exceptions $exceptions): void {
    // Handle authentication exceptions for API routes
    $exceptions->render(function (AuthenticationException $exception, Request $request) {
        if ($request->expectsJson() || $request->is('api/*')) {
            return response()->json([
                'status' => 401,
                'message' => 'Unauthenticated',
            ]);
        }
    });
})
```

## Controllers

### Keep Controllers Thin
```php
// ✅ GOOD
class PostController extends Controller
{
    public function __construct(
        protected PostService $postService
    ) {}

    public function store(StorePostRequest $request)
    {
        try {
            $post = $this->postService->create(
                $request->user()->currentOrganization,
                $request->validated(),
                $request->user()->id
            );

            return response()->json([
                'status' => 201,
                'message' => 'Post created successfully',
                'data' => new PostResource($post)
            ]);
        } catch (\Exception $exception) {
            Log::error('Post creation failed', ['error' => $exception->getMessage()]);

            return response()->json([
                'status' => 400,
                'message' => 'Failed to create post'
            ]);
        }
    }
}

// ❌ BAD - Fat controller with business logic
class PostController extends Controller
{
    public function store(Request $request)
    {
        // Validation in controller
        $validated = $request->validate([...]);

        // Business logic in controller
        DB::beginTransaction();
        $post = Post::create($validated);
        foreach ($request->variants as $variant) {
            $post->variants()->create($variant);
        }
        DB::commit();

        return response()->json($post);
    }
}
```

## Services

### Service Layer Pattern
```php
<?php

namespace App\Services;

use App\Models\Post;
use App\Models\Organization;
use Illuminate\Support\Facades\DB;

class PostService
{
    /**
     * Create a new post with variants.
     */
    public function create(Organization $organization, array $data, string $userId): Post
    {
        DB::beginTransaction();

        try {
            $post = Post::create([
                'organization_id' => $organization->id,
                'created_by' => $userId,
                'status' => isset($data['scheduled_at']) ? 'scheduled' : 'draft',
                'scheduled_at' => $data['scheduled_at'] ?? null,
            ]);

            if (isset($data['variants'])) {
                foreach ($data['variants'] as $variant) {
                    $post->variants()->create($variant);
                }
            }

            if ($post->status === 'scheduled') {
                PublishPostJob::dispatch($post)
                    ->delay($post->scheduled_at);
            }

            DB::commit();
            return $post->fresh(['variants']);

        } catch (\Exception $e) {
            DB::rollBack();
            throw $e;
        }
    }
}
```

## Validation

### Use Form Requests
```php
<?php

namespace App\Http\Requests;

class StorePostRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true; // Handle in policies
    }

    public function rules(): array
    {
        return [
            'scheduled_at' => 'nullable|date|after:now',
            'variants' => 'required|array|min:1',
            'variants.*.social_account_id' => 'required|uuid|exists:social_accounts,id',
            'variants.*.caption' => 'required|string|max:10000',
        ];
    }

    public function messages(): array
    {
        return [
            'variants.required' => 'At least one post variant is required',
            'scheduled_at.after' => 'Scheduled time must be in the future',
        ];
    }
}
```

## Security

### Always Validate Input
```php
// ✅ GOOD
public function store(StorePostRequest $request)
{
    $validated = $request->validated();
    // Use $validated, not $request->all()
}

// ❌ BAD
public function store(Request $request)
{
    $post = Post::create($request->all()); // Mass assignment vulnerability
}
```

### Use Policies for Authorization
```php
// PostPolicy.php
public function update(User $user, Post $post): bool
{
    return $user->id === $post->created_by
        || $user->hasRole('admin');
}

// Controller
public function update(Request $request, Post $post)
{
    $this->authorize('update', $post);
    // Update logic
}
```

### Protect Against SQL Injection
```php
// ✅ GOOD
$posts = Post::where('status', $status)->get();

// ❌ BAD
$posts = DB::select("SELECT * FROM posts WHERE status = '$status'");
```

## Testing

### Write Tests for Everything
```php
<?php

namespace Tests\Feature;

use Tests\TestCase;
use App\Models\{User, Organization, Post};
use Illuminate\Foundation\Testing\RefreshDatabase;

class PostServiceTest extends TestCase
{
    use RefreshDatabase;

    public function test_can_create_post(): void
    {
        $user = User::factory()->create();
        $organization = Organization::factory()->create();

        $post = $this->postService->create(
            $organization,
            ['scheduled_at' => now()->addDay()],
            $user->id
        );

        $this->assertInstanceOf(Post::class, $post);
        $this->assertEquals('scheduled', $post->status);
        $this->assertDatabaseHas('posts', [
            'id' => $post->id,
            'organization_id' => $organization->id,
        ]);
    }
}
```

## Performance

### Use Eager Loading
```php
// ✅ GOOD
$posts = Post::with('variants.socialAccount')->get();

// ❌ BAD
$posts = Post::all();
```

### Cache Expensive Queries
```php
$stats = Cache::remember('org-stats-' . $orgId, 3600, function () use ($orgId) {
    return DB::table('posts')
        ->where('organization_id', $orgId)
        ->selectRaw('COUNT(*) as total, SUM(views) as views')
        ->first();
});
```

### Queue Long-Running Tasks
```php
// Don't run in request
PublishPostJob::dispatch($post)->delay($post->scheduled_at);

// Not this
$this->socialMediaService->publish($post); // Blocks request
```

## Code Style

### Follow PSR-12
- Use 4 spaces for indentation
- Opening braces on same line for methods
- One blank line between methods
- Use strict types

```php
<?php

declare(strict_types=1);

namespace App\Services;

class ExampleService
{
    public function method(): void
    {
        // Code here
    }
}
```

### No Backslash Facade Calls
Never use backslash prefixed facade or helper calls. Always import the class at the top and use it without the backslash. This applies to all facades, helpers, and third-party libraries.

```php
// ✅ GOOD - Import at top, use without backslash
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Log;
use Illuminate\Support\Facades\Cache;
use Illuminate\Support\Facades\Storage;
use Illuminate\Support\Facades\Hash;
use Illuminate\Support\Facades\Mail;
use Illuminate\Support\Str;
use Carbon\Carbon;

class ExampleService
{
    public function process()
    {
        // Facades
        $data = DB::table('users')->get();
        Log::info('Processing data');
        Cache::remember('key', 3600, fn() => 'value');
        Storage::put('file.txt', 'contents');
        $hash = Hash::make('password');
        Mail::to($user)->send(new WelcomeEmail());

        // Helpers
        $uuid = Str::uuid();
        $slug = Str::slug('My Title');

        // Third-party
        $date = Carbon::parse('2024-01-01');
    }
}

// ❌ BAD - Using backslash facades
class ExampleService
{
    public function process()
    {
        // DON'T DO THIS - No imports, backslash prefixes
        $data = \DB::table('users')->get();
        \Log::info('Processing data');
        $hash = \Hash::make('password');

        // DON'T DO THIS - Full paths with backslashes
        $uuid = \Illuminate\Support\Str::uuid();
        $date = \Carbon\Carbon::parse('2024-01-01');
    }
}
```

**Why?**
- Cleaner, more readable code
- Easier to mock in tests
- Better IDE autocomplete and type hinting
- Follows PSR-12 coding standards
- Consistent with Laravel best practices

**Common Facades to Import:**
```php
// Database & Cache
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Cache;
use Illuminate\Support\Facades\Redis;
use Illuminate\Support\Facades\Schema;

// Authentication & Security
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\Hash;
use Illuminate\Support\Facades\Crypt;
use Illuminate\Support\Facades\Gate;

// Storage & Files
use Illuminate\Support\Facades\Storage;
use Illuminate\Support\Facades\File;

// Communication
use Illuminate\Support\Facades\Log;
use Illuminate\Support\Facades\Mail;
use Illuminate\Support\Facades\Notification;

// HTTP & Routing
use Illuminate\Support\Facades\Http;
use Illuminate\Support\Facades\Route;
use Illuminate\Support\Facades\URL;
use Illuminate\Support\Facades\Redirect;

// Queue & Events
use Illuminate\Support\Facades\Queue;
use Illuminate\Support\Facades\Event;
use Illuminate\Support\Facades\Bus;

// Helpers & Utilities
use Illuminate\Support\Str;
use Illuminate\Support\Arr;
use Illuminate\Support\Facades\Config;
use Illuminate\Support\Facades\Validator;

// Third-party
use Carbon\Carbon;
```

### Use Type Hints
```php
// ✅ GOOD
public function createPost(Organization $org, array $data, string $userId): Post
{
    // ...
}

// ❌ BAD
public function createPost($org, $data, $userId)
{
    // ...
}
```

## Documentation

### Add PHPDoc Blocks
```php
/**
 * Create a new post with variants.
 *
 * @param Organization $organization
 * @param array $data
 * @param string $userId
 * @return Post
 * @throws \Exception
 */
public function create(Organization $organization, array $data, string $userId): Post
{
    // ...
}
```

## Enums (PHP 8.1+)

### Use Enums for Fixed Sets of Values
```php
// ✅ GOOD - Type-safe, auto-complete
use App\Enums\PostStatus;

$post->status = PostStatus::PUBLISHED;

if ($post->status === PostStatus::DRAFT) {
    // ...
}

// ❌ BAD - Magic strings, prone to typos
$post->status = 'published';
if ($post->status === 'draft') {
    // ...
}
```

### Add Helper Methods to Enums
```php
enum PostStatus: string
{
    case DRAFT = 'draft';
    case PUBLISHED = 'published';

    public function label(): string
    {
        return match($this) {
            self::DRAFT => 'Draft',
            self::PUBLISHED => 'Published',
        };
    }

    public static function values(): array
    {
        return array_column(self::cases(), 'value');
    }
}
```

## Events & Listeners

### When to Use Events
- **Cross-cutting concerns**: When multiple parts of the app need to react
- **Decoupling**: When you don't want direct dependencies
- **Async operations**: When side effects should happen in background

### Event Structure
```php
namespace App\Events;

use App\Models\Post;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class PostPublished
{
    use Dispatchable, SerializesModels;

    public function __construct(
        public Post $post
    ) {}
}
```

### Listener Structure
```php
namespace App\Listeners;

use App\Events\PostPublished;
use Illuminate\Contracts\Queue\ShouldQueue;

class UpdatePostAnalytics implements ShouldQueue
{
    public function handle(PostPublished $event): void
    {
        // Handle the event asynchronously
    }
}
```

### Register in EventServiceProvider
```php
protected $listen = [
    PostPublished::class => [
        UpdatePostAnalytics::class,
        SendNotification::class,
    ],
];
```

## Observers vs Events

### Use Observers for Model Lifecycle
```php
class PostObserver
{
    public function created(Post $post): void
    {
        // Fired when post is created
    }

    public function updated(Post $post): void
    {
        // Check what changed
        if ($post->isDirty('status')) {
            // Status changed
        }
    }
}

// Register in EventServiceProvider
public function boot(): void
{
    Post::observe(PostObserver::class);
}
```

### When to Use Which
- **Observers**: Model-specific lifecycle hooks (created, updated, deleted)
- **Events**: Cross-cutting concerns, business events, async operations

## Notifications

### Use for Multi-Channel Communication
```php
namespace App\Notifications;

use Illuminate\Notifications\Notification;
use Illuminate\Notifications\Messages\MailMessage;

class LowCreditsNotification extends Notification
{
    public function via(object $notifiable): array
    {
        return ['mail', 'database']; // Multiple channels
    }

    public function toMail(object $notifiable): MailMessage
    {
        return (new MailMessage)
            ->subject('Low Credits Alert')
            ->line('Your credits are running low.')
            ->action('Purchase Credits', url('/billing'));
    }

    public function toArray(object $notifiable): array
    {
        return [
            'type' => 'low_credits',
            'remaining' => $this->remainingCredits,
        ];
    }
}

// Send notification
$user->notify(new LowCreditsNotification($credits));
```

## Mail vs Notifications

### Use Mail for System Emails
```php
// Simple transactional emails
Mail::to($user)->send(new WelcomeEmail($user));
```

### Use Notifications for User-Facing Alerts
```php
// Multi-channel (email + database + SMS)
$user->notify(new ImportantAlert($data));
```

**Recommendation**: Prefer Notifications for most use cases (Laravel recommended pattern)

## Console Commands

### Structure
```php
namespace App\Console\Commands;

use Illuminate\Console\Command;

class RefreshSocialTokensCommand extends Command
{
    protected $signature = 'social:refresh-tokens';
    protected $description = 'Refresh expiring social tokens';

    public function handle(): int
    {
        $this->info('Starting...');

        // Your logic

        $this->info('Done!');
        return self::SUCCESS;
    }
}
```

### Best Practices
- Use descriptive signatures: `resource:action`
- Add options and arguments
- Show progress for long operations
- Return proper exit codes (SUCCESS/FAILURE)

## Custom Validation Rules

### Create Reusable Rules
```php
namespace App\Rules;

use Illuminate\Contracts\Validation\ValidationRule;

class ValidTimezone implements ValidationRule
{
    public function validate(string $attribute, mixed $value, Closure $fail): void
    {
        if (!in_array($value, timezone_identifiers_list())) {
            $fail('The :attribute must be a valid timezone.');
        }
    }
}

// Usage
$request->validate([
    'timezone' => ['required', new ValidTimezone],
]);
```

## Complete Flow Example

### Post Publishing Flow
```php
// 1. Controller (HTTP layer)
class PostController extends Controller
{
    public function publish(Post $post)
    {
        $post->update(['status' => PostStatus::PUBLISHING]);
        PublishPostJob::dispatch($post);
        return response()->json(['message' => 'Publishing...']);
    }
}

// 2. Job (Async work)
class PublishPostJob implements ShouldQueue
{
    public function handle()
    {
        // Publish to social media
        $this->post->update(['status' => PostStatus::PUBLISHED]);
    }
}

// 3. Observer (Model lifecycle)
class PostObserver
{
    public function updated(Post $post)
    {
        if ($post->isDirty('status') && $post->status === PostStatus::PUBLISHED) {
            event(new PostPublished($post));
        }
    }
}

// 4. Event (Business event)
class PostPublished
{
    use Dispatchable;

    public function __construct(public Post $post) {}
}

// 5. Listener (Side effects)
class UpdatePostAnalytics implements ShouldQueue
{
    public function handle(PostPublished $event)
    {
        FetchPostMetricsJob::dispatch($event->post)->delay(now()->addHour());
    }
}

// 6. Notification (User alert)
class SendPostPublishedNotification implements ShouldQueue
{
    public function handle(PostPublished $event)
    {
        $event->post->organization->users()->each(fn($user) =>
            $user->notify(new PostPublishedNotification($event->post))
        );
    }
}
```

## Folder Usage Summary

### When to Use Each Folder

| Folder | Use Case | Example |
|--------|----------|---------|
| **Enums** | Fixed sets of values | PostStatus, UserRole |
| **Events** | Business events, cross-cutting concerns | PostPublished, UserRegistered |
| **Listeners** | React to events | SendEmail, UpdateAnalytics |
| **Observers** | Model lifecycle hooks | PostObserver (created, updated) |
| **Mail** | Simple transactional emails | WelcomeEmail, Receipt |
| **Notifications** | Multi-channel user alerts | LowCreditsNotification |
| **Rules** | Custom validation | ValidTimezone, UniqueSlug |
| **Console/Commands** | CLI commands, cron jobs | RefreshTokensCommand |
| **Services** | Business logic | PostService, AIService |
| **Jobs** | Async/queued work | PublishPostJob |

## Conclusion

Following these guidelines ensures:
- Maintainable code
- Consistent style across team
- Better performance
- Secure applications
- Easy testing
- Proper separation of concerns

Remember: **Clean code is not written by following a set of rules. Clean code is written by people who care.**

---

**Last Updated**: December 2025
