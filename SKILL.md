---
name: tenancy-development
description: "Develops multi-tenant Laravel applications with stancl/tenancy v3. Activates when working with tenant models, tenant identification (domain/subdomain/path/request-data), multi-database or single-database tenancy, tenant migrations, tenant-aware queues, jobs, caches, filesystems, Redis, console commands (tenants:migrate, tenants:seed, tenants:run), bootstrappers, features, event system (TenantCreated, TenancyInitialized, etc.), tenant testing, or when the user mentions tenancy, multi-tenancy, tenant isolation, stancl/tenancy, or saas in a Laravel project. Always use this skill whenever working with tenant-scoped functionality, even if the user doesn't explicitly mention the package."
license: MIT
metadata:
  author: Yvan de Wert
---

# Stancl Tenancy v3

## Documentation First

The official docs at https://tenancyforlaravel.com/docs/v3/ are the authoritative source. This skill covers the package's architecture and common production patterns.

## When to Apply

- Creating or modifying `app/Models/Tenant.php` or `app/Tenancy/*`
- Working with tenant identification middleware (`InitializeTenancyBy*`)
- Multi-database tenancy: database creation, migration, seeding
- Writing tenant-aware jobs, queues, cache, filesystems, or Redis
- Using tenant console commands (`tenants:migrate`, `tenants:seed`, `tenants:run`)
- Adding or modifying tenancy bootstrappers or features
- Wiring tenancy events (`TenantCreated`, `TenancyInitialized`, etc.)
- Testing tenant-scoped functionality
- Integrating third-party packages with tenancy (Scout, Spatie, Livewire, Horizon)

## Package Architecture

### The Two Applications

The package distinguishes between **central** and **tenant** contexts:

- **Central app**: Landing pages, tenant registration, admin dashboard. Routes are scoped to `config('tenancy.central_domains')`.
- **Tenant app**: The actual SaaS application, one instance per tenant. Routes use identification middleware to detect the tenant.

### Tenancy Modes

- **Automatic mode**: After tenant identification, bootstrappers automatically switch database, cache, filesystem, queue, and Redis to tenant context.
- **Manual mode**: The package provides model traits and the `tenant()` helper; you scope data yourself. Use `$tenant->run(fn () => ...)` to execute code in tenant context.

### Tenant Model

A tenant must implement `Stancl\Tenancy\Contracts\Tenant`. The base model `Stancl\Tenancy\Database\Models\Tenant` provides:

- Forced central connection (always queries central DB)
- `data` JSON column for arbitrary attributes not in custom columns
- UUID generation via `GeneratesIds` trait

To use domains AND databases:

```php
use Stancl\Tenancy\Contracts\TenantWithDatabase;
use Stancl\Tenancy\Database\Concerns\HasDatabase;
use Stancl\Tenancy\Database\Concerns\HasDomains;
use Stancl\Tenancy\Database\Models\Tenant as BaseTenant;

class Tenant extends BaseTenant implements TenantWithDatabase
{
    use HasDatabase, HasDomains;
}
```

Override `getCustomColumns()` to store attributes in dedicated table columns instead of the `data` JSON:

```php
public function getCustomColumns(): array
{
    return ['id', 'name', 'plan_id', 'status', 'max_users'];
}
```

### Tenant Identification Middleware

| Middleware | Identification method |
|---|---|
| `InitializeTenancyByDomain` | Full domain (e.g. `acme.com`) |
| `InitializeTenancyBySubdomain` | Subdomain on central domain (e.g. `acme.yoursaas.com`) |
| `InitializeTenancyByDomainOrSubdomain` | Both — dots in `domain` column = full domain, no dots = subdomain |
| `InitializeTenancyByPath` | URL path prefix `/{tenant}/...` |
| `InitializeTenancyByRequestData` | `X-Tenant` header or `?tenant=` query param |

All middleware have a static `$onFail` property for custom failure handling:

```php
InitializeTenancyByDomain::$onFail = function ($exception, $request, $next) {
    return redirect('https://my-central-domain.com/');
};
```

### Key Helpers

```php
// Get current tenant or a tenant attribute
tenant();           // Stancl\Tenancy\Contracts\Tenant|null
tenant('id');       // string|null

// Initialize tenancy manually
tenancy()->initialize($tenant);
tenancy()->end();

// Run code in tenant context
$tenant->run(function () {
    User::create([...]);
});

// Run for all tenants
Tenant::all()->runForEach(function () {
    User::factory()->create();
});
```

