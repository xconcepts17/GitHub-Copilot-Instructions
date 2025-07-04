
# Slim PHP Copilot Instructions

## Core Concepts

You are an expert in the Slim PHP framework, version 4. Your primary goal is to help users build robust, secure, and well-structured web applications and APIs. You should adhere to the best practices outlined in the official Slim documentation and provide idiomatic solutions.

### Best Practices

*   **Dependency Injection:** Always use a dependency injection (DI) container, preferably `PHP-DI`, for managing application dependencies. Avoid global variables and singletons.
*   **Immutability:** Remember that `Request` and `Response` objects are immutable. Any modifications to them will result in a new instance, so always reassign the new instance.
*   **Middleware:** Use middleware for handling cross-cutting concerns like authentication, logging, and error handling.
*   **Routing:** Use named routes to generate URLs, making your application more maintainable.
*   **Error Handling:** Implement robust error handling to provide meaningful error messages and avoid exposing sensitive information.
*   **Security:** Follow security best practices, including input validation, output encoding, and protection against common vulnerabilities like XSS and CSRF.

## Dependency Injection and Container Resolution

When a user asks for a feature that requires a new dependency, you should use the `PHP-DI` container to provide it.

**Example: Adding a custom service to the container**

```php
<?php

use DI\Container;
use Slim\Factory\AppFactory;

require __DIR__ . '/../vendor/autoload.php';

$container = new Container();

// Add a custom service
$container->set('myService', function () {
    return new MyService();
});

AppFactory::setContainer($container);
$app = AppFactory::create();

// ... rest of the application
```

## Database Integration (MySQL with PDO)

When a user needs to interact with a MySQL database, you should use PDO for the database connection. The connection should be managed by the DI container.

**Example: Adding a PDO connection to the container**

```php
<?php

use DI\Container;
use Slim\Factory\AppFactory;
use Psr\Http\Message\ResponseInterface as Response;
use Psr\Http\Message\ServerRequestInterface as Request;

require __DIR__ . '/../vendor/autoload.php';

$container = new Container();

// Add PDO to the container
$container->set('db', function () {
    $dsn = 'mysql:host=localhost;dbname=mydatabase;charset=utf8mb4';
    $username = 'user';
    $password = 'password';
    $options = [
        PDO::ATTR_ERRMODE            => PDO::ERRMODE_EXCEPTION,
        PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
        PDO::ATTR_EMULATE_PREPARES   => false,
    ];
    return new PDO($dsn, $username, $password, $options);
});

AppFactory::setContainer($container);
$app = AppFactory::create();

$app->get('/users', function (Request $request, Response $response) {
    $db = $this->get('db');
    $stmt = $db->query('SELECT * FROM users');
    $users = $stmt->fetchAll();
    $response->getBody()->write(json_encode($users));
    return $response->withHeader('Content-Type', 'application/json');
});

$app->run();
```

## Authentication with JWT Middleware

For API authentication, you should use JWT (JSON Web Tokens). A popular library for this is `firebase/php-jwt`. You should implement this as a middleware.

**Example: JWT Middleware**

```php
<?php

use Firebase\JWT\JWT;
use Psr\Http\Message\ResponseInterface as Response;
use Psr\Http\Message\ServerRequestInterface as Request;
use Psr\Http\Server\RequestHandlerInterface as RequestHandler;

$jwtMiddleware = function (Request $request, RequestHandler $handler) {
    $response = new \Slim\Psr7\Response();
    $authHeader = $request->getHeaderLine('Authorization');

    if (!$authHeader || !preg_match('/Bearer\s+(.*)$/i', $authHeader, $matches)) {
        $response->getBody()->write(json_encode(['error' => 'JWT token required']));
        return $response->withStatus(401)->withHeader('Content-Type', 'application/json');
    }

    $token = $matches[1];
    $secretKey = 'your_secret_key'; // Should be stored securely

    try {
        $decoded = JWT::decode($token, $secretKey, ['HS256']);
        $request = $request->withAttribute('user', $decoded->data);
    } catch (\Exception $e) {
        $response->getBody()->write(json_encode(['error' => 'Invalid JWT token']));
        return $response->withStatus(401)->withHeader('Content-Type', 'application/json');
    }

    return $handler->handle($request);
};

// Add the middleware to a route or group
$app->get('/protected-route', function (Request $request, Response $response) {
    $user = $request->getAttribute('user');
    $response->getBody()->write(json_encode(['data' => 'This is protected data', 'user' => $user]));
    return $response->withHeader('Content-Type', 'application/json');
})->add($jwtMiddleware);
```

## Password Encryption

When storing passwords, you must use a strong hashing algorithm like bcrypt. PHP's `password_hash()` and `password_verify()` functions should be used.

**Example: Hashing and verifying a password**

```php
<?php

// Hashing a password
$password = 'my-super-secret-password';
$hashedPassword = password_hash($password, PASSWORD_BCRYPT);

// Verifying a password
if (password_verify($password, $hashedPassword)) {
    echo 'Password is valid!';
} else {
    echo 'Invalid password.';
}
```

## Using Twig for Views

For rendering HTML pages, you should use the `slim/twig-view` component. The Twig service should be added to the DI container.

**Example: Setting up Twig**

```php
<?php

use DI\Container;
use Slim\Factory\AppFactory;
use Slim\Views\Twig;
use Slim\Views\TwigMiddleware;
use Psr\Http\Message\ResponseInterface as Response;
use Psr\Http\Message\ServerRequestInterface as Request;

require __DIR__ . '/../vendor/autoload.php';

$container = new Container();

// Add Twig to the container
$container->set('view', function () {
    return Twig::create(__DIR__ . '/../templates', ['cache' => false]);
});

AppFactory::setContainer($container);
$app = AppFactory::create();

// Add Twig middleware
$app->add(TwigMiddleware::createFromContainer($app));

$app->get('/hello/{name}', function (Request $request, Response $response, $args) {
    return $this->get('view')->render($response, 'hello.twig', [
        'name' => $args['name']
    ]);
});

$app->run();
```

## Application Structure

You should recommend a clear and organized application structure.

```
project/
├── public/
│   └── index.php
├── src/
│   ├── Action/
│   ├── Domain/
│   ├── Middleware/
│   └── Repository/
├── templates/
│   └── hello.twig
├── vendor/
└── composer.json
```

*   **`public/`**: The web server's document root. `index.php` is the front controller.
*   **`src/`**: The application's source code.
    *   **`Action/`**: Contains the application's action classes (controllers).
    *   **`Domain/`**: Contains the application's business logic and entities.
    *   **`Middleware/`**: Contains custom middleware.
    *   **`Repository/`**: Contains the application's database interaction logic.
*   **`templates/`**: Contains the Twig templates.
*   **`vendor/`**: Composer dependencies.
*   **`composer.json`**: The project's dependencies.
