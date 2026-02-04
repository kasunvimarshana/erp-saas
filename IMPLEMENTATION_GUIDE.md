# ERP SaaS Implementation Guide

## Introduction

This guide provides step-by-step instructions for implementing an ERP SaaS platform based on the architectural patterns and best practices observed in the **multi-x-erp-saas** repository.

## Table of Contents

1. [Project Setup](#project-setup)
2. [Core Architecture Setup](#core-architecture-setup)
3. [Multi-Tenancy Implementation](#multi-tenancy-implementation)
4. [Module Development](#module-development)
5. [API Development](#api-development)
6. [Testing Strategy](#testing-strategy)
7. [Security Implementation](#security-implementation)
8. [Performance Optimization](#performance-optimization)
9. [Deployment](#deployment)

## Project Setup

### Prerequisites

- PHP 8.3+
- Composer
- Node.js 20+
- MySQL 8.0+ or PostgreSQL 13+
- Redis (for caching and queues)

### Step 1: Create Laravel Project

```bash
composer create-project laravel/laravel erp-saas
cd erp-saas
```

### Step 2: Install Essential Packages

```bash
# Backend dependencies
composer require laravel/sanctum
composer require spatie/laravel-permission
composer require barryvdh/laravel-debugbar --dev

# Frontend setup
npm install
npm install vue@3 vue-router@4 pinia axios
npm install -D vite @vitejs/plugin-vue
```

### Step 3: Configure Environment

```env
APP_NAME="ERP SaaS"
APP_ENV=local
APP_DEBUG=true
APP_URL=http://localhost:8000

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=erp_saas
DB_USERNAME=your_username
DB_PASSWORD=your_password

CACHE_DRIVER=redis
QUEUE_CONNECTION=redis
SESSION_DRIVER=redis

SANCTUM_STATEFUL_DOMAINS=localhost:5173
```

## Core Architecture Setup

### Step 1: Create Base Repository

```bash
php artisan make:interface Contracts/RepositoryInterface
```

```php
// app/Contracts/RepositoryInterface.php
<?php

namespace App\Contracts;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Pagination\LengthAwarePaginator;
use Illuminate\Support\Collection;

interface RepositoryInterface
{
    public function all(): Collection;
    public function find(int $id): ?Model;
    public function create(array $data): Model;
    public function update(int $id, array $data): bool;
    public function delete(int $id): bool;
    public function paginate(int $perPage = 15): LengthAwarePaginator;
}
```

```php
// app/Repositories/BaseRepository.php
<?php

namespace App\Repositories;

use App\Contracts\RepositoryInterface;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Pagination\LengthAwarePaginator;
use Illuminate\Support\Collection;

abstract class BaseRepository implements RepositoryInterface
{
    protected Model $model;

    public function __construct()
    {
        $this->model = $this->makeModel();
    }

    abstract protected function model(): string;

    protected function makeModel(): Model
    {
        return app($this->model());
    }

    public function all(): Collection
    {
        return $this->model->all();
    }

    public function find(int $id): ?Model
    {
        return $this->model->find($id);
    }

    public function create(array $data): Model
    {
        return $this->model->create($data);
    }

    public function update(int $id, array $data): bool
    {
        $model = $this->find($id);
        
        if (!$model) {
            return false;
        }
        
        return $model->update($data);
    }

    public function delete(int $id): bool
    {
        $model = $this->find($id);
        
        if (!$model) {
            return false;
        }
        
        return $model->delete();
    }

    public function paginate(int $perPage = 15): LengthAwarePaginator
    {
        return $this->model->paginate($perPage);
    }
}
```

### Step 2: Create Base Service

```php
// app/Services/BaseService.php
<?php

namespace App\Services;

use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Log;

abstract class BaseService
{
    protected function transaction(callable $callback)
    {
        try {
            DB::beginTransaction();
            
            $result = $callback();
            
            DB::commit();
            
            return $result;
        } catch (\Exception $e) {
            DB::rollBack();
            
            Log::error('Transaction failed', [
                'service' => get_class($this),
                'error' => $e->getMessage(),
                'trace' => $e->getTraceAsString(),
            ]);
            
            throw $e;
        }
    }
}
```

### Step 3: Create Base Controller

```php
// app/Http/Controllers/Api/BaseController.php
<?php

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use Illuminate\Http\JsonResponse;

abstract class BaseController extends Controller
{
    protected function successResponse(
        $data = null,
        string $message = 'Success',
        int $code = 200
    ): JsonResponse {
        return response()->json([
            'success' => true,
            'message' => $message,
            'data' => $data,
            'meta' => [
                'timestamp' => now()->toIso8601String(),
            ],
        ], $code);
    }

    protected function errorResponse(
        string $message = 'Error',
        $errors = null,
        int $code = 400
    ): JsonResponse {
        return response()->json([
            'success' => false,
            'message' => $message,
            'errors' => $errors,
            'meta' => [
                'timestamp' => now()->toIso8601String(),
            ],
        ], $code);
    }
}
```

## Multi-Tenancy Implementation

### Step 1: Create Tenant Model and Migration

```bash
php artisan make:model Tenant -m
```

```php
// database/migrations/xxxx_create_tenants_table.php
public function up()
{
    Schema::create('tenants', function (Blueprint $table) {
        $table->id();
        $table->string('name');
        $table->string('slug')->unique();
        $table->string('domain')->unique()->nullable();
        $table->json('settings')->nullable();
        $table->boolean('is_active')->default(true);
        $table->timestamps();
        $table->softDeletes();
    });
}
```

### Step 2: Create TenantScoped Trait

```php
// app/Traits/TenantScoped.php
<?php

namespace App\Traits;

use Illuminate\Database\Eloquent\Builder;

trait TenantScoped
{
    protected static function bootTenantScoped()
    {
        // Apply global scope for queries
        static::addGlobalScope('tenant', function (Builder $builder) {
            if (auth()->check() && auth()->user()->tenant_id) {
                $builder->where(
                    $builder->getModel()->getTable() . '.tenant_id',
                    auth()->user()->tenant_id
                );
            }
        });

        // Automatically set tenant_id on create
        static::creating(function ($model) {
            if (auth()->check() && !$model->tenant_id) {
                $model->tenant_id = auth()->user()->tenant_id;
            }
        });
    }

    public function tenant()
    {
        return $this->belongsTo(\App\Models\Tenant::class);
    }
}
```

### Step 3: Update User Model

```php
// app/Models/User.php
public function up()
{
    Schema::table('users', function (Blueprint $table) {
        $table->foreignId('tenant_id')->after('id')->constrained()->cascadeOnDelete();
        $table->index('tenant_id');
    });
}
```

## Module Development

### Step 1: Create Module Structure

```bash
# Create module directories
mkdir -p app/Modules/Inventory/{Http/Controllers,Services,Repositories,Models,DTOs,Events,Listeners,Enums,Policies}
```

### Step 2: Create Product Module Example

```php
// app/Modules/Inventory/Models/Product.php
<?php

namespace App\Modules\Inventory\Models;

use App\Traits\TenantScoped;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\SoftDeletes;

class Product extends Model
{
    use TenantScoped, SoftDeletes;

    protected $fillable = [
        'tenant_id',
        'code',
        'name',
        'description',
        'type',
        'category_id',
        'buying_price',
        'selling_price',
        'stock_quantity',
        'reorder_level',
        'is_active',
    ];

    protected $casts = [
        'buying_price' => 'decimal:2',
        'selling_price' => 'decimal:2',
        'stock_quantity' => 'integer',
        'reorder_level' => 'integer',
        'is_active' => 'boolean',
    ];

    public function category()
    {
        return $this->belongsTo(Category::class);
    }

    public function stockLedger()
    {
        return $this->hasMany(StockLedger::class);
    }
}
```

```php
// app/Modules/Inventory/Repositories/ProductRepository.php
<?php

namespace App\Modules\Inventory\Repositories;

use App\Repositories\BaseRepository;
use App\Modules\Inventory\Models\Product;
use Illuminate\Support\Collection;

class ProductRepository extends BaseRepository
{
    protected function model(): string
    {
        return Product::class;
    }

    public function findByCode(string $code): ?Product
    {
        return $this->model->where('code', $code)->first();
    }

    public function search(string $query): Collection
    {
        return $this->model
            ->where('name', 'like', "%{$query}%")
            ->orWhere('code', 'like', "%{$query}%")
            ->orWhere('description', 'like', "%{$query}%")
            ->get();
    }

    public function belowReorderLevel(): Collection
    {
        return $this->model
            ->whereColumn('stock_quantity', '<=', 'reorder_level')
            ->where('is_active', true)
            ->get();
    }
}
```

```php
// app/Modules/Inventory/Services/ProductService.php
<?php

namespace App\Modules\Inventory\Services;

use App\Services\BaseService;
use App\Modules\Inventory\Repositories\ProductRepository;
use App\Modules\Inventory\Models\Product;
use App\Modules\Inventory\Events\ProductCreated;

class ProductService extends BaseService
{
    public function __construct(
        private ProductRepository $repository
    ) {}

    public function createProduct(array $data): Product
    {
        return $this->transaction(function () use ($data) {
            $product = $this->repository->create($data);
            
            event(new ProductCreated($product));
            
            return $product;
        });
    }

    public function updateProduct(int $id, array $data): Product
    {
        return $this->transaction(function () use ($id, $data) {
            $this->repository->update($id, $data);
            $product = $this->repository->find($id);
            
            event(new ProductUpdated($product));
            
            return $product;
        });
    }

    public function deleteProduct(int $id): bool
    {
        return $this->transaction(function () use ($id) {
            $product = $this->repository->find($id);
            
            if (!$product) {
                throw new \Exception('Product not found');
            }
            
            $result = $this->repository->delete($id);
            
            event(new ProductDeleted($product));
            
            return $result;
        });
    }
}
```

```php
// app/Modules/Inventory/Http/Controllers/ProductController.php
<?php

namespace App\Modules\Inventory\Http\Controllers;

use App\Http\Controllers\Api\BaseController;
use App\Modules\Inventory\Services\ProductService;
use App\Modules\Inventory\Http\Requests\StoreProductRequest;
use App\Modules\Inventory\Http\Requests\UpdateProductRequest;
use Illuminate\Http\JsonResponse;

class ProductController extends BaseController
{
    public function __construct(
        private ProductService $service
    ) {}

    public function index(): JsonResponse
    {
        $products = $this->service->getAllProducts();
        
        return $this->successResponse($products, 'Products retrieved successfully');
    }

    public function store(StoreProductRequest $request): JsonResponse
    {
        $product = $this->service->createProduct($request->validated());
        
        return $this->successResponse($product, 'Product created successfully', 201);
    }

    public function show(int $id): JsonResponse
    {
        $product = $this->service->getProduct($id);
        
        if (!$product) {
            return $this->errorResponse('Product not found', null, 404);
        }
        
        return $this->successResponse($product);
    }

    public function update(UpdateProductRequest $request, int $id): JsonResponse
    {
        $product = $this->service->updateProduct($id, $request->validated());
        
        return $this->successResponse($product, 'Product updated successfully');
    }

    public function destroy(int $id): JsonResponse
    {
        $result = $this->service->deleteProduct($id);
        
        return $this->successResponse(null, 'Product deleted successfully');
    }
}
```

### Step 3: Create Enum for Product Types

```php
// app/Modules/Inventory/Enums/ProductType.php
<?php

namespace App\Modules\Inventory\Enums;

enum ProductType: string
{
    case INVENTORY = 'inventory';
    case SERVICE = 'service';
    case COMBO = 'combo';
    case BUNDLE = 'bundle';

    public function label(): string
    {
        return match($this) {
            self::INVENTORY => 'Inventory Item',
            self::SERVICE => 'Service',
            self::COMBO => 'Combo Product',
            self::BUNDLE => 'Bundle Product',
        };
    }
}
```

## API Development

### Step 1: Create API Routes

```php
// routes/api.php
use Illuminate\Support\Facades\Route;
use App\Modules\Inventory\Http\Controllers\ProductController;

Route::prefix('v1')->group(function () {
    // Public routes
    Route::post('/auth/login', [AuthController::class, 'login']);
    Route::post('/auth/register', [AuthController::class, 'register']);
    
    // Protected routes
    Route::middleware('auth:sanctum')->group(function () {
        // Auth routes
        Route::get('/auth/user', [AuthController::class, 'user']);
        Route::post('/auth/logout', [AuthController::class, 'logout']);
        
        // Inventory routes
        Route::prefix('inventory')->group(function () {
            Route::apiResource('products', ProductController::class);
            Route::get('products/search', [ProductController::class, 'search']);
            Route::get('products/below-reorder-level', [ProductController::class, 'belowReorderLevel']);
        });
    });
});
```

### Step 2: Create Request Validation

```php
// app/Modules/Inventory/Http/Requests/StoreProductRequest.php
<?php

namespace App\Modules\Inventory\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class StoreProductRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true;
    }

    public function rules(): array
    {
        return [
            'code' => 'required|string|max:50|unique:products,code',
            'name' => 'required|string|max:255',
            'description' => 'nullable|string',
            'type' => 'required|in:inventory,service,combo,bundle',
            'category_id' => 'required|exists:categories,id',
            'buying_price' => 'required|numeric|min:0',
            'selling_price' => 'required|numeric|min:0',
            'reorder_level' => 'nullable|integer|min:0',
            'is_active' => 'boolean',
        ];
    }

    public function messages(): array
    {
        return [
            'code.unique' => 'A product with this code already exists.',
            'category_id.exists' => 'The selected category does not exist.',
            'selling_price.min' => 'Selling price must be greater than or equal to 0.',
        ];
    }
}
```

## Testing Strategy

### Step 1: Create Feature Test

```php
// tests/Feature/Modules/Inventory/ProductApiTest.php
<?php

namespace Tests\Feature\Modules\Inventory;

use Tests\TestCase;
use App\Models\User;
use App\Models\Tenant;
use App\Modules\Inventory\Models\Product;
use App\Modules\Inventory\Models\Category;
use Illuminate\Foundation\Testing\RefreshDatabase;

class ProductApiTest extends TestCase
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

    public function test_can_list_products(): void
    {
        Product::factory()->count(3)->create(['tenant_id' => $this->tenant->id]);
        
        $response = $this->actingAs($this->user)
            ->getJson('/api/v1/inventory/products');
        
        $response->assertStatus(200)
            ->assertJsonStructure([
                'success',
                'message',
                'data' => [
                    '*' => ['id', 'code', 'name', 'type']
                ]
            ])
            ->assertJsonCount(3, 'data');
    }

    public function test_can_create_product(): void
    {
        $category = Category::factory()->create(['tenant_id' => $this->tenant->id]);
        
        $productData = [
            'code' => 'PROD-001',
            'name' => 'Test Product',
            'type' => 'inventory',
            'category_id' => $category->id,
            'buying_price' => 100,
            'selling_price' => 150,
        ];
        
        $response = $this->actingAs($this->user)
            ->postJson('/api/v1/inventory/products', $productData);
        
        $response->assertStatus(201)
            ->assertJsonPath('success', true)
            ->assertJsonPath('data.code', 'PROD-001');
        
        $this->assertDatabaseHas('products', [
            'code' => 'PROD-001',
            'tenant_id' => $this->tenant->id,
        ]);
    }

    public function test_cannot_create_product_with_duplicate_code(): void
    {
        Product::factory()->create([
            'code' => 'PROD-001',
            'tenant_id' => $this->tenant->id,
        ]);
        
        $category = Category::factory()->create(['tenant_id' => $this->tenant->id]);
        
        $response = $this->actingAs($this->user)
            ->postJson('/api/v1/inventory/products', [
                'code' => 'PROD-001',
                'name' => 'Test Product',
                'type' => 'inventory',
                'category_id' => $category->id,
                'buying_price' => 100,
                'selling_price' => 150,
            ]);
        
        $response->assertStatus(422)
            ->assertJsonValidationErrors(['code']);
    }

    public function test_tenant_isolation(): void
    {
        $tenant2 = Tenant::factory()->create();
        $user2 = User::factory()->create(['tenant_id' => $tenant2->id]);
        
        $product = Product::factory()->create(['tenant_id' => $this->tenant->id]);
        
        // User from tenant 1 can access
        $this->actingAs($this->user)
            ->getJson("/api/v1/inventory/products/{$product->id}")
            ->assertStatus(200);
        
        // User from tenant 2 cannot access
        $this->actingAs($user2)
            ->getJson("/api/v1/inventory/products/{$product->id}")
            ->assertStatus(404);
    }
}
```

### Step 2: Create Unit Test

```php
// tests/Unit/Services/ProductServiceTest.php
<?php

namespace Tests\Unit\Services;

use Tests\TestCase;
use App\Modules\Inventory\Services\ProductService;
use App\Modules\Inventory\Repositories\ProductRepository;
use App\Modules\Inventory\Models\Product;
use Mockery;

class ProductServiceTest extends TestCase
{
    public function test_creates_product_successfully(): void
    {
        $repository = Mockery::mock(ProductRepository::class);
        
        $productData = [
            'code' => 'PROD-001',
            'name' => 'Test Product',
            'type' => 'inventory',
        ];
        
        $expectedProduct = Product::make(array_merge($productData, ['id' => 1]));
        
        $repository->shouldReceive('create')
            ->once()
            ->with($productData)
            ->andReturn($expectedProduct);
        
        $service = new ProductService($repository);
        $product = $service->createProduct($productData);
        
        $this->assertEquals('PROD-001', $product->code);
        $this->assertEquals('Test Product', $product->name);
    }

    protected function tearDown(): void
    {
        Mockery::close();
        parent::tearDown();
    }
}
```

## Security Implementation

### Step 1: Implement Authentication

```php
// app/Http/Controllers/Api/AuthController.php
<?php

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Models\User;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\Hash;

class AuthController extends Controller
{
    public function register(Request $request): JsonResponse
    {
        $validated = $request->validate([
            'name' => 'required|string|max:255',
            'email' => 'required|email|unique:users',
            'password' => 'required|min:8|confirmed',
            'tenant_id' => 'required|exists:tenants,id',
        ]);

        $user = User::create([
            'name' => $validated['name'],
            'email' => $validated['email'],
            'password' => Hash::make($validated['password']),
            'tenant_id' => $validated['tenant_id'],
        ]);

        $token = $user->createToken('api-token')->plainTextToken;

        return response()->json([
            'success' => true,
            'message' => 'User registered successfully',
            'token' => $token,
            'user' => $user,
        ], 201);
    }

    public function login(Request $request): JsonResponse
    {
        $credentials = $request->validate([
            'email' => 'required|email',
            'password' => 'required',
        ]);

        if (!Auth::attempt($credentials)) {
            return response()->json([
                'success' => false,
                'message' => 'Invalid credentials',
            ], 401);
        }

        $user = Auth::user();
        $token = $user->createToken('api-token')->plainTextToken;

        return response()->json([
            'success' => true,
            'message' => 'Login successful',
            'token' => $token,
            'user' => $user,
        ]);
    }

    public function logout(Request $request): JsonResponse
    {
        $request->user()->currentAccessToken()->delete();

        return response()->json([
            'success' => true,
            'message' => 'Logged out successfully',
        ]);
    }

    public function user(Request $request): JsonResponse
    {
        return response()->json([
            'success' => true,
            'data' => $request->user(),
        ]);
    }
}
```

### Step 2: Implement Authorization (RBAC)

```bash
composer require spatie/laravel-permission
php artisan vendor:publish --provider="Spatie\Permission\PermissionServiceProvider"
php artisan migrate
```

```php
// app/Models/User.php
use Spatie\Permission\Traits\HasRoles;

class User extends Authenticatable
{
    use HasRoles;
    // ...
}
```

```php
// Create permissions seeder
php artisan make:seeder PermissionsSeeder
```

```php
// database/seeders/PermissionsSeeder.php
<?php

namespace Database\Seeders;

use Illuminate\Database\Seeder;
use Spatie\Permission\Models\Role;
use Spatie\Permission\Models\Permission;

class PermissionsSeeder extends Seeder
{
    public function run(): void
    {
        // Create permissions
        $permissions = [
            'products.view',
            'products.create',
            'products.update',
            'products.delete',
        ];

        foreach ($permissions as $permission) {
            Permission::create(['name' => $permission]);
        }

        // Create roles
        $admin = Role::create(['name' => 'admin']);
        $manager = Role::create(['name' => 'manager']);
        $staff = Role::create(['name' => 'staff']);

        // Assign permissions
        $admin->givePermissionTo(Permission::all());
        $manager->givePermissionTo(['products.view', 'products.create', 'products.update']);
        $staff->givePermissionTo(['products.view']);
    }
}
```

```php
// Use in controller
public function store(StoreProductRequest $request): JsonResponse
{
    $this->authorize('products.create');
    
    $product = $this->service->createProduct($request->validated());
    
    return $this->successResponse($product, 'Product created successfully', 201);
}
```

## Performance Optimization

### Step 1: Implement Caching

```php
// app/Modules/Inventory/Repositories/ProductRepository.php
use Illuminate\Support\Facades\Cache;

public function find(int $id): ?Product
{
    return Cache::tags(['products'])
        ->remember(
            "product:{$id}",
            now()->addHours(24),
            fn() => $this->model->find($id)
        );
}

public function create(array $data): Product
{
    $product = parent::create($data);
    
    // Invalidate cache
    Cache::tags(['products'])->flush();
    
    return $product;
}
```

### Step 2: Implement Query Optimization

```php
// Eager loading to prevent N+1
public function all(): Collection
{
    return $this->model
        ->with(['category', 'stockLedger'])
        ->get();
}

// Add indexes in migration
public function up()
{
    Schema::table('products', function (Blueprint $table) {
        $table->index('code');
        $table->index('tenant_id');
        $table->index(['tenant_id', 'code']);
        $table->index(['tenant_id', 'category_id']);
    });
}
```

### Step 3: Implement Queue Processing

```php
// Create a job
php artisan make:job ProcessBulkProductImport
```

```php
// app/Jobs/ProcessBulkProductImport.php
<?php

namespace App\Jobs;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;

class ProcessBulkProductImport implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public $tries = 3;
    public $timeout = 300;

    public function __construct(
        public array $products
    ) {}

    public function handle(): void
    {
        foreach ($this->products as $productData) {
            Product::create($productData);
        }
    }
}
```

```php
// Dispatch the job
ProcessBulkProductImport::dispatch($productsArray);
```

## Deployment

### Step 1: Production Environment Setup

```env
APP_ENV=production
APP_DEBUG=false
APP_URL=https://your-domain.com

DB_CONNECTION=mysql
DB_HOST=your-db-host
DB_DATABASE=your-db-name
DB_USERNAME=your-db-user
DB_PASSWORD=your-secure-password

CACHE_DRIVER=redis
QUEUE_CONNECTION=redis
SESSION_DRIVER=redis

REDIS_HOST=your-redis-host
REDIS_PASSWORD=your-redis-password
```

### Step 2: Optimize Application

```bash
# Optimize autoloader
composer install --optimize-autoloader --no-dev

# Cache configuration
php artisan config:cache

# Cache routes
php artisan route:cache

# Cache views
php artisan view:cache

# Optimize
php artisan optimize
```

### Step 3: Set Up Queue Workers

```bash
# Using Supervisor
sudo apt-get install supervisor

# Create supervisor config
sudo nano /etc/supervisor/conf.d/laravel-worker.conf
```

```ini
[program:laravel-worker]
process_name=%(program_name)s_%(process_num)02d
command=php /path/to/your/app/artisan queue:work redis --sleep=3 --tries=3 --max-time=3600
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
user=www-data
numprocs=8
redirect_stderr=true
stdout_logfile=/path/to/your/app/storage/logs/worker.log
stopwaitsecs=3600
```

```bash
# Start supervisor
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl start laravel-worker:*
```

### Step 4: Configure Web Server (Nginx)

```nginx
server {
    listen 80;
    server_name your-domain.com;
    root /path/to/your/app/public;

    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";

    index index.php;

    charset utf-8;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    error_page 404 /index.php;

    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php8.3-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```

## Conclusion

This implementation guide provides a comprehensive foundation for building an ERP SaaS platform following the architectural patterns from multi-x-erp-saas. Follow these steps sequentially for best results.

### Next Steps

1. Implement remaining modules (CRM, Procurement, POS)
2. Add comprehensive test coverage
3. Implement advanced features (reporting, analytics)
4. Set up CI/CD pipeline
5. Configure monitoring and logging
6. Implement backup strategy

---

**Document Version**: 1.0  
**Last Updated**: February 4, 2026  
**Repository**: kasunvimarshana/erp-saas