### Event System

All events are in `Stancl\Tenancy\Events` namespace.

**Tenancy lifecycle:**
- `InitializingTenancy` → `TenancyInitialized` → `BootstrappingTenancy` → `TenancyBootstrapped`
- `EndingTenancy` → `TenancyEnded` → `RevertingToCentralContext` → `RevertedToCentralContext`

**Tenant lifecycle:**
- `CreatingTenant` → `TenantCreated`
- `UpdatingTenant` → `TenantUpdated`
- `DeletingTenant` → `TenantDeleted`

**Database lifecycle:**
- `DatabaseCreated`, `DatabaseMigrated`, `DatabaseSeeded`, `DatabaseDeleted`

**Listeners:** `BootstrapTenancy` executes bootstrappers on `TenancyInitialized`. `RevertToCentralContext` reverts on `TenancyEnded`.

### JobPipeline

Converts sequential jobs into event listeners — ensures correct order:

```php
Event::listen(TenantCreated::class, JobPipeline::make([
    CreateDatabase::class,
    MigrateDatabase::class,
    SeedDatabase::class,
])->send(fn (TenantCreated $event) => $event->tenant)
  ->shouldBeQueued(true)
  ->toListener());
```

### Console Commands

All tenant-aware commands accept `--tenants=<id>` (repeatable):

```bash
php artisan tenants:migrate
php artisan tenants:migrate --tenants=8075a580-...
php artisan tenants:seed
php artisan tenants:rollback
php artisan tenants:migrate-fresh --tenants=<id>
php artisan tenants:list
php artisan tenants:run email:send --tenants=<id> --option="queue=1" --argument="body=..."
```

### Bootstrappers (config: `tenancy.bootstrappers`)

| Bootstrapper | What it scopes |
|---|---|
| `DatabaseTenancyBootstrapper` | Switches default DB connection to tenant DB |
| `CacheTenancyBootstrapper` | Tags all cache keys with tenant ID (requires Redis) |
| `FilesystemTenancyBootstrapper` | Suffixes storage paths and disk names |
| `QueueTenancyBootstrapper` | Stores tenant ID on jobs; initializes tenancy before processing |
| `RedisTenancyBootstrapper` | Prefixes Redis keys with tenant ID (requires phpredis) |

### Features (config: `tenancy.features`)

Optional functionality classes:
- `UserImpersonation` — impersonate tenant users from central app
- `TelescopeTags` — tags Telescope entries with tenant ID
- `UniversalRoutes` — routes accessible from both central and tenant contexts
- `TenantConfig` — maps tenant storage keys to Laravel config keys
- `CrossDomainRedirect` — redirects between tenant domains
- `ViteBundler` — tenant-aware Vite asset bundling

## Production Patterns

These patterns come from real-world multi-tenant SaaS applications.

### Tenant Model Customization

Extend the base `Tenant` model beyond the package defaults:

```php
class Tenant extends BaseTenant implements TenantWithDatabase
{
    use HasDatabase, HasDomains;

    protected function casts(): array
    {
        return [
            'status' => TenantStatus::class,
            'max_users' => 'integer',
        ];
    }

    public function getCustomColumns(): array
    {
        return ['id', 'name', 'plan_id', 'max_users', 'status'];
    }

    // Relationships
    public function plan(): BelongsTo { ... }
    public function analytics(): HasMany { ... }

    // Feature toggling via data column
    public function hasFeature(TenantFeature $feature): bool
    {
        return in_array($feature->value, $this->enabled_features ?? []);
    }
}
```

### Typical TenantServiceProvider

```php
class TenancyServiceProvider extends AppServiceProvider
{
    public function boot(): void
    {
        // Tenant lifecycle — create DB, migrate, seed
        Event::listen(TenantCreated::class, JobPipeline::make([
            CreateDatabase::class,
            MigrateDatabase::class,
            SeedDatabase::class,
        ])->send(fn ($event) => $event->tenant)->toListener());

        Event::listen(TenantDeleted::class, JobPipeline::make([
            DeleteDatabase::class,
        ])->send(fn ($event) => $event->tenant)->toListener());

        // Bootstrapper lifecycle
        Event::listen(TenancyInitialized::class, BootstrapTenancy::class);
        Event::listen(TenancyEnded::class, RevertToCentralContext::class);

        // Make Livewire routes tenant-aware
        Livewire::setUpdateRoute(fn ($handle) => Route::post('/livewire/update', $handle)
            ->middleware(['web', InitializeTenancyByDomain::class, 'auth']));

        // Scope Spatie packages per tenant:
        // - Permission cache key: "spatie.permission.cache.tenant.{$tenantId}"
        // - MediaLibrary prefix: "tenant/{$tenantId}"
    }
}
```

