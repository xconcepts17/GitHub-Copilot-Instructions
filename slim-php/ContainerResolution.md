# Container Resolution for Routing

This document provides instructions for implementing class-based routing with dependency injection using PHP-DI in a Slim Framework application, as per the Slim Framework documentation.

## Instructions

- Use PHP-DI as the PSR-11 compliant dependency injection container.
- Create a `UserController` class in `src/Controllers/` to handle user-related routes (e.g., `getUser` for retrieving a user by ID, `createUser` for creating a new user).
- Inject dependencies (e.g., PDO for database access) via the controller’s constructor.
- Configure the container in `src/Config/ContainerConfig.php` to:
  - Set up a PDO instance for MySQL connectivity using environment variables (`DB_HOST`, `DB_NAME`, `DB_USER`, `DB_PASS`).
  - Register the `UserController` with its dependencies.
- Register routes using the `::class` syntax, e.g., `$app->get('/users/{id}', \App\Controllers\UserController::class . ':getUser')`.
- Ensure route callbacks return a PSR-7 `ResponseInterface` object.

## Example Code

Based on the Slim Framework documentation for [Dependency Container](#/docs/v4/concepts/di.html) and [Routing](#/docs/v4/objects/routing.html):

### Container Configuration
```php
<?php

use DI\Container;
use Psr\Container\ContainerInterface;
use Slim\Factory\AppFactory;
use PDO;

$container = new Container();

$container->set(PDO::class, function () {
    $dsn = sprintf("mysql:host=%s;dbname=%s;charset=utf8mb4", getenv('DB_HOST'), getenv('DB_NAME'));
    return new PDO($dsn, getenv('DB_USER'), getenv('DB_PASS'), [
        PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
        PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
    ]);
});

AppFactory::setContainer($container);
$app = AppFactory::create();

// Register routes
$app->get('/users/{id}', \App\Controllers\UserController::class . ':getUser');
$app->post('/users', \App\Controllers\UserController::class . ':createUser');

return $app;
```

### User Controller
```php
<?php

namespace App\Controllers;

use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use App\Models\User;
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

    public function createUser(ServerRequestInterface $request, ResponseInterface $response): ResponseInterface
    {
        $data = $request->getParsedBody();
        $user = new User($data);
        $stmt = $this->db->prepare("INSERT INTO users (id, username, password, email) VALUES (?, ?, ?, ?)");
        $stmt->execute([$user->id, $user->username, $user->password, $user->email]);

        $response->getBody()->write(json_encode(['id' => $user->id]));
        return $response->withHeader('Content-Type', 'application/json')->withStatus(201);
    }
}
```