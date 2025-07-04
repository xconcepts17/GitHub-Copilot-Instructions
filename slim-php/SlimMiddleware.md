# Packaged Middleware

This document outlines the use of Slim’s packaged middleware in a Slim Framework application, based on the Slim 4 documentation for [Middleware](#/docs/v4/concepts/middleware.html).

## Instructions

- Slim provides built-in middleware:
  - `BodyParsingMiddleware`: Parses JSON, form data, and XML request bodies for `getParsedBody()`.
  - `RoutingMiddleware`: Resolves and dispatches routes.
  - `ErrorMiddleware`: Handles exceptions and errors, supporting custom HTTP exceptions.
- Add middleware in `public/index.php` in the following order:
  1. `BodyParsingMiddleware` for request body parsing.
  2. `RoutingMiddleware` for route resolution.
  3. `ErrorMiddleware` for error handling (set `displayErrorDetails`, `logErrors`, `logErrorDetails` to `true` for development).
  4. `TwigMiddleware` (from `slim/twig-view`) for rendering templates (e.g., Swagger UI).
- Use `add()` method to apply middleware to specific routes (e.g., `JwtMiddleware` for protected routes).
- In the user management API, ensure `BodyParsingMiddleware` is added for JSON parsing in POST requests and `ErrorMiddleware` handles exceptions like `HttpNotFoundException`.

## Example Code

### Middleware Setup
```php
<?php

use Slim\Factory\AppFactory;
use Slim\Views\Twig;
use Slim\Views\TwigMiddleware;
use App\Middleware\JwtMiddleware;

require __DIR__ . '/../vendor/autoload.php';
require __DIR__ . '/../src/Config/ContainerConfig.php';

// Middleware
$app->addBodyParsingMiddleware();
$app->addRoutingMiddleware();
$app->addErrorMiddleware(true, true, true);

// Twig Middleware for Swagger UI
$twig = Twig::create(__DIR__ . '/../templates', ['cache' => false]);
$app->add(TwigMiddleware::create($app, $twig));

// Routes
$app->get('/users/{id}', \App\Controllers\UserController::class . ':getUser')->add(new JwtMiddleware());
$app->post('/users', \App\Controllers\UserController::class . ':createUser');
$app->post('/login', \App\Controllers\AuthController::class . ':login');
$app->get('/docs', function ($request, $response) {
    $view = Twig::fromRequest($request);
    return $view->render($response, 'swagger.html.twig', [
        'swagger_yaml' => file_get_contents(__DIR__ . '/../swagger.yaml'),
    ]);
});

$app->run();
```

### Error Handling Example
```php
<?php

namespace App\Controllers;

use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;

class UserController
{
    public function getUser(ServerRequestInterface $request, ResponseInterface $response, string $id): ResponseInterface
    {
        // Simulate database query
        $user = null;
        if (!$user) {
            throw new \Slim\Exception\HttpNotFoundException($request, "User not found");
        }

        $response->getBody()->write(json_encode($user));
        return $response->withHeader('Content-Type', 'application/json');
    }
}
```

- Set `displayErrorDetails` to `false` in production for `ErrorMiddleware`.
- Ensure `BodyParsingMiddleware` is added before routes that use `getParsedBody()`.