# Orion Reference

Supplement for [SKILL.md](SKILL.md). Read when registering relationship routes or debugging endpoint shapes.

## Route registration helpers

| Eloquent relation | Orion helper |
|-------------------|--------------|
| `hasOne` | `Orion::hasOneResource($parent, $relation, $controller)` |
| `hasMany` | `Orion::hasManyResource($parent, $relation, $controller)` |
| `belongsTo` | `Orion::belongsToResource($parent, $relation, $controller)` |
| `belongsToMany` | `Orion::belongsToManyResource($parent, $relation, $controller)` |
| `hasOneThrough` | `Orion::hasOneThroughResource($parent, $relation, $controller)` |
| `hasManyThrough` | `Orion::hasManyThroughResource($parent, $relation, $controller)` |
| `morphOne` | `Orion::morphOneResource($parent, $relation, $controller)` |
| `morphMany` | `Orion::morphManyResource($parent, $relation, $controller)` |
| `morphTo` | `Orion::morphToResource($parent, $relation, $controller)` |
| `morphToMany` | `Orion::morphToManyResource($parent, $relation, $controller)` |
| `morphedByMany` | `Orion::morphedByManyResource($parent, $relation, $controller)` |

Chain `->withSoftDeletes()` on any helper for restore endpoints.

## Relation endpoint shapes

### One-to-one (`hasOne`, `hasOneThrough`, `morphOne`, `belongsTo`, `morphTo`)

`store`, `show`, `update`, `destroy`. `belongsTo` and `morphTo` have no `store`.

```
POST   api/{parent}/{parentId}/{relation}
GET    api/{parent}/{parentId}/{relation}/{relatedId?}
PATCH  api/{parent}/{parentId}/{relation}/{relatedId?}
DELETE api/{parent}/{parentId}/{relation}/{relatedId?}
```

### One-to-many (`hasMany`, `hasManyThrough`, `morphMany`)

`index`, `search`, `store`, `show`, `update`, `destroy`, `associate`, `dissociate`, batch endpoints.

Associate payload: `{"related_key": 5}`

### Many-to-many (`belongsToMany`, `morphToMany`, `morphedByMany`)

Adds: `attach`, `detach`, `sync`, `toggle`, `updatePivot`.

Attach/sync payload: `{"resources": [1, 2, 3]}` · Pivot update: `{"pivot": {"role": "admin"}}`

Set `$pivotFillable` on the relation controller for writable pivot columns.

## Query customization methods

| Endpoint | Build | Run | Perform |
|----------|-------|-----|---------|
| index | `buildIndexFetchQuery` | `runIndexFetchQuery` | — |
| store | `buildStoreFetchQuery` | `runStoreFetchQuery` | `performStore` |
| show | `buildShowFetchQuery` | `runShowFetchQuery` | — |
| update | `buildUpdateFetchQuery` | `runUpdateFetchQuery` | `performUpdate` |
| destroy | `buildDestroyFetchQuery` | `runDestroyFetchQuery` | `performDestroy` |
| restore | `buildRestoreFetchQuery` | `runRestoreFetchQuery` | `performRestore` |

Shared across show/update/destroy/restore: `buildFetchQuery`, `runFetchQuery`.

## Batch operation payloads

**batchStore:** `{"resources": [{"title": "A"}, {"title": "B"}]}`

**batchUpdate:** `{"resources": {"5": {"title": "Updated"}}}`

**batchDestroy:** `{"resources": [5, 6]}`

**batchRestore:** `{"resources": [5, 6]}`

## Config highlights (`config/orion.php`)

| Key | Purpose |
|-----|---------|
| `auth.guard` | Guard for `resolveUser()` — default to `sanctum` |
| `use_validated` | Fill models with validated request data only |
| `search.case-sensitive` | Default keyword search case sensitivity |
| `search.max_nested_depth` | Max nested filter depth |
| `transactions.enabled` | Wrap hooks + operations in DB transactions |
| `specs.info` / `specs.servers` | OpenAPI metadata |

## Security reference

Docs: https://orion.tailflow.org/guide/security

1. **Authentication** — Laravel Sanctum (`auth:sanctum` middleware on routes)
2. **Authorization** — Laravel policies, resolved by Orion
3. **Validation** — `Orion\Http\Requests\Request`, resolved by Orion

## Policy generation reference

```bash
php artisan make:policy PostPolicy --model=Post
```

### Endpoint → policy ability map

| HTTP / Orion action | Policy method |
|---------------------|---------------|
| GET index / POST search | `viewAny` |
| GET show | `view` |
| POST store / batchStore | `create` |
| PUT/PATCH update / batchUpdate | `update` |
| DELETE destroy / batchDestroy | `delete` |
| POST restore / batchRestore | `restore` |

### Relation policy signatures

Parent model is the **last** argument:

```php
public function create(User $user, Post $post): bool {}
public function update(User $user, Comment $comment, Post $post): bool {}
```

```php
return Gate::forUser($user)->inspect('update', $post);
```

### Authentication / Sanctum guard

Docs: https://orion.tailflow.org/guide/security#usage-with-sanctum-or-any-other-custom-auth-guard

