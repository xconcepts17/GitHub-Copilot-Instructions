# JWT Middleware

This document provides instructions for implementing JSON Web Token (JWT) authentication middleware and a login endpoint in a Slim Framework application, aligned with the Slim Framework documentation for [Middleware](#/docs/v4/concepts/middleware.html).

## Instructions

- Install `firebase/php-jwt` via Composer.
- Create a `JwtMiddleware` class in `src/Middleware/` to:
  - Check for a `Bearer` token in the `Authorization` header.
  - Validate the JWT using `JWT::decode()` with a secret key from `JWT_SECRET` environment variable.
  - Attach the decoded payload to the request as a `user` attribute.
  - Return a 401 response with a JSON error if the token is missing or invalid.
- Create an `AuthController` in `src/Controllers/` with a `login` method to:
  - Handle POST requests with `username` and `password`.
  - Verify credentials using `password_verify()` against the database.
  - Generate a JWT with `sub` (user ID), `iat` (issued at), and `exp` (1-hour expiration).
  - Return the JWT in a JSON response (200 status) or a 401 error for invalid credentials.
- Apply `JwtMiddleware` to protected routes using the `add()` method.

## Example Code

### JWT Middleware
```php
<?php

namespace App\Middleware;

use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Server\RequestHandlerInterface;
use Psr\Http\Message\ResponseInterface;
use Firebase\JWT\JWT;
use Firebase\JWT\Key;
use Slim\Psr7\Factory\ResponseFactory;

class JwtMiddleware
{
    public function __invoke(ServerRequestInterface $request, RequestHandlerInterface $handler): ResponseInterface
    {
        $authHeader = $request->getHeaderLine('Authorization');
        if (!$authHeader || !preg_match('/Bearer\s(\S+)/', $authHeader, $matches)) {
            $response = (new ResponseFactory())->createResponse(401);
            $response->getBody()->write(json_encode(['error' => 'Missing or invalid token']));
            return $response->withHeader('Content-Type', 'application/json');
        }

        try {
            $decoded = JWT::decode($matches[1], new Key(getenv('JWT_SECRET'), 'HS256'));
            $request = $request->withAttribute('user', $decoded);
        } catch (\Exception $e) {
            $response = (new ResponseFactory())->createResponse(401);
            $response->getBody()->write(json_encode(['error' => 'Invalid token']));
            return $response->withHeader('Content-Type', 'application/json');
        }

        return $handler->handle($request);
    }
}
```

### Auth Controller
```php
<?php

namespace App\Controllers;

use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Firebase\JWT\JWT;
use PDO;

class AuthController
{
    private PDO $db;

    public function __construct(PDO $db)
    {
        $this->db = $db;
    }

    public function login(ServerRequestInterface $request, ResponseInterface $response): ResponseInterface
    {
        $data = $request->getParsedBody();
        $stmt = $this->db->prepare("SELECT * FROM users WHERE username = ?");
        $stmt->execute([$data['username']]);
        $user = $stmt->fetch(PDO::FETCH_ASSOC);

        if ($user && password_verify($data['password'], $user['password'])) {
            $payload = [
                'sub' => $user['id'],
                'iat' => time(),
                'exp' => time() + 3600,
            ];
            $token = JWT::encode($payload, getenv('JWT_SECRET'), 'HS256');
            $response->getBody()->write(json_encode(['token' => $token]));
            return $response->withHeader('Content-Type', 'application/json');
        }

        $response->getBody()->write(json_encode(['error' => 'Invalid credentials']));
        return $response->withHeader('Content-Type', 'application/json')->withStatus(401);
    }
}
```