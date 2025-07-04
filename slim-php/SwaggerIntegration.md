# Swagger Integration

This document outlines the setup for Swagger API documentation with centralized schema definitions in a Slim Framework application, based on the Slim Framework documentation for [Twig Templates](#/docs/v4/features/twig-view.html).

## Instructions

- Install `nelmio/api-doc-bundle` and `slim/twig-view` via Composer.
- Create a `swagger.yaml` file in the project root using OpenAPI 3.0.0 to define:
  - Paths: `/users` (POST), `/users/{id}` (GET), `/login` (POST).
  - A `User` schema in `components/schemas/` with properties: `id` (UUID), `username` (string), `email` (email format), `created_at` (date-time).
  - Responses for each endpoint (e.g., 200, 201, 401, 404) with JSON schemas referencing the `User` schema.
- Create a `GET /docs` route to render Swagger UI using a Twig template (`swagger.html.twig` in `templates/`).
- In the Twig template, load `swagger.yaml` and render it with Swagger UI via a CDN (e.g., `unpkg.com/swagger-ui-dist`).
- Add `TwigMiddleware` to the Slim app for rendering templates.

## Example Code

### Swagger YAML
```yaml
openapi: 3.0.0
info:
  title: Slim Framework API
  version: 1.0.0
paths:
  /users:
    post:
      summary: Create a new user
      requestBody:
        content:
          application/json:
            schema:
              $ref: #'components/schemas/User'
      responses:
        '201':
          description: User created
          content:
            application/json:
              schema:
                type: object
                properties:
                  id:
                    type: string
                    format: uuid
  /users/{id}:
    get:
      summary: Get a user by ID
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: string
            format: uuid
      responses:
        '200':
          description: User details
          content:
            application/json:
              schema:
                $ref: #'components/schemas/User'
        '404':
          description: User not found
  /login:
    post:
      summary: User login
      requestBody:
        content:
          application/json:
            schema:
              type: object
              properties:
                username:
                  type: string
                password:
                  type: string
      responses:
        '200':
          description: Login successful
          content:
            application/json:
              schema:
                type: object
                properties:
                  token:
                    type: string
        '401':
          description: Invalid credentials
components:
  schemas:
    User:
      type: object
      properties:
        id:
          type: string
          format: uuid
        username:
          type: string
        email:
          type: string
          format: email
        created_at:
          type: string
          format: date-time
```

### Swagger UI Route
```php
<?php

use Slim\Factory\AppFactory;
use Slim\Views\Twig;
use Slim\Views\TwigMiddleware;

require __DIR__ . '/../vendor/autoload.php';
require __DIR__ . '/../src/Config/ContainerConfig.php';

$app->addBodyParsingMiddleware();
$app->addRoutingMiddleware();
$app->addErrorMiddleware(true, true, true);

// Add Twig for Swagger UI
$twig = Twig::create(__DIR__ . '/../templates', ['cache' => false]);
$app->add(TwigMiddleware::create($app, $twig));

// Swagger UI route
$app->get('/docs', function ($request, $response) {
    $view = Twig::fromRequest($request);
    return $view->render($response, 'swagger.html.twig', [
        'swagger_yaml' => file_get_contents(__DIR__ . '/../swagger.yaml'),
    ]);
});

$app->run();
```

### Swagger UI Template
```html
<!DOCTYPE html>
<html>
<head>
    <title>API Documentation</title>
    <link rel="stylesheet" type="text/css" href="https://unpkg.com/swagger-ui-dist@3/swagger-ui.css">
    <script src="https://unpkg.com/swagger-ui-dist@3/swagger-ui-bundle.js"></script>
    <script>
        window.onload = function() {
            SwaggerUIBundle({
                spec: {{ swagger_yaml | json_encode | raw }},
                dom_id: '#swagger-ui',
                presets: [
                    SwaggerUIBundle.presets.apis,
                    SwaggerUIBundle.SwaggerUIStandalonePreset
                ]
            });
        }
    </script>
</head>
<body>
    <div id="swagger-ui"></div>
</body>
</html>
```