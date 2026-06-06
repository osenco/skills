---
name: laravel-orion-integration
description: >-
  Integrates tailflow/laravel-orion into a Laravel backend to expose Eloquent
  models and relationships as REST APIs. Use when adding Orion, creating Orion
  controllers, registering Orion routes, policies, request classes, API
  resources, search scoping/filtering (exposedScopes, filterableBy),
  relationship APIs, hooks, or OpenAPI specs for Laravel models. Assumes
  Laravel Sanctum for API authentication unless the project specifies otherwise.
---

# Laravel Orion Integration

[Orion for Laravel](https://orion.tailflow.org) (`tailflow/laravel-orion`) generates REST endpoints for Eloquent models and relationships using Laravel policies, request classes, and API resources.

Docs: https://orion.tailflow.org/guide/

## Before integrating

Read the target project's existing API layout first:

- Sanctum auth (`auth:sanctum` middleware, `sanctum` guard) — primary default
- Route file (`routes/api.php`) and URI prefix
- Controller namespace (commonly `App\Http\Controllers\Api`)
- Whether model policies already exist

Orion does **not** replace custom action endpoints (e.g. `orders/{order}/cancel`). Register Orion resources alongside existing routes for standard CRUD/search on models.

## Installation

```bash
composer require tailflow/laravel-orion
php artisan vendor:publish --tag=orion-config
```

Composer v1 users: upgrade to v2, or run `composer require doctrine/dbal` before installing Orion.

## Integration workflow

```
Orion integration:
- [ ] Install package and publish config
- [ ] Security: auth:sanctum middleware + config auth.guard sanctum (or resolveUser)
- [ ] Security: policy with viewAny/view/create/update/delete (+ restore)
- [ ] Security: Orion request class (commonRules/storeRules/updateRules)
- [ ] Generate API resource + collection resource
- [ ] Create Orion controller with exposedScopes + filterableBy
- [ ] Register Orion route inside auth:sanctum group
- [ ] Verify with php artisan route:list
- [ ] Smoke-test index/store/search endpoints
```

## Step 1: Model controller

Create `App\Http\Controllers\Api\{Model}Controller` extending `Orion\Http\Controllers\Controller`.

**Every Orion controller must implement scoping and filtering** ([Search guide](https://orion.tailflow.org/guide/search)):

1. **`exposedScopes()`** — whitelist model scopes the client may apply via the search payload (preferred over filters for server-owned logic)
2. **`filterableBy()`** — whitelist fields the client may filter on (always required, never leave empty on user-facing resources)

```php
<?php

namespace App\Http\Controllers\Api;

use App\Models\Post;
use Orion\Http\Controllers\Controller;

class PostsController extends Controller
{
    protected $model = Post::class;

    public function exposedScopes(): array
    {
        return ['published', 'whereCategory'];
    }

    public function filterableBy(): array
    {
        return ['id', 'title', 'status', 'user_id', 'created_at'];
    }

    public function sortableBy(): array
    {
        return ['id', 'title', 'created_at'];
    }

    public function searchableBy(): array
    {
        return ['title', 'body'];
    }

    public function includes(): array
    {
        return ['user', 'tags'];
    }
}
```

### Scoping vs filtering

| Layer | Where | Who controls | Use for |
|-------|-------|--------------|---------|
| Scopes | `exposedScopes()` + `scopes` in search payload | Client requests, server whitelists | Reusable query constraints on the model |
| Filters | `filterableBy()` + `filters` in search payload | Client requests, server whitelists | Ad-hoc field comparisons |

Prefer **scopes** over **filters** when the constraint is business logic that may change — scopes stay on the model, clients only pass a name ([Search — Filtering](https://orion.tailflow.org/guide/search)).

Key controller options:

| Need | Approach |
|------|----------|
| Disable pagination | `use Orion\Concerns\DisablePagination` |
| Custom route key (slug/uuid) | Override `keyName(): string` |
| Custom query building | Override `buildIndexFetchQuery()` / `buildFetchQuery()` |
| Custom store/update logic | Override `performStore()` / `performUpdate()` or use hooks |
| Custom policy | Set `protected $policy = CustomPolicy::class` |
| Custom request class | Set `protected $request = CustomRequest::class` |
| Custom API resource | Set `protected $resource` / `protected $collectionResource` |
| Auth guard | See [Security](#step-2-security-policies--requests) |

## Step 2: Security (policies & requests)

Orion does not handle authentication itself — you wire that up. Authorization and validation are Orion's two security layers, both documented in the [Security guide](https://orion.tailflow.org/guide/security).

### Authentication (Sanctum)

Orion provides **no** authentication — set that up yourself. Use **Laravel Sanctum** for token-based API auth ([Security — Authentication](https://orion.tailflow.org/guide/security)).

**Route middleware** — protect Orion routes with Sanctum in `routes/api.php`:

```php
Route::middleware('auth:sanctum')->group(function () {
    Orion::resource('posts', PostsController::class);
});
```

**Orion user resolution for policies** — by default Orion uses the `api` guard to resolve the authenticated user for authorization. With Sanctum, that guard mismatch causes 403s. Fix it using **one** of the two approaches from the [Sanctum guard docs](https://orion.tailflow.org/guide/security#usage-with-sanctum-or-any-other-custom-auth-guard):

**Option A — global (preferred):** set `auth.guard` in `config/orion.php` after `php artisan vendor:publish --tag=orion-config`:

```php
'auth' => [
    'guard' => 'sanctum',
],
```

**Option B — per controller:** override `resolveUser()`:

```php
<?php

namespace App\Http\Controllers\Api;

use App\Models\Post;
use Illuminate\Support\Facades\Auth;
use Orion\Http\Controllers\Controller;

class PostsController extends Controller
{
    protected $model = Post::class;

    /**
     * Retrieves currently authenticated user based on the guard.
     *
     * @return \Illuminate\Contracts\Auth\Authenticatable|null
     */
    public function resolveUser()
    {
        return Auth::guard('sanctum')->user();
    }
}
```

Use Option A unless a specific controller needs a different guard. For other auth systems (e.g. Passport), apply the same pattern with the appropriate guard name.

### Authorization (policies)

Both model and relation controllers authorize via **Laravel policies**. Orion calls the matching policy method before each operation — no manual `$this->authorize()` in controllers. Missing policy → **403**.

Generate:

```bash
php artisan make:policy PostPolicy --model=Post
```

Laravel auto-discovers `App\Policies\{Model}Policy` for `App\Models\{Model}`.

| Orion endpoint | Policy method |
|----------------|---------------|
| `index`, `search` | `viewAny` |
| `show` | `view` |
| `store`, `batchStore` | `create` |
| `update`, `batchUpdate` | `update` |
| `destroy`, `batchDestroy` | `delete` |
| `restore`, `batchRestore` | `restore` |

**Custom policy class** — set on the controller when rules differ per context:

```php
protected $policy = PostPolicy::class;
```

**Relation controllers** — two policies, parent passed as extra argument ([Authorizing Parent Entities](https://orion.tailflow.org/guide/security)):

```php
protected $parentPolicy = PostPolicy::class;
protected $policy = PostMetaPolicy::class;
```

```php
class PostPolicy
{
    public function update(User $user, Post $post): bool
    {
        return $post->user_id === $user->id;
    }
}

class PostMetaPolicy
{
    // $post is the parent — authorize against parent, not the relation row
    public function update(User $user, PostMeta $postMeta, Post $post): bool
    {
        return Gate::forUser($user)->inspect('update', $post);
    }

    public function create(User $user, Post $post): bool
    {
        return Gate::forUser($user)->inspect('update', $post);
    }
}
```

**Disable authorization** (local prototyping only):

```php
use Orion\Concerns\DisableAuthorization;

class PostsController extends Controller
{
    use DisableAuthorization;
    protected $model = Post::class;
}
```

**Policy patterns:**

- Reuse model helpers and ownership checks in `view`/`update`/`delete`
- Use policies for row-level authorization on show/update/delete
- Custom action routes keep their own authorization — Orion policies cover standard CRUD/batch/restore only

### Validation (request classes)

Request classes **must** extend `Orion\Http\Requests\Request` — **not** `Illuminate\Foundation\Http\FormRequest`.

Orion auto-resolves `App\Http\Requests\{Model}Request` for `App\Models\{Model}`. Override with `protected $request` on the controller:

```php
protected $request = PostRequest::class;
```

**Rules** — define per endpoint using Orion's methods (not `rules()`):

| Method | Used for | Merged with `commonRules`? |
|--------|----------|---------------------------|
| `commonRules()` | Shared store + update rules | — |
| `storeRules()` | POST store | Yes — overwrites same keys |
| `updateRules()` | PUT/PATCH update | Yes — overwrites same keys |
| `associateRules()` | Relation associate | No |
| `attachRules()` | Relation attach | No |
| `detachRules()` | Relation detach | No |
| `syncRules()` | Relation sync | No |
| `toggleRules()` | Relation toggle | No |
| `updatePivotRules()` | Relation pivot update | No |
| `batchStoreRules()` | Batch store | No |
| `batchUpdateRules()` | Batch update | No |

```php
<?php

namespace App\Http\Requests;

use Orion\Http\Requests\Request;

class PostRequest extends Request
{
    public function commonRules(): array
    {
        return ['title' => 'required|string|max:255'];
    }

    public function storeRules(): array
    {
        return ['status' => 'required|in:draft,published'];
    }

    public function updateRules(): array
    {
        return ['status' => 'sometimes|in:draft,published'];
    }
}
```

On `store`, both `commonRules` and `storeRules` apply. On `update`, only `commonRules` + `updateRules` apply.

**Messages** — mirror the rules structure with `commonMessages()`, `storeMessages()`, `updateMessages()`, and relation/batch variants. Per-endpoint messages overwrite `commonMessages` for the same key; relation/batch messages are **not** merged with `commonMessages`.

**Filling models** — by default Orion passes **all** request data to `fill()`. To use only validated fields:

```php
'use_validated' => true, // config/orion.php
```

Full rule/message method list: [reference.md](reference.md#validation-request-classes).

## Step 3: Generate API resources

Orion transforms Eloquent models into JSON via API Resources ([Responses guide](https://orion.tailflow.org/guide/responses)).

### Naming and auto-resolution

| Model | Single resource | Collection resource |
|-------|-----------------|---------------------|
| `App\Models\Post` | `App\Http\Resources\PostResource` | `App\Http\Resources\PostCollectionResource` |

```bash
php artisan make:resource PostResource
php artisan make:resource PostCollectionResource
```

If names collide with other packages (admin panels, etc.), set explicit classes on the controller:

```php
protected $resource = ApiPostResource::class;
protected $collectionResource = ApiPostCollectionResource::class;
```

### Extend Orion base classes

Change generated stubs to extend Orion's resource classes (not `Illuminate\Http\Resources\Json\JsonResource`):

```php
<?php

namespace App\Http\Resources;

use Illuminate\Http\Request;
use Orion\Http\Resources\Resource;

class PostResource extends Resource
{
    public function toArray(Request $request): array
    {
        return $this->toArrayWithMerge([
            'id'        => $this->id,
            'title'     => $this->title,
            'status'    => $this->status,
            'createdAt' => $this->created_at?->toIso8601String(),
            'author'    => $this->whenLoaded('user', fn () => [
                'id'   => $this->user->id,
                'name' => $this->user->name,
            ]),
        ]);
    }
}
```

```php
<?php

namespace App\Http\Resources;

use Illuminate\Http\Request;
use Orion\Http\Resources\CollectionResource;

class PostCollectionResource extends CollectionResource
{
    public function toArray(Request $request): array
    {
        return $this->toArrayWithMerge([
            'data' => PostResource::collection($this->collection),
        ]);
    }
}
```

`toArrayWithMerge()` merges your array with Orion's response envelope (pagination meta, etc.).

### Resource checklist

```
API resource setup:
- [ ] Create {Model}Resource extending Orion\Http\Resources\Resource
- [ ] Create {Model}CollectionResource extending Orion\Http\Resources\CollectionResource
- [ ] Eager-load relations used in toArray() or use whenLoaded()
- [ ] Set $resource / $collectionResource on controller if names are non-standard
- [ ] Whitelist same relations in includes() that the resource exposes
```

### Delegating to existing transformers

If the project already shapes API JSON in services or DTOs, delegate from the Orion resource to keep client payloads stable:

```php
return app(PostTransformer::class)->toArray($this->resource);
```

Place API resources in `App\Http\Resources\` — separate from admin panel resource classes.

## Step 4: Register routes

In `routes/api.php`, inside the Sanctum middleware group:

```php
use Orion\Facades\Orion;
use App\Http\Controllers\Api\PostsController;

Route::middleware('auth:sanctum')->group(function () {
    Orion::resource('posts', PostsController::class)->withSoftDeletes();
});
```

Soft-deleted models: chain `->withSoftDeletes()` to add `restore` and `batchRestore` routes.

Disable batch endpoints:

```php
Orion::resource('posts', PostsController::class)->withoutBatch();
```

Verify:

```bash
php artisan route:list --path=api/posts
```

### Standard model endpoints

| Method | URI | Action |
|--------|-----|--------|
| GET | `api/{resource}` | index |
| POST | `api/{resource}/search` | search |
| POST | `api/{resource}` | store |
| GET | `api/{resource}/{id}` | show |
| PUT/PATCH | `api/{resource}/{id}` | update |
| DELETE | `api/{resource}/{id}` | destroy |
| POST | `api/{resource}/batch` | batchStore |
| PATCH | `api/{resource}/batch` | batchUpdate |
| DELETE | `api/{resource}/batch` | batchDestroy |
| POST | `api/{resource}/{id}/restore` | restore (with soft deletes) |

Adjust `api/` prefix to match the project's route configuration.

## Step 5: Relationship controllers

```php
<?php

namespace App\Http\Controllers\Api;

use App\Models\Post;
use Orion\Http\Controllers\RelationController;

class PostCommentsController extends RelationController
{
    protected $model = Post::class;
    protected $relation = 'comments';

    public function exposedScopes(): array { return []; }
    public function filterableBy(): array { return ['id', 'created_at']; }
}
```

Register with the matching helper for the Eloquent relation type:

```php
Orion::hasManyResource('posts', 'comments', PostCommentsController::class);
Orion::belongsToManyResource('users', 'roles', UserRolesController::class);
Orion::morphToManyResource('posts', 'tags', PostTagsController::class);
```

For `belongsToMany` / `morphToMany`, set `$pivotFillable` and `$pivotJson` on the controller.

Full route helper list: [reference.md](reference.md).

## Search, filter, sort, include

Docs: [Search guide](https://orion.tailflow.org/guide/search)

Both `GET .../index` and `POST .../search` run through the same query pipeline. **Always** whitelist `exposedScopes()` and `filterableBy()` on the controller.

### Constraint application order

`scopes` → `filters` → `search` → `sort` → `includes` → `aggregates`

### Required controller methods

| Method | Required? | Purpose |
|--------|-----------|---------|
| `exposedScopes()` | **Yes** | Whitelist model scopes |
| `filterableBy()` | **Yes** | Whitelist filterable fields |
| `sortableBy()` | Recommended | Whitelist sort fields |
| `searchableBy()` | If keyword search needed | Whitelist search fields |
| `includes()` | If eager loading needed | Whitelist includable relations |
| `aggregates()` | If counts/sums needed | Whitelist aggregate targets |

### Scopes

```json
// POST /api/posts/search
{
  "scopes": [
    {"name": "published"},
    {"name": "whereCategory", "parameters": ["news"]}
  ]
}
```

Expose model `scopeXxx` or `#[Scope]` methods via `exposedScopes()`.

### Filters

```json
{
  "filters": [
    {"field": "status", "operator": "=", "value": "published"},
    {"field": "created_at", "operator": ">=", "value": "2026-01-01"},
    {"type": "or", "field": "user.id", "operator": "in", "value": [1, 2, 3]}
  ]
}
```

Operators: `<`, `<=`, `>`, `>=`, `=`, `!=`, `like`, `not like`, `ilike`, `not ilike`, `in`, `not in`, `all in`, `any in`.

Field notation: `column`, `relation.field`, `json->key`, `pivot.field`.

Full search reference: [reference.md](reference.md#search-reference).

## Hooks

Prefer hooks over overriding full controller methods for side effects:

```php
protected function beforeSave(Request $request, $post): void
{
    if ($request->isMethod('POST')) {
        $post->user()->associate($request->user());
    }
}
```

Enable transactions globally via `transactions.enabled` in `config/orion.php`.

## OpenAPI specs

```bash
php artisan orion:specs
php artisan orion:specs --path="specs/api.yaml" --format="yaml"
```

## Common pitfalls

1. **403 on every request** — missing policy or wrong guard; set `auth.guard` to `sanctum` or override `resolveUser()`
2. **Validation not running** — request extends `FormRequest` instead of `Orion\Http\Requests\Request`
3. **Extra fields saved to model** — set `use_validated` to `true` in config
4. **Search returns too much data** — empty `filterableBy()` / `exposedScopes()`, or missing policy checks
5. **Filter/scope silently ignored** — not whitelisted on controller
6. **Route conflicts** — Orion CRUD routes colliding with custom action routes
7. **Relation route helper mismatch** — helper must match Eloquent relation type

## When to use Orion vs custom controllers

| Use Orion | Keep custom controllers |
|-----------|-------------------------|
| Standard CRUD on a model | Multi-step workflows |
| Search/filter/sort with whitelists | Cross-model aggregations |
| Nested relation CRUD/attach/sync | Auth endpoints |
| Batch create/update/delete | File uploads or webhooks |

## Additional resources

- [reference.md](reference.md) — routes, security, search, resources
- [Search](https://orion.tailflow.org/guide/search)
- [Security](https://orion.tailflow.org/guide/security)
- [Responses](https://orion.tailflow.org/guide/responses)
- https://orion.tailflow.org/guide/
