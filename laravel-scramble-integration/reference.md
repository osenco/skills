# Scramble Reference

Supplement for [SKILL.md](SKILL.md). Read when publishing config, registering Orion docs support, or debugging route matching.

## AppServiceProvider boot (Scramble + Orion)

```php
<?php

namespace App\Providers;

use App\Http\Scramble\OrionResponseOperationExtension;
use Dedoc\Scramble\Configuration\OperationTransformers;
use Dedoc\Scramble\Scramble;
use Dedoc\Scramble\Support\Generator\OpenApi;
use Dedoc\Scramble\Support\Generator\SecurityScheme;
use Dedoc\Scramble\Support\Generator\ServerVariable;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        Scramble::ignoreDefaultRoutes();
    }

    public function boot(): void
    {
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
    }
}
```

Optional: add route resolver and server URL in the same `boot()` if not using published config — see SKILL.md Step 4–5.

## OrionResponseOperationExtension

Copy verbatim to `app/Http/Scramble/OrionResponseOperationExtension.php`. Required for Orion APIs so Scramble documents Resource/CollectionResource responses and Try It request bodies.

```php
<?php

namespace App\Http\Scramble;

use Dedoc\Scramble\Extensions\OperationExtension;
use Dedoc\Scramble\Support\Generator\Operation;
use Dedoc\Scramble\Support\Generator\Reference;
use Dedoc\Scramble\Support\Generator\RequestBodyObject;
use Dedoc\Scramble\Support\Generator\Response;
use Dedoc\Scramble\Support\Generator\Schema;
use Dedoc\Scramble\Support\Generator\Types\ArrayType;
use Dedoc\Scramble\Support\Generator\Types\ObjectType as GeneratorObjectType;
use Dedoc\Scramble\Support\Generator\Types\StringType;
use Dedoc\Scramble\Support\OperationExtensions\RulesExtractor\RulesToParameters;
use Dedoc\Scramble\Support\RouteInfo;
use Dedoc\Scramble\Support\Type\ObjectType as InferObjectType;
use Illuminate\Http\Request;
use Orion\Http\Controllers\Controller as OrionController;
use Orion\Http\Controllers\RelationController as OrionRelationController;
use ReflectionClass;

/**
 * Injects response schemas and request body schemas for Orion API routes so Scramble
 * documents the correct Resource/CollectionResource responses and Try It request bodies.
 */
class OrionResponseOperationExtension extends OperationExtension
{
    private const BODY_METHODS = ['post', 'put', 'patch'];

    public function handle(Operation $operation, RouteInfo $routeInfo): void
    {
        $controllerClass = $routeInfo->className();
        if (! $controllerClass || ! $this->isOrionController($controllerClass)) {
            return;
        }

        $modelClass = $this->getControllerModel($controllerClass);
        if (! $modelClass) {
            return;
        }

        $resourceModelClass = $this->resolveResourceModelClass($controllerClass, $modelClass);
        $resourceClass = $this->modelToResourceClass($resourceModelClass);
        $collectionClass = $this->modelToCollectionResourceClass($resourceModelClass);

        if ($resourceClass && class_exists($resourceClass)) {
            $method = $routeInfo->method;
            $isIndex = 'get' === $method && $this->isIndexRoute($routeInfo);
            $isListOperation = $isIndex || ('post' === $method && $this->isBatchRoute($routeInfo));

            if ($isListOperation && class_exists($collectionClass)) {
                $response = $this->openApiTransformer->toResponse(new InferObjectType($collectionClass));
            } else {
                $response = $this->openApiTransformer->toResponse(new InferObjectType($resourceClass));
            }

            if ($response) {
                $this->remove200Responses($operation);
                $operation->addResponse($response);
            }
        }

        $this->injectRequestBodyIfNeeded($operation, $routeInfo, $controllerClass, $resourceModelClass ?? $modelClass);
    }

    private function injectRequestBodyIfNeeded(Operation $operation, RouteInfo $routeInfo, string $controllerClass, string $resourceModelClass): void
    {
        if (! in_array(strtolower($routeInfo->method), self::BODY_METHODS, true)) {
            return;
        }

        if ($this->isSearchRoute($routeInfo)) {
            $operation->addRequestBodyObject(
                RequestBodyObject::make()
                    ->setContent('application/json', Schema::fromType($this->makeSearchRequestBodySchema()))
                    ->required(false)
            );

            return;
        }

        $requestClass = $this->getControllerRequestClass($controllerClass)
            ?? $this->modelToRequestClass($resourceModelClass);

        if (! $requestClass || ! class_exists($requestClass) || ! is_subclass_of($requestClass, \Orion\Http\Requests\Request::class, true)) {
            return;
        }

        $rules = $this->getOrionRequestRules($requestClass, $routeInfo);
        if ([] === $rules) {
            return;
        }

        $parameters = (new RulesToParameters(
            $rules,
            [],
            $this->openApiTransformer,
            'body'
        ))->mergeDotNotatedKeys(false)->handle();

        if ([] === $parameters) {
            return;
        }

        $schema = Schema::createFromParameters($parameters);
        $operation->addRequestBodyObject(
            RequestBodyObject::make()
                ->setContent('application/json', $schema)
                ->required($this->hasRequiredRules($rules))
        );
    }

    private function getOrionRequestRules(string $requestClass, RouteInfo $routeInfo): array
    {
        $actionMethod = $routeInfo->route->getActionMethod();
        try {
            $request = $requestClass::createFrom(Request::create(
                $routeInfo->route->uri(),
                $routeInfo->method
            ));
            $request->setRouteResolver(fn () => $routeInfo->route);

            return $request->rules();
        } catch (\Throwable) {
            return [];
        }
    }

    private function hasRequiredRules(array $rules): bool
    {
        foreach ($rules as $fieldRules) {
            $a = is_array($fieldRules) ? $fieldRules : [$fieldRules];
            if (in_array('required', $a, true)) {
                return true;
            }
        }

        return false;
    }

    private function isSearchRoute(RouteInfo $routeInfo): bool
    {
        return 'post' === $routeInfo->method && str_contains($routeInfo->route->uri(), '/search');
    }

    private function makeSearchRequestBodySchema(): GeneratorObjectType
    {
        $scopeItem = new GeneratorObjectType;
        $scopeItem->addProperty('name', (new StringType)->setDescription('Scope name'));
        $scopeItem->addProperty('parameters', new ArrayType);

        $filterItem = new GeneratorObjectType;
        $filterItem->addProperty('type', (new StringType)->setDescription('and|or'));
        $filterItem->addProperty('field', (new StringType)->setDescription('Whitelisted field name'));
        $filterItem->addProperty('operator', (new StringType)->setDescription('e.g. =, >=, in, like'));
        $filterItem->addProperty('value', new StringType);

        $searchObj = new GeneratorObjectType;
        $searchObj->addProperty('value', (new StringType)->setDescription('Search phrase'));
        $searchObj->addProperty('case_sensitive', (new StringType)->setDescription('true|false'));

        $sortItem = new GeneratorObjectType;
        $sortItem->addProperty('field', (new StringType)->setDescription('Sortable field'));
        $sortItem->addProperty('direction', (new StringType)->setDescription('asc|desc'));

        $includeItem = new GeneratorObjectType;
        $includeItem->addProperty('relation', new StringType);

        $aggregateItem = new GeneratorObjectType;
        $aggregateItem->addProperty('relation', new StringType);
        $aggregateItem->addProperty('type', (new StringType)->setDescription('count|avg|sum|min|max|exists'));
        $aggregateItem->addProperty('field', new StringType);

        $schema = new GeneratorObjectType;
        $schema->addProperty('scopes', (new ArrayType)->setItems($scopeItem));
        $schema->addProperty('filters', (new ArrayType)->setItems($filterItem));
        $schema->addProperty('search', $searchObj);
        $schema->addProperty('sort', (new ArrayType)->setItems($sortItem));
        $schema->addProperty('includes', (new ArrayType)->setItems($includeItem));
        $schema->addProperty('aggregates', (new ArrayType)->setItems($aggregateItem));

        return $schema;
    }

    private function resolveResourceModelClass(string $controllerClass, string $modelClass): string
    {
        if (! is_subclass_of($controllerClass, OrionRelationController::class, true)) {
            return $modelClass;
        }
        try {
            $ref = new ReflectionClass($controllerClass);

            if (! $ref->hasProperty('relation')) {
                return $modelClass;
            }

            $prop = $ref->getProperty('relation');
            $relationName = $prop->getValue($ref->newInstanceWithoutConstructor());

            if (! is_string($relationName)) {
                return $modelClass;
            }

            $parent = app($modelClass);
            $relation = $parent->{$relationName}();

            return get_class($relation->getRelated());
        } catch (\Throwable) {
            return $modelClass;
        }
    }

    private function getControllerRequestClass(string $controllerClass): ?string
    {
        try {
            $ref = new ReflectionClass($controllerClass);
            if (! $ref->hasProperty('request')) {
                return null;
            }
            $prop = $ref->getProperty('request');
            $value = $prop->getValue($ref->newInstanceWithoutConstructor());

            return is_string($value) ? $value : null;
        } catch (\Throwable) {
            return null;
        }
    }

    private function modelToRequestClass(string $modelClass): string
    {
        $ref = new ReflectionClass($modelClass);
        $requestNamespace = str_replace('\\Models', '\\Http\\Requests', $ref->getNamespaceName());

        return $requestNamespace.'\\'.($ref->getShortName()).'Request';
    }

    private function remove200Responses(Operation $operation): void
    {
        if (! $operation->responses) {
            return;
        }
        $operation->responses = array_values(array_filter(
            $operation->responses,
            fn ($r) => 200 !== $this->getResponseCode($r)
        ));
    }

    private function getResponseCode(mixed $response): ?int
    {
        if ($response instanceof Response) {
            return $response->code;
        }
        if ($response instanceof Reference) {
            try {
                return $response->resolve()->code ?? null;
            } catch (\Throwable) {
                return null;
            }
        }

        return null;
    }

    private function isOrionController(?string $class): bool
    {
        if (! $class) {
            return false;
        }

        return is_subclass_of($class, OrionController::class, true)
            || is_subclass_of($class, OrionRelationController::class, true);
    }

    private function getControllerModel(string $controllerClass): ?string
    {
        try {
            $ref = new ReflectionClass($controllerClass);
            if (! $ref->hasProperty('model')) {
                return null;
            }
            $prop = $ref->getProperty('model');

            $model = $prop->getValue($ref->newInstanceWithoutConstructor());

            return is_string($model) ? $model : null;
        } catch (\Throwable) {
            return null;
        }
    }

    private function modelToResourceClass(string $modelClass): string
    {
        $ref = new ReflectionClass($modelClass);
        $resourceNamespace = str_replace('\\Models', '\\Http\\Resources', $ref->getNamespaceName());

        return $resourceNamespace.'\\'.($ref->getShortName()).'Resource';
    }

    private function modelToCollectionResourceClass(string $modelClass): string
    {
        $ref = new ReflectionClass($modelClass);
        $resourceNamespace = str_replace('\\Models', '\\Http\\Resources', $ref->getNamespaceName());

        return $resourceNamespace.'\\'.($ref->getShortName()).'CollectionResource';
    }

    private function isIndexRoute(RouteInfo $routeInfo): bool
    {
        if ('get' !== $routeInfo->method) {
            return false;
        }
        $paramNames = $routeInfo->route->parameterNames();
        $controllerClass = $routeInfo->className();
        if (is_subclass_of($controllerClass, OrionRelationController::class, true)) {
            return 1 === count($paramNames);
        }

        return 0 === count($paramNames);
    }

    private function isBatchRoute(RouteInfo $routeInfo): bool
    {
        $path = $routeInfo->route->uri();

        return str_contains($path, 'batch');
    }
}
```

