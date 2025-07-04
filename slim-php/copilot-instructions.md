# GitHub Copilot Instructions for Slim Framework Application

This document provides high-level instructions for implementing a RESTful API using the Slim Framework, with specific requirements for routing, database structure, authentication, API documentation, and core Slim concepts. To reduce token usage, detailed instructions for each topic are provided in separate documents. Agents should refer only to the relevant topic-specific documentation linked below when implementing each feature.

## Project Overview

The application is a RESTful API for user management with the following features:

- **Container Resolution for Routing**: Use class-based controllers with dependency injection via PHP-DI.
- **MySQL Database Structure**: Define a MySQL schema for user data with UUIDs.
- **JWT Middleware**: Implement JSON Web Token authentication for secure routes.
- **Password Hashing Policy**: Enforce secure password hashing for user records.
- **UUID Generation**: Generate unique UUIDs for each database record.
- **Swagger Integration**: Use Swagger for API documentation with centralized schemas.
- **Slim Core Concepts**: Implement request handling, response generation, routing, and middleware as per Slim 4 documentation.

## Prerequisites

- PHP 8.1 or higher.
- Composer for dependency management.
- MySQL database.
- Install Composer packages: `slim/slim:^4.12`, `slim/psr7`, `php-di/php-di`, `firebase/php-jwt`, `ramsey/uuid`, `slim/twig-view`, `nelmio/api-doc-bundle`.

## Directory Structure

```
project/
├── src/
│   ├── Controllers/
│   ├── Middleware/
│   ├── Models/
│   ├── Schemas/
│   └── Config/
├── templates/
├── public/
│   └── index.php
├── composer.json
├── .env
└── swagger.yaml
```

## Instructions by Topic

Agents should refer to the following topic-specific documentation files for detailed implementation guidance. Only consult the documents relevant to the task at hand to optimize token usage.

### Project-Specific Features

1. **[Container Resolution for Routing]**: Instructions for setting up PHP-DI and class-based controllers.
   - [ContainerResolution.md](ContainerResolution.md)
2. **[MySQL Database Structure]**: Schema design and UUID/password hashing policies.
   - [DatabaseStructure.md](DatabaseStructure.md)
3. **[JWT Middleware]**: Implementation of JWT authentication middleware and login endpoint.
   - [JwtMiddleware.md](JwtMiddleware.md)
4. **[Swagger Integration]**: Setup for Swagger documentation with centralized schemas.
   - [SwaggerIntegration.md](SwaggerIntegration.md)

### Slim Framework Core Concepts

5. **[The Request]**: Handling HTTP requests in Slim.
   - [SlimRequest.md](SlimRequest.md)
6. **[The Response]**: Generating HTTP responses in Slim.
   - [SlimResponse.md](SlimResponse.md)
7. **[Routing]**: Defining and managing routes in Slim.
   - [SlimRouting.md](SlimRouting.md)
8. **[Packaged Middleware]**: Using Slim’s built-in middleware.
   - [SlimMiddleware.md](SlimMiddleware.md)

## General Guidelines

- Use environment variables (`.env`) for sensitive data (e.g., `DB_HOST`, `JWT_SECRET`).
- Add middleware in `public/index.php` in this order: `BodyParsingMiddleware`, `RoutingMiddleware`, `ErrorMiddleware`, `TwigMiddleware`.
- Ensure PSR-7 compliance for all responses with appropriate `Content-Type` headers.
- Handle errors using `ErrorMiddleware` and throw HTTP exceptions (e.g., `HttpNotFoundException`).
- Test routes to verify JWT protection, password hashing, UUID uniqueness, Swagger accuracy, and Slim core functionality.
- For production, set `displayErrorDetails` to `false` in `ErrorMiddleware` and enable Twig caching.
