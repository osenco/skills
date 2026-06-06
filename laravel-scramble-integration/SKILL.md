---
name: laravel-scramble-integration
description: >-
  Integrates dedoc/scramble into a Laravel backend to auto-generate OpenAPI docs
  and a /docs/api UI. Use when adding Scramble, configuring API documentation,
  OpenAPI export, subdomain API routing with API_DOMAIN, Sanctum bearer auth in
  docs, OrionResponseOperationExtension, or documenting Orion/custom API
  controllers. Assumes API routes use domain('API_DOMAIN') and ->as('api.') with
  apiPrefix '' in bootstrap/app.php.
---

# Laravel Scramble Integration

[Scramble](https://scramble.dedoc.co) (`dedoc/scramble`) generates OpenAPI 3.1 and a Stoplight-style docs UI from Laravel routes, controllers, Form Requests, and API Resources — no manual annotations required for most endpoints.

Docs: https://scramble.dedoc.co

## Project routing conventions

Follow these rules on HomeCare-style backends (subdomain API, no `/api` prefix):

1. **ignore default routes** — call `Scramble::ignoreDefaultRoutes()`; do not use Scramble's built-in `/docs/api` registration on the main app domain.
2. **register routes in web.php with `domain('API_DOMAIN')`** — Scramble UI and JSON spec routes live in `routes/web.php` inside a domain group.
3. **All API routes have `domain('API_DOMAIN')` and `->as('api.')`** — wrap every API route group with these chained on `Route::`.
4. **set `apiPrefix` to `''` in `bootstrap/app.php` after `api: ...` in `->withRouting`** — API paths are `/v1/...`, not `/api/v1/...`.

```php
// bootstrap/app.php
->withRouting(
    web: __DIR__.'/../routes/web.php',
    api: __DIR__.'/../routes/api.php',
    apiPrefix: '',
    commands: __DIR__.'/../routes/console.php',
    health: '/up',
)
```

## Before integrating

Read the target project's routing first:

- `bootstrap/app.php` — confirm `apiPrefix: ''`
- `routes/api.php` — confirm `Route::domain(env('API_DOMAIN'))->as('api.')` wrapper
- `.env` — `API_DOMAIN` value (e.g. `api.homecare.dot`)
- Auth — Sanctum bearer tokens on protected routes
- Controllers — return types / API Resources Scramble can infer

Scramble complements [laravel-orion-integration](../laravel-orion-integration/SKILL.md). Orion controllers need **`OrionResponseOperationExtension`** — copy the full class from [reference.md](reference.md#orionresponseoperationextension) into `app/Http/Scramble/OrionResponseOperationExtension.php`.

## Integration workflow

```
Scramble integration:
- [ ] composer require dedoc/scramble
- [ ] Confirm bootstrap/app.php has apiPrefix: ''
- [ ] Confirm API routes use domain(API_DOMAIN)->as('api.')
- [ ] AppServiceProvider::register — Scramble::ignoreDefaultRoutes()
- [ ] AppServiceProvider::boot — bearer auth, server variables, Orion extension (see below)
- [ ] app/Http/Scramble/OrionResponseOperationExtension.php — copy from reference.md
- [ ] routes/web.php — register UI + JSON routes on domain(API_DOMAIN)
- [ ] php artisan vendor:publish --tag=scramble-config (optional)
- [ ] config/scramble.php — api_path v1, api_domain from env
- [ ] Mark public routes @unauthenticated in PHPDoc
- [ ] Verify /docs/api on API_DOMAIN shows Orion responses + Try It bodies
```

## Step 1: Install

```bash
composer require dedoc/scramble
```

Requires PHP 8.1+, Laravel 10+.

## Step 2: Ignore default routes

In `App\Providers\AppServiceProvider::register()`:

```php
use Dedoc\Scramble\Scramble;

public function register(): void
{
    Scramble::ignoreDefaultRoutes();
}
```

## Step 3: Register docs routes on API domain

In `routes/web.php` (not on the marketing site domain):

```php
use Dedoc\Scramble\Scramble;
use Illuminate\Support\Facades\Route;

Route::domain(env('API_DOMAIN', 'api.homecare.dot'))->group(function (): void {
    Scramble::registerUiRoute('docs/api');
    Scramble::registerJsonSpecificationRoute('docs/api.json');
});
```

Result:

- UI: `https://{API_DOMAIN}/docs/api`
- OpenAPI JSON: `https://{API_DOMAIN}/docs/api.json`

## Step 4: Route resolution

Default Scramble matching uses `api_path` (`api` by default). With `apiPrefix: ''` and paths like `/v1/bookings`, publish config and customize resolution.

**Option A — config** (`config/scramble.php` after publish):

```php
'api_path' => 'v1',
'api_domain' => env('API_DOMAIN'),
```

**Option B — custom resolver** (preferred when routes use `->as('api.')` naming):

```php
// AppServiceProvider::boot()
use Dedoc\Scramble\Scramble;
use Illuminate\Routing\Route;

Scramble::configure()
    ->routes(function (Route $route): bool {
        $name = $route->getName() ?? '';

        return str_starts_with($name, 'api.')
            || str_starts_with($name, 'v1.'); // local env without domain wrapper
    });
```

Option B takes precedence over config path matching when both are set.

## Step 5: OpenAPI server URL

Set the documented server to the API subdomain:

```php
Scramble::configure()
    ->withDocumentTransformers(function (\Dedoc\Scramble\Support\Generator\OpenApi $openApi): void {
        $openApi->servers = [];
        $openApi->addServer(
            url: 'https://'.env('API_DOMAIN', 'api.homecare.dot'),
            description: 'API',
        );
    });
```

Or via published config `scramble.servers`.

## Step 6: AppServiceProvider boot (auth + Orion extension)

Register Scramble in `AppServiceProvider::boot()` with these imports:

```php
use App\Http\Scramble\OrionResponseOperationExtension;
use Dedoc\Scramble\Configuration\OperationTransformers;
use Dedoc\Scramble\Scramble;
use Dedoc\Scramble\Support\Generator\OpenApi;
use Dedoc\Scramble\Support\Generator\SecurityScheme;
use Dedoc\Scramble\Support\Generator\ServerVariable;
```

```php
Scramble::configure()
    ->withDocumentTransformers(function (OpenApi $openApi) {
        $openApi->secure(
            SecurityScheme::http('bearer')
        );
    });

Scramble::defineServerVariables([
    'version' => ServerVariable::make(
        default: 'v1',
        description: 'The version of the API being requested.',
    ),
]);

Scramble::configure()->withOperationTransformers(
    function (OperationTransformers $transformers): void {
        $transformers->append(OrionResponseOperationExtension::class);
    }
);
```

## Step 7: OrionResponseOperationExtension

**Required** when the API uses [laravel-orion](../laravel-orion-integration/SKILL.md). Scramble cannot infer Orion trait responses or Orion search payloads on its own.

Create `app/Http/Scramble/OrionResponseOperationExtension.php` — copy the full implementation from [reference.md § OrionResponseOperationExtension](reference.md#orionresponseoperationextension). Do not abbreviate or restructure it.

The extension:

| Responsibility | Detail |
|----------------|--------|
| Response schemas | Resolves `{Model}Resource` / `{Model}CollectionResource` from controller `$model` (and `$relation` on relation controllers); replaces generic 200 responses |
| Index vs show | `GET` with no route params → collection; `GET` with id → resource; relation index → one parent param |
| Search bodies | `POST …/search` gets Orion search payload schema (scopes, filters, sort, includes, aggregates) |
| Store/update bodies | Resolves `{Model}Request` rules via `RulesToParameters` for Try It |

Naming convention (auto-resolved, same as Orion):

- `App\Models\Booking` → `App\Http\Resources\BookingResource`, `BookingCollectionResource`, `BookingRequest`

## Step 8: Public route annotations

Mark public routes (register, login, forgot-password, reset-password) with PHPDoc:

```php
/**
 * @unauthenticated
 */
public function login(Request $request): JsonResponse
```

## Step 9: Docs access control

By default, docs are **local-only**. For other environments, define the `viewApiDocs` gate:

```php
use Illuminate\Support\Facades\Gate;

Gate::define('viewApiDocs', function ($user): bool {
    return $user?->isAdmin() ?? false;
});
```

## Step 10: Improving generated schemas

| Need | Approach |
|------|----------|
| Orion CRUD/search | `OrionResponseOperationExtension` (Step 7) |
| Hide internal/debug route | `#[ExcludeRouteFromDocs]` on method or `#[ExcludeAllRoutesFromDocs]` on controller |
| Describe a single endpoint | `#[Endpoint(...)]` attribute (Scramble 0.12+) |
| Group/tag endpoints | `#[Group('Bookings')]` on controller |
| Custom action response | Add explicit return type: `JsonResponse`, `*Resource` |
| Enum values | Backed enums in request rules / model casts are inferred |

## API routes file pattern

`routes/api.php` should wrap all endpoints:

```php
$apiRoutes = function (): void {
    Route::prefix('v1')->as('v1.')->group(function (): void {
        // auth, Orion::resource(...), custom actions
    });
};

Route::domain(env('API_DOMAIN', 'api.homecare.dot'))
    ->as('api.')
    ->group($apiRoutes);

// Optional: same routes without domain for php artisan serve / PHPUnit
if (app()->environment('local', 'testing')) {
    $apiRoutes();
}
```

Route names become `api.v1.bookings.index` on the API domain.

## Verification

```bash
php artisan route:list --name=api
php artisan route:list --path=docs
```

1. Visit `https://{API_DOMAIN}/docs/api` (or tunnel URL from `.env`)
2. Confirm all `v1/*` endpoints appear
3. Confirm auth routes show no lock icon (`@unauthenticated`)
4. Confirm protected routes show bearer security
5. Download `/docs/api.json` and validate in Stoplight or Swagger Editor

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| Empty docs | `api_path` still `api` while routes are `v1/*` — set `api_path` to `v1` or use custom `->routes()` resolver |
| Docs on wrong domain | Register Scramble routes in `web.php` inside `domain(API_DOMAIN)` |
| Marketing routes in docs | Tighten `->routes()` resolver to `api.*` / `v1.*` name prefix only |
| 403 on docs in staging | Define `viewApiDocs` gate |
| Duplicate `/api` in paths | Confirm `apiPrefix: ''` in `bootstrap/app.php` |
| Orion routes missing schemas | Add `OrionResponseOperationExtension` and register in `AppServiceProvider::boot()` |
| Try It empty for Orion store/search | Extension injects request bodies from `{Model}Request` rules and search schema |

## Additional resources

- `OrionResponseOperationExtension` source + AppServiceProvider: [reference.md](reference.md)
- Scramble docs: https://scramble.dedoc.co
- Orion API patterns: [laravel-orion-integration](../laravel-orion-integration/SKILL.md)