## routes/web.php docs block

```php
use Dedoc\Scramble\Scramble;
use Illuminate\Support\Facades\Route;

Route::domain(env('API_DOMAIN', 'api.homecare.dot'))->group(function (): void {
    Scramble::registerUiRoute('docs/api');
    Scramble::registerJsonSpecificationRoute('docs/api.json');
});
```

## Published config highlights

```bash
php artisan vendor:publish --provider="Dedoc\Scramble\ScrambleServiceProvider" --tag="scramble-config"
```

```php
// config/scramble.php
return [
    'api_path' => 'v1',
    'api_domain' => env('API_DOMAIN'),

    'info' => [
        'version' => env('API_VERSION', '1.0.0'),
        'description' => 'HomeCare mobile API',
    ],

    'servers' => null,

    'middleware' => [
        'web',
        \Dedoc\Scramble\Http\Middleware\RestrictedDocsAccess::class,
    ],

    'extensions' => [],
];
```

## Route resolver (subdomain + apiPrefix '')

```php
use Dedoc\Scramble\Scramble;
use Illuminate\Routing\Route;

Scramble::configure()
    ->routes(function (Route $route): bool {
        $name = $route->getName() ?? '';

        return str_starts_with($name, 'api.')
            || str_starts_with($name, 'v1.');
    });
```

## Excluding routes

```php
use Dedoc\Scramble\Attributes\ExcludeRouteFromDocs;
use Dedoc\Scramble\Attributes\ExcludeAllRoutesFromDocs;

#[ExcludeAllRoutesFromDocs]
class InternalController extends Controller {}

class BookingsController extends OrionController
{
    #[ExcludeRouteFromDocs]
    public function accept(Request $request, Booking $booking): JsonResponse
    {
        // hidden from docs but still routable
    }
}
```

## Public auth endpoints

```php
/**
 * @unauthenticated
 */
public function register(Request $request): JsonResponse
```

## Environment variables

| Variable | Purpose |
|----------|---------|
| `API_DOMAIN` | Subdomain for API routes and Scramble docs |
| `API_VERSION` | OpenAPI `info.version` (optional) |

## Useful commands

```bash
composer require dedoc/scramble
php artisan vendor:publish --tag=scramble-config
php artisan route:list --name=api
curl -s "https://${API_DOMAIN}/docs/api.json" | jq '.paths | keys | length'
```