### Tenant Route Middleware Stack

A production-ready tenant route middleware stack:

```php
Route::middleware([
    InitializeTenancyByDomain::class,    // Identify tenant by domain
    ScopeSessions::class,                // Prevent session leaking between tenants
    PreventAccessFromCentralDomains::class, // Block central domains
    'auth',
    'verified',
    'ensure_tenant_is_active',          // Custom: check tenant status
])->group(function () {
    // Tenant routes
});
```

### Custom Bootstrapper Pattern

```php
use Stancl\Tenancy\Contracts\TenancyBootstrapper;
use Stancl\Tenancy\Contracts\Tenant;

class ScoutTenancyBootstrapper implements TenancyBootstrapper
{
    public function bootstrap(Tenant $tenant): void
    {
        // Avoid prefix accumulation in long-running processes
        $indexPrefix = config('scout.prefix');
        config(['scout.prefix' => "{$indexPrefix}tenant_{$tenant->getTenantKey()}_"]);
    }

    public function revert(): void
    {
        // Strip tenant prefix, restore original
        $prefix = config('scout.prefix');
        config(['scout.prefix' => preg_replace('/tenant_[a-f0-9-]+_/', '', $prefix)]);
    }
}
```

Register in `config/tenancy.php`:

```php
'bootstrappers' => [
    // ... defaults ...
    Stancl\Tenancy\Bootstrappers\DatabaseTenancyBootstrapper::class,
    App\Tenancy\Bootstrappers\ScoutTenancyBootstrapper::class,
],
```

### Tenant-Specific Config via Custom Feature

Map tenant storage to Laravel config dynamically:

```php
use Stancl\Tenancy\Contracts\Feature;
use Stancl\Tenancy\Contracts\Tenant;

class TenantConfig implements Feature
{
    public function bootstrap(Tenant $tenant): void
    {
        // Mail configuration per tenant
        if ($tenant->mail_mailer) {
            config()->set('mail.default', $tenant->mail_mailer);
            config()->set('mail.from.address', $tenant->mail_from_address);
            config()->set('mail.from.name', $tenant->name);
        }

        // Company information
        config()->set('company.name', $tenant->name);
        config()->set('company.address', $tenant->address);
    }
}
```

### Horizon Job Tagging

Tag queued jobs with the tenant ID for monitoring:

```php
trait TenantTagged
{
    public function middleware(): array
    {
        return [new TenantAwareMiddleware];
    }
}

class TenantAwareMiddleware
{
    public function handle($job, $next): void
    {
        if ($tenant = tenant()) {
            $job->tags = array_merge($job->tags ?? [], ["tenant:{$tenant->getTenantKey()}"]);
        }
        $next($job);
    }
}
```

## Common Patterns

### Creating a Tenant

```php
$tenant = Tenant::create(['id' => 'acme', 'name' => 'Acme Corp']);
$tenant->domains()->create(['domain' => 'acme.localhost']);
```

### Running Code in Tenant Context

```php
$tenant->run(function () {
    User::create(['name' => 'Admin', 'email' => 'admin@acme.com', 'password' => bcrypt('secret')]);
});
```

### Making a Model Tenant-Aware (Single-Database)

```php
use Stancl\Tenancy\Database\Concerns\BelongsToTenant;

class Task extends Model
{
    use BelongsToTenant;
}
// Automatically adds tenant_id scoping to queries
```

### Writing a Tenant-Aware Artisan Command

```php
use Stancl\Tenancy\Concerns\HasATenantsOption;
use Stancl\Tenancy\Concerns\TenantAwareCommand;

class MyCommand extends Command
{
    use TenantAwareCommand, HasATenantsOption;

    protected $signature = 'my:command {--tenants=*}';

    public function handle(): int
    {
        // $this->getTenants() returns the tenants to operate on
        return 0;
    }
}
```

