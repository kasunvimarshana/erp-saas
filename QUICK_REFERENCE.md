# Quick Reference Guide - ERP SaaS Patterns

## Common Code Patterns

### 1. Creating a New Module

#### File Structure
```
app/Modules/YourModule/
├── Http/Controllers/YourModuleController.php
├── Services/YourModuleService.php
├── Repositories/YourModuleRepository.php
├── Models/YourModel.php
├── DTOs/YourModuleDTO.php
├── Events/YourModuleCreated.php
├── Listeners/HandleYourModuleCreated.php
├── Enums/YourModuleStatus.php
└── Policies/YourModulePolicy.php
```

#### Migration Template
```php
Schema::create('your_models', function (Blueprint $table) {
    $table->id();
    $table->foreignId('tenant_id')->constrained()->cascadeOnDelete();
    $table->string('code')->unique();
    $table->string('name');
    $table->text('description')->nullable();
    $table->boolean('is_active')->default(true);
    $table->timestamps();
    $table->softDeletes();
    
    // Indexes
    $table->index('tenant_id');
    $table->index('code');
    $table->unique(['tenant_id', 'code']);
});
```

#### Model Template
```php
<?php

namespace App\Modules\YourModule\Models;

use App\Traits\TenantScoped;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\SoftDeletes;

class YourModel extends Model
{
    use TenantScoped, SoftDeletes;

    protected $fillable = [
        'tenant_id',
        'code',
        'name',
        'description',
        'is_active',
    ];

    protected $casts = [
        'is_active' => 'boolean',
    ];
}
```

#### Repository Template
```php
<?php

namespace App\Modules\YourModule\Repositories;

use App\Repositories\BaseRepository;
use App\Modules\YourModule\Models\YourModel;

class YourModuleRepository extends BaseRepository
{
    protected function model(): string
    {
        return YourModel::class;
    }

    public function findByCode(string $code): ?YourModel
    {
        return $this->model->where('code', $code)->first();
    }
}
```

#### Service Template
```php
<?php

namespace App\Modules\YourModule\Services;

use App\Services\BaseService;
use App\Modules\YourModule\Repositories\YourModuleRepository;
use App\Modules\YourModule\Models\YourModel;
use App\Modules\YourModule\Events\YourModuleCreated;

class YourModuleService extends BaseService
{
    public function __construct(
        private YourModuleRepository $repository
    ) {}

    public function create(array $data): YourModel
    {
        return $this->transaction(function () use ($data) {
            $model = $this->repository->create($data);
            
            event(new YourModuleCreated($model));
            
            return $model;
        });
    }
}
```

#### Controller Template
```php
<?php

namespace App\Modules\YourModule\Http\Controllers;

use App\Http\Controllers\Api\BaseController;
use App\Modules\YourModule\Services\YourModuleService;
use App\Modules\YourModule\Http\Requests\StoreYourModuleRequest;
use Illuminate\Http\JsonResponse;

class YourModuleController extends BaseController
{
    public function __construct(
        private YourModuleService $service
    ) {}

    public function index(): JsonResponse
    {
        $items = $this->service->getAll();
        return $this->successResponse($items);
    }

    public function store(StoreYourModuleRequest $request): JsonResponse
    {
        $item = $this->service->create($request->validated());
        return $this->successResponse($item, 'Created successfully', 201);
    }

    public function show(int $id): JsonResponse
    {
        $item = $this->service->find($id);
        return $item 
            ? $this->successResponse($item)
            : $this->errorResponse('Not found', null, 404);
    }

    public function update(UpdateYourModuleRequest $request, int $id): JsonResponse
    {
        $item = $this->service->update($id, $request->validated());
        return $this->successResponse($item, 'Updated successfully');
    }

    public function destroy(int $id): JsonResponse
    {
        $this->service->delete($id);
        return $this->successResponse(null, 'Deleted successfully');
    }
}
```

### 2. Creating an Event and Listener

#### Event
```php
<?php

namespace App\Modules\YourModule\Events;

use App\Modules\YourModule\Models\YourModel;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class YourModuleCreated
{
    use Dispatchable, SerializesModels;

    public function __construct(
        public YourModel $model
    ) {}
}
```

