---
title: Calling Actions from PHP
nav_order: 7
permalink: /calling-from-php/
---

# Calling Actions from PHP
{: .no_toc }

1. TOC
{:toc}

---

This chapter shows how to invoke an Action API from a modern PHP application. We'll use Mezzio (the successor to Zend Expressive / ZF3) and the IBM i PHP Toolkit, building the handler that backs the *Delete Supplier* button in a React frontend.

If you're still on ZF3 (`Zend\Mvc`), the principles are identical — the syntax just looks slightly different. The K3S [official API documentation](https://technical.k3s.com/docs/api/) shows the ZF3 form. Adjust as needed.

## What we're building

A web application sends a POST to `/api/suppliers/delete` with a JSON body:

```json
{
  "buyr": "00001",
  "locn": "00001",
  "supl": "ACME01",
  "suplsub": ""
}
```

The server hands the request to a Mezzio handler. The handler authenticates the user, calls `AC_DELSUPL` on the IBM i, and returns:

```json
{
  "success": true,
  "errors": "N",
  "errmsg": "",
  "errfield": ""
}
```

Or, on validation failure:

```json
{
  "success": false,
  "errors": "Y",
  "errmsg": "Cannot delete: supplier has open purchase orders",
  "errfield": "IDSUPL"
}
```

The frontend turns the response into a toast notification or an inline error highlight on the offending field.

## The handler

```php
<?php
declare(strict_types=1);

namespace K3sApp\Suppliers\Handler;

use K3sBase\IbmI\ToolkitGateway;
use K3sBase\Auth\AuthenticatedUser;
use Laminas\Diactoros\Response\JsonResponse;
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Server\RequestHandlerInterface;

final class DeleteSupplierHandler implements RequestHandlerInterface
{
    public function __construct(
        private ToolkitGateway $gateway,
        private AuthenticatedUser $user,
        private array $config
    ) {}

    public function handle(ServerRequestInterface $request): ResponseInterface
    {
        $body = $request->getParsedBody() ?? [];

        // 1. Extract and lightly validate inputs at the boundary.
        $buyr     = trim((string) ($body['buyr']    ?? ''));
        $locn     = trim((string) ($body['locn']    ?? ''));
        $supl     = trim((string) ($body['supl']    ?? ''));
        $suplsub  = trim((string) ($body['suplsub'] ?? ''));

        if ($buyr === '' || $locn === '' || $supl === '') {
            return new JsonResponse([
                'success'  => false,
                'errors'   => 'Y',
                'errmsg'   => 'Missing required identifier',
                'errfield' => 'IDSUPL',
            ], 400);
        }

        // 2. Pull the K3S envelope values from injected config.
        $k3sobj  = $this->config['k3s_settings']['k3sobj'];
        $comp    = $this->config['k3s_settings']['comp'];
        $compcod = $this->config['k3s_settings']['compcod'];

        // 3. The "USER" is the IBM i profile mapped to the
        //    authenticated web user. The Auth service handles that lookup.
        $iSeriesUser = $this->user->getIbmIProfile();

        // 4. Build the toolkit parameter list — order matches the
        //    parameter list in AC_DELSUPL exactly.
        $errors   = '';
        $errmsg   = '';
        $errfield = '';

        $params = [
            $this->gateway->charParam('BOTH', 10,  'K3SOBJ',    $k3sobj),
            $this->gateway->charParam('BOTH', 1,   'COMP',      $comp),
            $this->gateway->charParam('BOTH', 3,   'COMPCOD',   $compcod),
            $this->gateway->charParam('BOTH', 10,  'USER',      $iSeriesUser),
            $this->gateway->charParam('BOTH', 1,   'ERRORS',    $errors),
            $this->gateway->charParam('BOTH', 100, 'ERRMSG',    $errmsg),
            $this->gateway->charParam('BOTH', 20,  'ERRFIELD',  $errfield),
            $this->gateway->charParam('BOTH', 5,   'IDBUYR',    $buyr),
            $this->gateway->charParam('BOTH', 5,   'IDLOCN',    $locn),
            $this->gateway->charParam('BOTH', 10,  'IDSUPL',    $supl),
            $this->gateway->charParam('BOTH', 10,  'IDSUPLSUB', $suplsub),
        ];

        // 5. Call the Action.
        $result = $this->gateway->call('AC_DELSUPL', $k3sobj, $params);

        if ($result === false) {
            // Toolkit call itself failed — connection issue, not a
            // business-logic failure. This is a 500.
            return new JsonResponse([
                'success'  => false,
                'errors'   => 'Y',
                'errmsg'   => 'Toolkit error: ' . $this->gateway->lastError(),
                'errfield' => '',
            ], 500);
        }

        // 6. Translate the envelope into the response.
        $rvErrors   = trim($result['io_param']['ERRORS']);
        $rvErrmsg   = trim($result['io_param']['ERRMSG']);
        $rvErrfield = trim($result['io_param']['ERRFIELD']);

        $statusCode = $rvErrors === 'Y' ? 400 : 200;

        return new JsonResponse([
            'success'  => $rvErrors === 'N',
            'errors'   => $rvErrors,
            'errmsg'   => $rvErrmsg,
            'errfield' => $rvErrfield,
        ], $statusCode);
    }
}
```