### Custom Bootstrapper

```php
use Stancl\Tenancy\Contracts\TenancyBootstrapper;
use Stancl\Tenancy\Contracts\Tenant;

class MyBootstrapper implements TenancyBootstrapper
{
    public function bootstrap(Tenant $tenant): void
    {
        // Apply tenant-specific config
    }

    public function revert(): void
    {
        // Revert to central config
    }
}
```

Then register in `config/tenancy.php` `bootstrappers` array.

### Custom Feature

```php
use Stancl\Tenancy\Contracts\Feature;

class MyFeature implements Feature
{
    public function bootstrap(Tenant $tenant): void
    {
        // Feature logic — runs whether or not tenancy is initialized
    }
}
```

Then register in `config/tenancy.php` `features` array.

## Testing with Tenancy

### Central App Tests

Write standard Laravel tests. No special setup needed.

### Tenant App Tests

**Important:** With multi-database automatic mode, you cannot use `RefreshDatabase` or `:memory:` SQLite. Tenancy switches the default connection.

Pattern for tenant tests:

```php
class TenantTestCase extends TestCase
{
    protected Tenant $tenant;

    protected function setUp(): void
    {
        parent::setUp();
        $this->tenant = Tenant::factory()->create();
        tenancy()->initialize($this->tenant);
    }
}
```

Or selectively per-test:

```php
class FooTest extends TestCase
{
    protected bool $tenancy = false;

    protected function setUp(): void
    {
        parent::setUp();
        if ($this->tenancy) {
            tenancy()->initialize(Tenant::factory()->create());
        }
    }

    public function test_something(): void
    {
        $this->tenancy = true;
        // ... test tenant-scoped behavior
    }
}
```

**Event faking caution:** Don't use `Event::fake()` globally — it breaks tenancy initialization. Use `Event::fake([MyEvent::class])` to fake only specific events.

## Integration Notes

### With Spatie Permission
- Cache key must be tenant-scoped: `'spatie.permission.cache.tenant.' . $tenantId`

### With Spatie MediaLibrary
- Prefix must be tenant-scoped: `'tenant/' . $tenantId`

### With Laravel Scout
- Use a custom bootstrapper to prefix search indexes per tenant
- Meilisearch index format: `{env}_{tenant_id}_` prefix
- Guard against prefix accumulation in long-running processes (queue workers)

### With Livewire
- Update and file-preview routes must include tenant identification middleware: `InitializeTenancyByDomain`
- Use `Livewire::setUpdateRoute()` in `TenancyServiceProvider::boot()`

### With Horizon
- Tag tenant-specific jobs with `tenant:<id>` for monitoring
- See https://tenancyforlaravel.com/docs/v3/integrations/horizon

### With Queues
- Database queue driver: set `'connection' => 'central'` in queue config
- Redis: ensure queue connection isn't in `tenancy.redis.prefixed_connections`
- For central-only jobs: use a queue connection with `'central' => true`

## Configuration Reference

See `config/tenancy.php` for all options. Key static properties settable in `TenancyServiceProvider::boot()`:

```php
// Custom onFail for domain identification
InitializeTenancyByDomain::$onFail = fn ($e, $r, $n) => abort(404);

// Custom header for request data identification
InitializeTenancyByRequestData::$header = 'X-Team';
InitializeTenancyByRequestData::$queryParameter = null;

// Make job pipelines queued by default
JobPipeline::$shouldBeQueuedByDefault = true;
```

## Documentation Links

- Quickstart: https://tenancyforlaravel.com/docs/v3/quickstart/
- Installation: https://tenancyforlaravel.com/docs/v3/installation/
- Configuration: https://tenancyforlaravel.com/docs/v3/configuration/
- Tenants: https://tenancyforlaravel.com/docs/v3/tenants/
- Tenant Identification: https://tenancyforlaravel.com/docs/v3/tenant-identification/
- Event System: https://tenancyforlaravel.com/docs/v3/event-system/
- Multi-Database: https://tenancyforlaravel.com/docs/v3/multi-database-tenancy/
- Queues: https://tenancyforlaravel.com/docs/v3/queues/
- Testing: https://tenancyforlaravel.com/docs/v3/testing/
- Console Commands: https://tenancyforlaravel.com/docs/v3/console-commands/
- Integrations: https://tenancyforlaravel.com/docs/v3/integrating/