#### Listener
```php
<?php

namespace App\Modules\YourModule\Listeners;

use App\Modules\YourModule\Events\YourModuleCreated;
use Illuminate\Contracts\Queue\ShouldQueue;

class HandleYourModuleCreated implements ShouldQueue
{
    public function handle(YourModuleCreated $event): void
    {
        // Handle the event
        // E.g., send notification, update cache, etc.
    }
}
```

#### Registration (EventServiceProvider)
```php
protected $listen = [
    YourModuleCreated::class => [
        HandleYourModuleCreated::class,
    ],
];
```

### 3. Creating a Feature Test

```php
<?php

namespace Tests\Feature\Modules\YourModule;

use Tests\TestCase;
use App\Models\User;
use App\Models\Tenant;
use App\Modules\YourModule\Models\YourModel;
use Illuminate\Foundation\Testing\RefreshDatabase;

class YourModuleApiTest extends TestCase
{
    use RefreshDatabase;

    private User $user;
    private Tenant $tenant;

    protected function setUp(): void
    {
        parent::setUp();
        
        $this->tenant = Tenant::factory()->create();
        $this->user = User::factory()->create(['tenant_id' => $this->tenant->id]);
    }

    public function test_can_list_items(): void
    {
        YourModel::factory()->count(3)->create(['tenant_id' => $this->tenant->id]);
        
        $response = $this->actingAs($this->user)
            ->getJson('/api/v1/your-module');
        
        $response->assertStatus(200)
            ->assertJsonCount(3, 'data');
    }

    public function test_can_create_item(): void
    {
        $data = [
            'code' => 'TEST-001',
            'name' => 'Test Item',
        ];
        
        $response = $this->actingAs($this->user)
            ->postJson('/api/v1/your-module', $data);
        
        $response->assertStatus(201)
            ->assertJsonPath('data.code', 'TEST-001');
        
        $this->assertDatabaseHas('your_models', [
            'code' => 'TEST-001',
            'tenant_id' => $this->tenant->id,
        ]);
    }

    public function test_tenant_isolation(): void
    {
        $tenant2 = Tenant::factory()->create();
        $user2 = User::factory()->create(['tenant_id' => $tenant2->id]);
        
        $item = YourModel::factory()->create(['tenant_id' => $this->tenant->id]);
        
        // User 1 can access
        $this->actingAs($this->user)
            ->getJson("/api/v1/your-module/{$item->id}")
            ->assertStatus(200);
        
        // User 2 cannot access
        $this->actingAs($user2)
            ->getJson("/api/v1/your-module/{$item->id}")
            ->assertStatus(404);
    }
}
```

## Common Commands

### Development

```bash
# Start development server
php artisan serve

# Run migrations
php artisan migrate

# Seed database
php artisan db:seed

# Create migration
php artisan make:migration create_your_table --create=your_table

# Create model with migration
php artisan make:model YourModel -m

# Create controller
php artisan make:controller Api/YourController --api

# Create seeder
php artisan make:seeder YourSeeder

# Create factory
php artisan make:factory YourFactory

# Create test
php artisan make:test YourTest --unit
php artisan make:test YourFeatureTest
```

### Testing

```bash
# Run all tests
php artisan test

# Run specific test
php artisan test --filter YourTest

# Run with coverage
php artisan test --coverage

# Run parallel tests
php artisan test --parallel
```

### Queue Management

```bash
# Process queue jobs
php artisan queue:work

# Process specific queue
php artisan queue:work redis --queue=high,default

# List failed jobs
php artisan queue:failed

# Retry failed job
php artisan queue:retry <job-id>

# Retry all failed jobs
php artisan queue:retry all
```

### Cache Management

```bash
# Clear all caches
php artisan cache:clear

# Clear configuration cache
php artisan config:clear

# Clear route cache
php artisan route:clear

# Clear view cache
php artisan view:clear

# Optimize application
php artisan optimize

# Cache configuration
php artisan config:cache

# Cache routes
php artisan route:cache
```

## Environment Configuration Checklist