Walk through this top to bottom. The handler is the thinnest possible wrapper around the Action. It does the things only PHP can do — request parsing, authentication, JSON serialization — and delegates everything else to the IBM i.

### What the handler is responsible for

**Boundary validation.** The handler checks that `buyr`, `locn`, and `supl` are non-empty before calling the Action. This isn't a duplicate of the RPG's validation; it's a cheap pre-check that catches "completely missing input" without burning a toolkit call. Real business validation — *does this buyer exist?*, *is this supplier deletable?* — happens in the RPG.

**Authentication and authorization mapping.** The web user is whoever signed in through the application's auth flow. The IBM i needs an `IBMI` user profile (`USER` parameter). The handler resolves that mapping (`$this->user->getIbmIProfile()`). Authorization — *is this user allowed to delete suppliers?* — can happen here in PHP, or it can happen in the RPG, or both. K3S puts the *coarse* check (does the user have the supplier-delete permission?) in PHP and the *fine-grained* check (does this user have access to *this particular supplier*?) in the RPG.

**Envelope construction.** The toolkit needs the parameter list in exact order, with exact lengths. The handler builds it. Note the `BOTH` direction — IBM i Toolkit parameters are bidirectional by default, which matches our envelope (input *and* output via `ERRORS`/`ERRMSG`/`ERRFIELD`).

**HTTP status mapping.** `ERRORS=N` → 200. `ERRORS=Y` → 400 (the request was syntactically valid but semantically rejected — a normal validation failure is a client error). Toolkit *itself* failing → 500. This is a small, opinionated mapping, but it's consistent across every Action handler in the codebase.

### What the handler is *not* responsible for

The handler does not:

