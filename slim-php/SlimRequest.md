# The Request

This document provides instructions for handling HTTP requests in a Slim Framework application, based on the Slim 4 documentation for [The Request](#/docs/v4/objects/request.html).

## Instructions

- Slim uses PSR-7 `ServerRequestInterface` objects to represent HTTP requests.
- Access request data (e.g., headers, query parameters, body) in route callbacks via the `$request` parameter.
- Use methods like:
  - `getMethod()` to retrieve the HTTP method (e.g., GET, POST).
  - `getUri()` to get the request URI.
  - `getQueryParams()` to access query string parameters.
  - `getParsedBody()` to access parsed JSON or form data (requires `BodyParsingMiddleware`).
  - `getHeaderLine('Header-Name')` to retrieve header values.
  - `getAttribute()` to access route parameters or middleware-added attributes (e.g., JWT payload).
- In the user management API, use `getParsedBody()` to retrieve user data from POST requests (e.g., for `/users` or `/login`) and `getAttribute()` for JWT-decoded user data in protected routes.

## Example Code

### Accessing Request Data
```php
<?php

namespace App\Controllers;

use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;

class UserController
{
    public function createUser(ServerRequestInterface $request, ResponseInterface $response): ResponseInterface
    {
        $data = $request->getParsedBody(); // Access JSON/form data
        $username = $data['username'] ?? null;
        $email = $data['email'] ?? null;

        // Validate and process data
        $response->getBody()->write(json_encode(['message' => "Received user: $username"]));
        return $response->withHeader('Content-Type', 'application/json')->withStatus(201);
    }

    public function getUser(ServerRequestInterface $request, ResponseInterface $response, string $id): ResponseInterface
    {
        $userId = $request->getAttribute('user')->sub; // Access JWT payload
        $queryParams = $request->getQueryParams(); // Access query parameters
        $method = $request->getMethod(); // Get HTTP method

        $response->getBody()->write(json_encode(['id' => $id, 'userId' => $userId]));
        return $response->withHeader('Content-Type', 'application/json');
    }
}
```

- Ensure `BodyParsingMiddleware` is added to parse JSON/form data for `getParsedBody()`.
- Use `getAttribute('user')` to access JWT payload added by `JwtMiddleware`.