### Development (.env.local)
```env
APP_ENV=local
APP_DEBUG=true
APP_URL=http://localhost:8000

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=erp_dev
DB_USERNAME=root
DB_PASSWORD=

CACHE_DRIVER=redis
QUEUE_CONNECTION=sync  # Use sync for development
SESSION_DRIVER=file

MAIL_MAILER=log
```

### Testing (.env.testing)
```env
APP_ENV=testing
APP_DEBUG=true

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=erp_testing
DB_USERNAME=root
DB_PASSWORD=

CACHE_DRIVER=array
QUEUE_CONNECTION=sync
SESSION_DRIVER=array
```

### Production (.env)
```env
APP_ENV=production
APP_DEBUG=false
APP_URL=https://your-domain.com

DB_CONNECTION=mysql
DB_HOST=your-db-host
DB_PORT=3306
DB_DATABASE=erp_production
DB_USERNAME=your-db-user
DB_PASSWORD=your-secure-password

CACHE_DRIVER=redis
QUEUE_CONNECTION=redis
SESSION_DRIVER=redis

REDIS_HOST=your-redis-host
REDIS_PASSWORD=your-redis-password

MAIL_MAILER=smtp
MAIL_HOST=your-mail-host
MAIL_PORT=587
MAIL_USERNAME=your-mail-username
MAIL_PASSWORD=your-mail-password
MAIL_ENCRYPTION=tls
```

## API Response Formats

### Success Response
```json
{
  "success": true,
  "message": "Operation successful",
  "data": {
    "id": 1,
    "code": "ITEM-001",
    "name": "Item Name"
  },
  "meta": {
    "timestamp": "2026-02-04T18:57:01Z"
  }
}
```

### Error Response
```json
{
  "success": false,
  "message": "Validation error",
  "errors": {
    "code": ["The code has already been taken."],
    "name": ["The name field is required."]
  },
  "meta": {
    "timestamp": "2026-02-04T18:57:01Z"
  }
}
```

### Paginated Response
```json
{
  "success": true,
  "message": "Items retrieved successfully",
  "data": [
    {"id": 1, "name": "Item 1"},
    {"id": 2, "name": "Item 2"}
  ],
  "meta": {
    "current_page": 1,
    "per_page": 15,
    "total": 100,
    "last_page": 7,
    "timestamp": "2026-02-04T18:57:01Z"
  },
  "links": {
    "first": "http://api.example.com/items?page=1",
    "last": "http://api.example.com/items?page=7",
    "prev": null,
    "next": "http://api.example.com/items?page=2"
  }
}
```

## Common Database Queries

### Basic CRUD
```php
// Create
$model = YourModel::create($data);

// Read
$model = YourModel::find($id);
$models = YourModel::all();
$models = YourModel::where('is_active', true)->get();

// Update
$model->update($data);
YourModel::where('id', $id)->update($data);

// Delete
$model->delete(); // Soft delete
$model->forceDelete(); // Permanent delete

// Restore soft deleted
$model->restore();
```

### Eager Loading
```php
// Prevent N+1 queries
$products = Product::with(['category', 'stockLedger'])->get();

// Nested relationships
$orders = Order::with(['customer.contacts', 'items.product'])->get();

// Conditional eager loading
$products = Product::with([
    'stockLedger' => function ($query) {
        $query->where('quantity', '>', 0);
    }
])->get();
```

### Pagination
```php
// Simple pagination
$items = YourModel::paginate(15);

// Custom pagination
$items = YourModel::paginate(
    $perPage = 15,
    $columns = ['*'],
    $pageName = 'page',
    $page = null
);

// Cursor pagination (for large datasets)
$items = YourModel::cursorPaginate(15);
```

### Transactions
```php
use Illuminate\Support\Facades\DB;

DB::transaction(function () {
    $product = Product::create($productData);
    $stockLedger = StockLedger::create($stockData);
    
    // If any exception occurs, both will rollback
});

// Manual transaction control
DB::beginTransaction();
try {
    // Operations
    DB::commit();
} catch (\Exception $e) {
    DB::rollBack();
    throw $e;
}
```

## Security Best Practices Checklist

### Input Validation
- ✅ Use Form Requests for validation
- ✅ Validate all user input
- ✅ Sanitize data before database operations
- ✅ Use type hints in PHP 8.3+

