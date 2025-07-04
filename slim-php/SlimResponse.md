# The Response

This document outlines how to generate HTTP responses in a Slim Framework application, based on the Slim 4 documentation for [The Response](#/docs/v4/objects/response.html).

## Instructions

- Slim uses PSR-7 `ResponseInterface` objects to represent HTTP responses.
- Route callbacks must return a `ResponseInterface` object.
- Modify responses using:
  - `withHeader()` to set headers (e.g., `Content-Type: application/json`).
  - `withStatus()` to set HTTP status codes (e.g., 200, 201, 404).
  - `getBody()->write()` to write response content (e.g., JSON).
- Use `Slim\Psr7\Factory\ResponseFactory` to create responses in middleware or error handling.
- In the user management API, return JSON responses with appropriate status codes (e.g., 201 for user creation, 404 for user not found, 401 for invalid JWT).

## Example Code

### Generating Responses
```php
<?php

namespace App\Controllers;

use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Slim\Psr7\Factory\ResponseFactory;

class UserController
{
    public function getUser(ServerRequestInterface $request, ResponseInterface $response, string $id): ResponseInterface
    {
        // Simulate database query
        $user = ['id' => $id, 'username' => 'example'];

        if (!$user) {
            throw new \Slim\Exception\HttpNotFoundException($request, "User not found");
        }

        $response->getBody()->write(json_encode($user));
        return $response->withHeader('Content-Type', 'application/json')->withStatus(200);
    }

    public function createUser(ServerRequestInterface $request, ResponseInterface $response): ResponseInterface
    {
        $data = $request->getParsedBody();
        // Process user creation
        $response->getBody()->write(json_encode(['id' => 'new-user-id']));
        return $response->withHeader('Content-Type', 'application/json')->withStatus(201);
    }
}

// Example in Middleware
use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Server\RequestHandlerInterface;

class JwtMiddleware
{
    public function __invoke(ServerRequestInterface $request, RequestHandlerInterface $handler): ResponseInterface
    {
        $authHeader = $request->getHeaderLine('Authorization');
        if (!$authHeader) {
            $response = (new ResponseFactory())->createResponse(401);
            $response->getBody()->write(json_encode(['error' => 'Missing token']));
            return $response->withHeader('Content-Type', 'application/json');
        }
        return $handler->handle($request);
    }
}
```

- Always set `Content-Type: application/json` for API responses.
- Use appropriate status codes (e.g., 201 for creation, 401 for unauthorized).