Orion does not authenticate requests. Sanctum handles that via route middleware. Orion separately resolves the current user for **policy authorization** — it defaults to the `api` guard, which will not see Sanctum token users unless reconfigured.

| Layer | Responsibility | Configuration |
|-------|----------------|---------------|
| Request auth | Laravel Sanctum | `Route::middleware('auth:sanctum')` on Orion route group |
| Policy user resolution | Orion | `auth.guard` in `config/orion.php` **or** `resolveUser()` on controller |

**Option A — global (preferred):**

```php
// config/orion.php
'auth' => [
    'guard' => 'sanctum',
],
```

**Option B — per controller:**

```php
public function resolveUser()
{
    return Auth::guard('sanctum')->user();
}
```

Use one approach, not both unless overriding a single controller. Replace `sanctum` with another guard name for Passport or custom guards.

### Controller policy overrides

```php
protected $policy = CustomPostPolicy::class;
protected $parentPolicy = CustomPostPolicy::class; // relation controllers
```

## Validation (request classes)

Request classes **must** extend `Orion\Http\Requests\Request`.

Auto-resolution: `App\Http\Requests\{Model}Request` for `App\Models\{Model}`.

Override: `protected $request = CustomPostRequest::class;`

### Rules methods

| Method | Merged with `commonRules`? |
|--------|---------------------------|
| `commonRules()` | Base |
| `storeRules()` | Yes — overwrites same keys |
| `updateRules()` | Yes — overwrites same keys |
| `associateRules()`, `attachRules()`, `detachRules()`, `syncRules()`, `toggleRules()`, `updatePivotRules()` | No |
| `batchStoreRules()`, `batchUpdateRules()` | No |

### Messages methods

`commonMessages()`, `storeMessages()`, `updateMessages()`, plus relation/batch variants mirroring the rules methods. Relation/batch messages are not merged with `commonMessages`.

### Retrieving request data

| `use_validated` | Behavior |
|-----------------|----------|
| `false` (default) | All request input → `fill()` |
| `true` | Validated fields only → `fill()` |

## API resource generation reference

Docs: https://orion.tailflow.org/guide/responses

Auto-resolution:

- `App\Http\Resources\{Model}Resource` — single model responses
- `App\Http\Resources\{Model}CollectionResource` — index/search responses

Base classes: `Orion\Http\Resources\Resource`, `Orion\Http\Resources\CollectionResource`

Use `toArrayWithMerge()` to preserve Orion's pagination envelope.

If the project has existing API transformers, delegate from the resource:

```php
return app(PostTransformer::class)->toArray($this->resource);
```

Keep API resources in `App\Http\Resources\` — separate from admin panel resource classes.

## Search reference

Docs: https://orion.tailflow.org/guide/search

### Integration rule

Every Orion model/relation controller **must** implement:

1. `exposedScopes()` — whitelist of model scopes clients may apply
2. `filterableBy()` — whitelist of filterable fields

Optionally: `sortableBy()`, `searchableBy()`, `includes()`, `alwaysIncludes()`, `aggregates()`.

For custom query logic, optionally override `buildIndexFetchQuery()` / `buildFetchQuery()` — see [Models guide](https://orion.tailflow.org/guide/models).

### Constraint order

```
scopes → filters → search → sort → includes → aggregates
```

### Scope payload

```json
{
  "scopes": [
    {"name": "published"},
    {"name": "whereCategory", "parameters": ["news"]}
  ]
}
```

### Filter payload

```json
{
  "filters": [
    {"field": "created_at", "operator": ">=", "value": "2020-01-01"},
    {"field": "options->visible", "operator": "=", "value": true},
    {"type": "or", "nested": [
      {"field": "status", "operator": "=", "value": "draft"},
      {"field": "user.id", "operator": "=", "value": 5, "type": "or"}
    ]}
  ]
}
```

**Operators:** `<`, `<=`, `>`, `>=`, `=`, `!=`, `like`, `not like`, `ilike`, `not ilike`, `in`, `not in`, `all in`, `any in`

**Field notation:** `column`, `relation.field`, `json->key`, `pivot.field`

**Nested depth:** `search.max_nested_depth` (default 1)

### Full search payload example

```json
{
  "scopes": [{"name": "published"}],
  "filters": [{"field": "status", "operator": "=", "value": "published"}],
  "search": {"value": "Example", "case_sensitive": false},
  "sort": [{"field": "created_at", "direction": "desc"}],
  "includes": [
    {"relation": "user"},
    {"relation": "tags", "filters": [{"field": "tags.name", "operator": "like", "value": "%laravel%"}]}
  ],
  "aggregates": [
    {"type": "count", "relation": "comments"},
    {"type": "avg", "relation": "comments", "field": "rating"}
  ]
}
```

## Useful commands

```bash
php artisan route:list --path=api
php artisan orion:specs
php artisan make:policy ModelPolicy --model=Model
php artisan make:request ModelRequest   # change base to Orion\Http\Requests\Request
php artisan make:resource ModelResource
php artisan make:resource ModelCollectionResource
```