### Authentication & Authorization
- ✅ Use Laravel Sanctum for API authentication
- ✅ Implement RBAC with spatie/laravel-permission
- ✅ Check permissions in controllers/policies
- ✅ Verify tenant_id on all operations

### Multi-Tenancy
- ✅ Add tenant_id to all tables
- ✅ Use TenantScoped trait on all models
- ✅ Test tenant isolation thoroughly
- ✅ Never trust client-provided tenant_id

### Data Protection
- ✅ Use HTTPS in production
- ✅ Enable CSRF protection
- ✅ Hash passwords with bcrypt
- ✅ Never log sensitive data

### API Security
- ✅ Implement rate limiting
- ✅ Use API versioning
- ✅ Return appropriate HTTP status codes
- ✅ Don't expose internal errors to clients

## Performance Optimization Checklist

### Database
- ✅ Index frequently queried columns
- ✅ Use eager loading to prevent N+1
- ✅ Optimize queries with query builder
- ✅ Use database transactions appropriately
- ✅ Implement database connection pooling

### Caching
- ✅ Cache frequently accessed data
- ✅ Use Redis for application cache
- ✅ Implement cache tags for easy invalidation
- ✅ Cache API responses where appropriate
- ✅ Use query caching

### Application
- ✅ Use queues for long-running tasks
- ✅ Optimize autoloader with Composer
- ✅ Cache configuration and routes
- ✅ Use lazy loading where appropriate
- ✅ Minimize middleware overhead

### Frontend
- ✅ Implement lazy loading for components
- ✅ Use Vite for optimal bundling
- ✅ Implement code splitting
- ✅ Optimize images and assets
- ✅ Use CDN for static assets

## Troubleshooting Guide

### Common Issues

#### 1. Tenant Isolation Not Working
```php
// Check if TenantScoped trait is used
class Product extends Model
{
    use TenantScoped; // Make sure this is present
}

// Check if user has tenant_id
if (!auth()->user()->tenant_id) {
    // User needs tenant assignment
}

// Manually verify in query
$products = Product::where('tenant_id', auth()->user()->tenant_id)->get();
```

#### 2. N+1 Query Problems
```php
// Bad - causes N+1
foreach ($products as $product) {
    echo $product->category->name; // Query for each iteration
}

// Good - eager loading
$products = Product::with('category')->get();
foreach ($products as $product) {
    echo $product->category->name; // No additional queries
}

// Debug queries
\DB::enableQueryLog();
// Your code here
dd(\DB::getQueryLog());
```

#### 3. Queue Jobs Not Processing
```bash
# Check queue status
php artisan queue:work --once

# Check failed jobs
php artisan queue:failed

# Retry failed jobs
php artisan queue:retry all

# Check supervisor status
sudo supervisorctl status
sudo supervisorctl restart laravel-worker:*
```

#### 4. Cache Not Invalidating
```php
// Clear specific cache
Cache::forget('key');

// Clear cache tags
Cache::tags(['products'])->flush();

// Clear all cache
Cache::flush();

// In production, use:
php artisan cache:clear
```

## Git Workflow

### Branch Strategy
```bash
# Main branches
main          # Production-ready code
develop       # Development branch

# Feature branches
feature/module-name
bugfix/issue-description
hotfix/critical-fix
```

### Common Git Commands
```bash
# Create feature branch
git checkout -b feature/inventory-module

# Commit changes
git add .
git commit -m "Add inventory module with CRUD operations"

# Push branch
git push origin feature/inventory-module

# Update from main
git checkout main
git pull origin main
git checkout feature/inventory-module
git merge main

# Squash commits before merge
git rebase -i HEAD~3
```

## Useful Resources

### Documentation
- Laravel: https://laravel.com/docs
- Vue.js: https://vuejs.org/guide
- Pinia: https://pinia.vuejs.org
- Sanctum: https://laravel.com/docs/sanctum

### Tools
- Postman: API testing
- TablePlus: Database management
- Redis Commander: Redis management
- Laravel Telescope: Debugging
- Laravel Debugbar: Development debugging

---

**Quick Reference Version**: 1.0  
**Last Updated**: February 4, 2026  
**Repository**: kasunvimarshana/erp-saas