- Re-implement supplier-deletion validation. That's the RPG's job.
- Decide whether the supplier is deletable. The RPG checks for open purchase orders. The handler trusts the RPG's answer.
- Cache the response. (Commands are not cacheable. Queries might be, but that's a per-Action decision and lives elsewhere.)
- Modify the response shape. The PHP envelope (`success`, `errors`, `errmsg`, `errfield`) is a one-to-one translation of the IBM i envelope.

The discipline is to keep the handler *boring*. If a handler grows beyond ~80 lines, something has leaked across the boundary in the wrong direction — usually validation that should be in the RPG, or response shaping that should be in a separate transformer.

## Wiring it up in Mezzio

Three files, beyond the handler itself:

### The config provider

```php
namespace K3sApp\Suppliers;

use K3sApp\Suppliers\Handler\DeleteSupplierHandler;

class ConfigProvider
{
    public function __invoke(): array
    {
        return [
            'dependencies' => $this->getDependencies(),
        ];
    }

    public function getDependencies(): array
    {
        return [
            'factories' => [
                DeleteSupplierHandler::class
                    => Handler\DeleteSupplierHandlerFactory::class,
            ],
        ];
    }
}
```

### The factory

```php
namespace K3sApp\Suppliers\Handler;

use Psr\Container\ContainerInterface;
use K3sBase\IbmI\ToolkitGateway;
use K3sBase\Auth\AuthenticatedUser;

class DeleteSupplierHandlerFactory
{
    public function __invoke(ContainerInterface $container): DeleteSupplierHandler
    {
        return new DeleteSupplierHandler(
            $container->get(ToolkitGateway::class),
            $container->get(AuthenticatedUser::class),
            $container->get('config')
        );
    }
}
```

### The route

In `config/routes.php`:

```php
use K3sApp\Suppliers\Handler\DeleteSupplierHandler;

return function (
    \Mezzio\Application $app,
    \Mezzio\MiddlewareFactory $factory,
    \Psr\Container\ContainerInterface $container
): void {
    // ... other routes ...

    $app->post('/api/suppliers/delete', DeleteSupplierHandler::class, 'suppliers.delete');
};
```

That's it. A POST to `/api/suppliers/delete` invokes the handler, which calls `AC_DELSUPL`, which calls `AR_DELSUPL`, which deletes the supplier (or returns a clean error). The whole stack from React click to RPG `delete` op is on the order of 30–80ms in production, with most of that being the network round-trip and the toolkit's XMLSERVICE overhead.

## The toolkit gateway

The `ToolkitGateway` referenced above is a thin K3S abstraction over the official IBM i Toolkit. It exists because the toolkit's API is verbose, and we don't want every handler littered with `$toolkit->AddParameterChar(...)` calls. The gateway exposes `charParam()`, `packDecParam()`, `call()`, and `lastError()` and handles connection lifecycle, persistent connections, and error normalization.

A simplified version:

```php
namespace K3sBase\IbmI;

use ToolkitApi\Toolkit;

final class ToolkitGateway
{
    public function __construct(private Toolkit $toolkit) {}

    public function charParam(
        string $direction,
        int $size,
        string $name,
        string $value
    ): array {
        return $this->toolkit->AddParameterChar(
            $direction, $size, $name, $name, $value
        );
    }

    public function call(string $program, string $library, array $params): array|false
    {
        return $this->toolkit->PgmCall($program, $library, $params, null, null);
    }

    public function lastError(): string
    {
        return $this->toolkit->getLastError() ?? '';
    }
}
```

The gateway is itself constructed by a factory that pulls the `Toolkit` instance from a connection pool (one persistent connection per worker process is the right default). Connection management is out of scope here, but: persistent, pooled, with a sane timeout. Don't open a new connection per request; the cost of the open is many times the cost of the actual call.

{: .tip }
> If you're seeing 200ms+ per Action API call in PHP, persistent connections are almost certainly the missing piece. A cold toolkit connection is expensive. A warm one — reused across requests within the same PHP-FPM worker — is fast.

## Common patterns across handlers

Once you've written one handler, the others are essentially copies with different parameters. K3S maintains an internal `AbstractActionHandler` that captures the boilerplate:

```php
abstract class AbstractActionHandler implements RequestHandlerInterface
{
    public function __construct(
        protected ToolkitGateway $gateway,
        protected AuthenticatedUser $user,
        protected array $config
    ) {}

    abstract protected function programName(): string;
    abstract protected function buildParams(array $body, array &$out): array;

    public function handle(ServerRequestInterface $request): ResponseInterface
    {
        $body  = $request->getParsedBody() ?? [];
        $out   = ['ERRORS' => '', 'ERRMSG' => '', 'ERRFIELD' => ''];

        $params = $this->buildParams($body, $out);
        $result = $this->gateway->call(
            $this->programName(),
            $this->config['k3s_settings']['k3sobj'],
            $params
        );

        if ($result === false) {
            return new JsonResponse([
                'success' => false,
                'errors'  => 'Y',
                'errmsg'  => 'Toolkit error: ' . $this->gateway->lastError(),
            ], 500);
        }

        $rvErrors   = trim($result['io_param']['ERRORS']);
        $rvErrmsg   = trim($result['io_param']['ERRMSG']);
        $rvErrfield = trim($result['io_param']['ERRFIELD']);

        return new JsonResponse([
            'success'  => $rvErrors === 'N',
            'errors'   => $rvErrors,
            'errmsg'   => $rvErrmsg,
            'errfield' => $rvErrfield,
            'data'     => $this->extractData($result['io_param']),
        ], $rvErrors === 'Y' ? 400 : 200);
    }

    protected function extractData(array $ioParam): array
    {
        return [];   // overridden by query handlers
    }
}
```

Concrete handlers then become tiny:

```php
final class DeleteSupplierHandler extends AbstractActionHandler
{
    protected function programName(): string { return 'AC_DELSUPL'; }

    protected function buildParams(array $body, array &$out): array
    {
        $envelope = $this->envelopeParams($out);
        return array_merge($envelope, [
            $this->gateway->charParam('BOTH', 5,  'IDBUYR',    $body['buyr']    ?? ''),
            $this->gateway->charParam('BOTH', 5,  'IDLOCN',    $body['locn']    ?? ''),
            $this->gateway->charParam('BOTH', 10, 'IDSUPL',    $body['supl']    ?? ''),
            $this->gateway->charParam('BOTH', 10, 'IDSUPLSUB', $body['suplsub'] ?? ''),
        ]);
    }
}
```

Once the abstract is in place, adding a new Action handler is fifteen lines. This is the practical payoff of the discipline: the pattern compounds in a good way.

## Query handlers and result sets

Query handlers (`GetSupplierHandler`, `ListSuppliersHandler`) follow the same skeleton with one twist: they override `extractData()` to pull `RV*` fields out of the response and shape them for the frontend.

A `GET` handler:

```php
final class GetSupplierHandler extends AbstractActionHandler
{
    protected function programName(): string { return 'AC_GETSUPL'; }

    protected function buildParams(array $body, array &$out): array
    {
        // ... ID params plus RV* output params
    }

    protected function extractData(array $ioParam): array
    {
        return [
            'name'      => trim($ioParam['RVSUPLNAM']),
            'phone'     => trim($ioParam['RVPHONE']),
            'address1'  => trim($ioParam['RVADDR1']),
            'address2'  => trim($ioParam['RVADDR2']),
            'city'      => trim($ioParam['RVCITY']),
            'state'     => trim($ioParam['RVSTATE']),
            'zip'       => trim($ioParam['RVZIP']),
            'leadTime'  => (int) $ioParam['RVLEADTIME'],
            'orderCycle'=> (int) $ioParam['RVORDCYCLE'],
        ];
    }
}
```

A `LST` handler that uses an SQL result set is more involved (you'd use `getResultSets()` on the toolkit and iterate), but the structure is the same: dispatch the call, transform the result, return JSON.

## What you have now

Action APIs callable from any web client that can speak HTTP. A consistent JSON envelope. Error handling that's mechanical and uniform. A handler abstract that makes new Actions cheap to add.

In the next chapter we expose the same Actions to a fundamentally different kind of caller: an AI agent, through the Model Context Protocol. The interesting result is that we don't have to change the RPG, the CL, or even most of the PHP. The Action API is already the right shape for MCP.

[Next: Calling Actions from MCP]({% link 07-calling-from-mcp.md %}){: .btn .btn-primary }
