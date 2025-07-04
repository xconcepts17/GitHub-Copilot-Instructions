# Routing

This document provides instructions for defining and managing routes in a Slim Framework application, based on the Slim 4 documentation for [Routing](#/docs/v4/objects/routing.html).

## Instructions

- Define routes using `$app->get()`, `$app->post()`, etc., in `public/index.php` or a configuration file like `src/Config/ContainerConfig.php`.
- Use class-based controllers with `::class` syntax for container-resolved routing (e.g., `\App\Controllers\UserController::class . ':method'`).
- Support route parameters (e.g., `{id}`) and access them via `$args` in route callbacks or `$request->getAttribute()`.
- Use HTTP methods appropriately:
  - GET for retrieving data (e.g., `/users/{id}`).
  - POST for creating resources (e.g., `/users`, `/login`).
- In the user management API, define routes for:
  - `GET /users/{id}`: Retrieve a user by UUID.
  - `POST /users`: Create a new user.
  - `POST /login`: Generate a JWT token.
  - `GET /docs`: Serve Swagger UI.

## Example Code

### Route Definitions
```php
<?php

use Slim\Factory\AppFactory;
use App\Middleware\JwtMiddleware;

require __DIR__ . '/../vendor/autoload.php';
require __DIR__ . '/../src/Config/ContainerConfig.php';

$app->addBodyParsingMiddleware();
$app->addRoutingMiddleware();
$app->addErrorMiddleware(true, true, true);

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

### Controller with Route Parameters
```php
<?php

namespace App\Controllers;

use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use PDO;

class UserController
{
    private PDO $db;

    public function __construct(PDO $db)
    {
        $this->db = $db;
    }

    public function getUser(ServerRequestInterface $request, ResponseInterface $response, string $id): ResponseInterface
    {
        $stmt = $this->db->prepare("SELECT * FROM users WHERE id = ?");
        $stmt->execute([$id]);
        $user = $stmt->fetch(PDO::FETCH_ASSOC);

        if (!$user) {
            throw new \Slim\Exception\HttpNotFoundException($request, "User not found");
        }

        $response->getBody()->write(json_encode($user));
        return $response->withHeader('Content-Type', 'application/json');
    }
}
```

- Use `RoutingMiddleware` to enable route resolution.
- Apply `JwtMiddleware` to protected routes like `GET /users/{id}`.