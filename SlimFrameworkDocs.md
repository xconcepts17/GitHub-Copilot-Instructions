# Lifecycle - Slim Framework

# Lifecycle

[Edit This Page](https://github.com/slimphp/Slim-Website/tree/gh-pages/docs/v4/concepts/life-cycle.md)

## Application Life Cycle

### 1\. Instantiation

First, you instantiate the `Slim\App` class. This is the Slim application object. During instantiation, Slim registers default services for each application dependency.

### 2\. Route Definitions

Second, you define routes using the application instance’s `get()`, `post()`, `put()`, `delete()`, `patch()`, `head()`, and `options()` routing methods. These instance methods register a route with the application’s Router object. Each routing method returns the Route instance so you can immediately invoke the Route instance’s methods to add middleware or assign a name.

### 3\. Application Runner

Third, you invoke the application instance’s `run()` method. This method starts the following process:

#### A. Enter Middleware Stack

The `run()` method begins to inwardly traverse the application’s middleware stack. This is a concentric structure of middleware layers that receive (and optionally manipulate) the Environment, Request, and Response objects before (and after) the Slim application runs. The Slim application is the innermost layer of the concentric middleware structure. Each middleware layer is invoked inwardly beginning from the outermost layer.

#### B. Run Application

After the `run()` method reaches the innermost middleware layer, it invokes the application instance and dispatches the current HTTP request to the appropriate application route object. If a route matches the HTTP method and URI, the route’s middleware and callable are invoked. If a matching route is not found, the Not Found or Not Allowed handler is invoked.

#### C. Exit Middleware Stack

After the application dispatch process completes, each middleware layer reclaims control outwardly, beginning from the innermost layer.

#### D. Send HTTP Response

After the outermost middleware layer cedes control, the application instance prepares, serializes, and returns the HTTP response. The HTTP response headers are set with PHP’s native `header()` method, and the HTTP response body is output to the current output buffer.

- [Get Started: Upgrade Guide](/docs/v4/start/upgrade.html)
- [Concepts: PSR-7](/docs/v4/concepts/value-objects.html)

# PSR-7 and Value Objects - Slim Framework

# PSR-7 and Value Objects

[Edit This Page](https://github.com/slimphp/Slim-Website/tree/gh-pages/docs/v4/concepts/value-objects.md)

Slim supports [PSR-7](https://github.com/php-fig/http-message) interfaces for its Request and Response objects. This makes Slim flexible because it can use _any_ PSR-7 implementation. For example, you could return an instance of `GuzzleHttp\Psr7\CachingStream` or any instance returned by the `GuzzleHttp\Psr7\stream_for()` function.

Slim provides its own PSR-7 implementation. However, you are free to [install a third-party implementation](/docs/v4/start/installation.html).

## Value objects

Request and Response objects are [_immutable value objects_](http://en.wikipedia.org/wiki/Value_object). They can be “changed” only by requesting a cloned version that has updated property values. Value objects have a nominal overhead because they must be cloned when their properties are updated. This overhead does not affect performance in any meaningful way.

You can request a copy of a value object by invoking any of its PSR-7 interface methods (these methods typically have a `with` prefix). For example, a PSR-7 Response object has a `withHeader($name, $value)` method that returns a cloned value object with the new HTTP header.

    <?php

    use Psr\Http\Message\ResponseInterface as Response;
    use Psr\Http\Message\ServerRequestInterface as Request;
    use Slim\Factory\AppFactory;

    require __DIR__ . '/../vendor/autoload.php';

    $app = AppFactory::create();

    $app->get('/foo', function (Request $request, Response $response, array $args) {
        $payload = json_encode(['hello' => 'world'], JSON_PRETTY_PRINT);
        $response->getBody()->write($payload);
        return $response->withHeader('Content-Type', 'application/json');
    });

    $app->run();

The PSR-7 interface provides these methods to transform Request and Response objects:

- `withProtocolVersion($version)`
- `withHeader($name, $value)`
- `withAddedHeader($name, $value)`
- `withoutHeader($name)`
- `withBody(StreamInterface $body)`

The PSR-7 interface provides these methods to transform Request objects:

- `withMethod($method)`
- `withUri(UriInterface $uri, $preserveHost = false)`
- `withCookieParams(array $cookies)`
- `withQueryParams(array $query)`
- `withUploadedFiles(array $uploadedFiles)`
- `withParsedBody($data)`
- `withAttribute($name, $value)`
- `withoutAttribute($name)`

The PSR-7 interface provides these methods to transform Response objects:

- `withStatus($code, $reasonPhrase = '')`

Refer to the [PSR-7 documentation](https://www.php-fig.org/psr/psr-7/) for more information about these methods.

- [Concepts: Application Life Cycle](/docs/v4/concepts/life-cycle.html)
- [Concepts: Middleware](/docs/v4/concepts/middleware.html)

# Middleware - Slim Framework

# Middleware

[Edit This Page](https://github.com/slimphp/Slim-Website/tree/gh-pages/docs/v4/concepts/middleware.md)

You can run code _before_ and _after_ your Slim application to manipulate the Request and Response objects as you see fit. This is called _middleware_. Why would you want to do this? Perhaps you want to protect your app from cross-site request forgery. Maybe you want to authenticate requests before your app runs. Middleware is perfect for these scenarios.

## What is middleware?

Middleware is a layer that sits between the client request and the server response in a web application. It intercepts, processes, and potentially alters HTTP requests and responses as they pass through the application pipeline.

A middleware can handle a variety of tasks such as authentication, authorization, logging, request modification, response transformation, error handling, and more.

Each middleware performs its function and then passes control to the next middleware in the chain, enabling a modular and reusable approach to handling cross-cutting concerns in web applications.

## How does middleware work?

Different frameworks use middleware differently. Slim adds middleware as concentric layers surrounding your core application. Each new middleware layer surrounds any existing middleware layers. The concentric structure expands outwardly as additional middleware layers are added.

The last middleware layer added is the first to be executed.

When you run the Slim application, the Request object traverses the middleware structure from the outside in. They first enter the outermost middleware, then the next outermost middleware, (and so on), until they ultimately arrive at the Slim application itself. After the Slim application dispatches the appropriate route, the resultant Response object exits the Slim application and traverses the middleware structure from the inside out. Ultimately, a final Response object exits the outermost middleware, is serialized into a raw HTTP response, and is returned to the HTTP client. Here’s a diagram that illustrates the middleware process flow:

![Middleware architecture](/docs/v4/images/middleware.png)

## How do I write middleware?

Middleware is a callable that accepts two arguments: a `Request` object and a `RequestHandler` object. Each middleware **MUST** return an instance of `Psr\Http\Message\ResponseInterface`.

### Closure middleware

This example middleware is a Closure.

    <?php

    use Psr\Http\Message\ResponseInterface;
    use Psr\Http\Message\ServerRequestInterface as Request;
    use Psr\Http\Server\RequestHandlerInterface as RequestHandler;
    use Slim\Factory\AppFactory;

    require __DIR__ . '/../vendor/autoload.php';

    $app = AppFactory::create();

    $beforeMiddleware = function (Request $request, RequestHandler $handler) use ($app) {
        // Example: Check for a specific header before proceeding
        $auth = $request->getHeaderLine('Authorization');
        if (!$auth) {
            // Short-circuit and return a response immediately
            $response = $app->getResponseFactory()->createResponse();
            $response->getBody()->write('Unauthorized');

            return $response->withStatus(401);
        }

        // Proceed with the next middleware
        return $handler->handle($request);
    };

    $afterMiddleware = function (Request $request, RequestHandler $handler) {
        // Proceed with the next middleware
        $response = $handler->handle($request);

        // Modify the response after the application has processed the request
        $response = $response->withHeader('X-Added-Header', 'some-value');

        return $response;
    };

    $app->add($afterMiddleware);
    $app->add($beforeMiddleware);

    // ...

    $app->run();

### Invokable class middleware

This example middleware is an invokable class that implements the magic `__invoke()` method.

    <?php

    use Psr\Http\Message\ServerRequestInterface as Request;
    use Psr\Http\Server\RequestHandlerInterface as RequestHandler;
    use Psr\Http\Message\ResponseInterface as Response;

    class ExampleBeforeMiddleware
    {
        public function __invoke(Request $request, RequestHandler $handler): Response
        {
            // Handle the incoming request
            // ...

            // Invoke the next middleware and return response
            return $handler->handle($request);
        }
    }


    <?php

    use Psr\Http\Message\ResponseInterface as Response;
    use Psr\Http\Message\ServerRequestInterface as Request;
    use Psr\Http\Server\RequestHandlerInterface as RequestHandler;

    class ExampleAfterMiddleware
    {
        public function __invoke(Request $request, RequestHandler $handler): Response
        {
            // Invoke the next middleware and get response
            $response = $handler->handle($request);

            // Handle the outgoing response
            // ...

            return $response;
        }
    }

### PSR-15 middleware

[PSR-15](https://www.php-fig.org/psr/psr-15/#22-psrhttpservermiddlewareinterface) is a standard that defines common interfaces for HTTP server request handlers and middleware components.

Slim provides built-in support for PSR-15 middleware.

**Key Interfaces**

- `Psr\Http\Server\MiddlewareInterface`: This interface defines the **process** method that middleware must implement.
- `Psr\Http\Server\RequestHandlerInterface`: An HTTP request handler that process an HTTP request in order to produce an HTTP response.

To create a PSR-15 middleware class, you need to implement the `MiddlewareInterface`.

Below is an example of a simple PSR-15 middleware:

    <?php

    namespace App\Middleware;

    use Psr\Http\Message\ResponseInterface as Response;
    use Psr\Http\Message\ServerRequestInterface as Request;
    use Psr\Http\Server\MiddlewareInterface;
    use Psr\Http\Server\RequestHandlerInterface as RequestHandler;

    class ExampleMiddleware implements MiddlewareInterface
    {
        public function process(Request $request, RequestHandler $handler): Response
        {
            // Optional: Handle the incoming request
            // ...

            // Invoke the next middleware and get response
            $response = $handler->handle($request);

            // Optional: Handle the outgoing response
            // ...

            return $response;
        }
    }

Incoming requests can be authenticated, authorized, logged, validated, or modified.

Outgoing responses can be logged, transformed, compressed, or have additional headers added.

#### Creating a new response in a PSR-15 middleware

To create a new response, use the `Psr\Http\Message\ResponseFactoryInterface`, which provides a `createResponse()` method to create a new response object.

Here is an example of a PSR-15 middleware class that creates a new response:

    <?php

    use Psr\Http\Message\ResponseFactoryInterface;
    use Psr\Http\Message\ResponseInterface;
    use Psr\Http\Message\ServerRequestInterface;
    use Psr\Http\Server\MiddlewareInterface;
    use Psr\Http\Server\RequestHandlerInterface;

    class ExampleMiddleware implements MiddlewareInterface
    {
        private ResponseFactoryInterface $responseFactory;

        public function __construct(ResponseFactoryInterface $responseFactory)
        {
            $this->responseFactory = $responseFactory;
        }

        public function process(ServerRequestInterface $request, RequestHandlerInterface $handler): ResponseInterface
        {
            // Check some condition to determine if a new response should be created
            if (true) {
                // Create a new response using the response factory
                $response = $this->responseFactory->createResponse();
                $response->getBody()->write('New response created by middleware');

                return $response;
            }

            // Proceed with the next middleware
            return $handler->handle($request);
        }
    }

The response is created with a `200` OK status code by default. To change the HTTP status code, you can pass the desired status code as an argument to the `createResponse` method.

    $response = $this->responseFactory->createResponse(201);

Note that the response factory is a dependency that must be injected into a middleware. Make sure that the Slim  
DI container (e.g. PHP-DI) is properly configured to provide an instance of `Psr\Http\Message\ResponseFactoryInterface`.

**Example:** PHP-DI definition using the `slim\psr7` package

    use Psr\Container\ContainerInterface;
    use Slim\Psr7\Factory\ResponseFactory;
    // ...

    return [
        // ...
        ResponseFactoryInterface::class => function (ContainerInterface $container) {
            return $container->get(ResponseFactory::class);
        },
    ];

**Example:** PHP-DI definition using the `nyholm/psr7` package

    use Nyholm\Psr7\Factory\Psr17Factory;
    use Psr\Container\ContainerInterface;
    // ...

    return [
        // ...
        ResponseFactoryInterface::class => function (ContainerInterface $container) {
            return $container->get(Psr17Factory::class);
        },
    ];

### Registering middleware

To use a middleware, you need to register each middleware on the Slim **$app**, a route or a route group.

    // Add middleware to the App
    $app->add(new ExampleMiddleware());

    // Add middleware to the App using dependency injection
    $app->add(ExampleMiddleware::class);

    // Add middleware to a route
    $app->get('/', function () { ... })->add(new ExampleMiddleware());

    // Add middleware to a route group
    $app->group('/', function () { ... })->add(new ExampleMiddleware());

### Middleware execution order

Slim processes middleware in a Last In, First Out (LIFO) order. This means the last middleware added is the first one to be executed. If you add multiple middleware components, they will be executed in the reverse order of their addition.

    $app->add(new MiddlewareOne());
    $app->add(new MiddlewareTwo());
    $app->add(new MiddlewareThree());

In this case, `MiddlewareThree` will be executed first, followed by `MiddlewareTwo`, and finally `MiddlewareOne`.

### Route middleware

Route middleware is invoked _only if_ its route matches the current HTTP request method and URI. Route middleware is specified immediately after you invoke any of the Slim application’s routing methods (e.g., **get()** or **post()**). Each routing method returns an instance of **\\Slim\\Route**, and this class provides the same middleware interface as the Slim application instance. Add middleware to a Route with the Route instance’s **add()** method. This example adds the Closure middleware example above:

    <?php

    use Psr\Http\Message\ResponseInterface as Response;
    use Psr\Http\Message\ServerRequestInterface as Request;
    use Psr\Http\Server\RequestHandlerInterface as RequestHandler;
    use Slim\Factory\AppFactory;

    require __DIR__ . '/../vendor/autoload.php';

    $app = AppFactory::create();

    $middleware = function (Request $request, RequestHandler $handler) {
        $response = $handler->handle($request);
        $response->getBody()->write('World');

        return $response;
    };

    $app->get('/', function (Request $request, Response $response) {
        $response->getBody()->write('Hello ');

        return $response;
    })->add($middleware);

    $app->run();

This would output this HTTP response body:

    Hello World

### Group middleware

In addition to the overall application, and standard routes being able to accept middleware, the **group()** multi-route definition functionality, also allows individual routes internally. Route group middleware is invoked _only if_ its route matches one of the defined HTTP request methods and URIs from the group. To add middleware within the callback, and entire-group middleware to be set by chaining **add()** after the **group()** method.

Sample Application, making use of callback middleware on a group of url-handlers.

    <?php

    use Psr\Http\Message\ResponseInterface as Response;
    use Psr\Http\Message\ServerRequestInterface as Request;
    use Psr\Http\Server\RequestHandlerInterface as RequestHandler;
    use Slim\Factory\AppFactory;
    use Slim\Routing\RouteCollectorProxy;

    require __DIR__ . '/../vendor/autoload.php';

    $app = AppFactory::create();

    $app->get('/', function (Request $request, Response $response) {
        $response->getBody()->write('Hello World');
        return $response;
    });

    $app->group('/utils', function (RouteCollectorProxy $group) {
        $group->get('/date', function (Request $request, Response $response) {
            $response->getBody()->write(date('Y-m-d H:i:s'));
            return $response;
        });

        $group->get('/time', function (Request $request, Response $response) {
            $response->getBody()->write((string)time());
            return $response;
        });
    })->add(function (Request $request, RequestHandler $handler) use ($app) {
        $response = $handler->handle($request);
        $dateOrTime = (string) $response->getBody();

        $response = $app->getResponseFactory()->createResponse();
        $response->getBody()->write('It is now ' . $dateOrTime . '. Enjoy!');

        return $response;
    });

    $app->run();

When calling the **/utils/date** method, this would output a string similar to the below.

    It is now 2015-07-06 03:11:01. Enjoy!

Visiting **/utils/time** would output a string similar to the below.

    It is now 1436148762. Enjoy!

But visiting **/** _(domain-root)_, would be expected to generate the following output as no middleware has been assigned.

    Hello World

### Passing variables from middleware

The easiest way to pass attributes from middleware is to use the request’s attributes.

Setting the variable in the middleware:

    $request = $request->withAttribute('foo', 'bar');

Getting the variable in the route callback:

    $foo = $request->getAttribute('foo');

## Finding available middleware

You may find a PSR-15 Middleware class already written that will satisfy your needs. Here are a few unofficial lists to search.

- [Github PSR-15: HTTP Server Request Handlers](https://github.com/topics/psr-15)
- [middlewares/awesome-psr15-middlewares](https://github.com/middlewares/awesome-psr15-middlewares)

- [Concepts: PSR-7](/docs/v4/concepts/value-objects.html)
- [Concepts: Dependency Container](/docs/v4/concepts/di.html)

# Dependency Container - Slim Framework

# Dependency Container

[Edit This Page](https://github.com/slimphp/Slim-Website/tree/gh-pages/docs/v4/concepts/di.md)

Slim uses an optional dependency container to prepare, manage, and inject application dependencies. Slim supports containers that implement [PSR-11](http://www.php-fig.org/psr/psr-11/) like [PHP-DI](http://php-di.org/doc/frameworks/slim.html).

## Example usage with PHP-DI

You don’t _have_ to provide a dependency container. If you do, however, you must provide an instance of the container to `AppFactory` before creating an `App`.

    <?php

    use DI\Container;
    use Psr\Http\Message\ResponseInterface as Response;
    use Psr\Http\Message\ServerRequestInterface as Request;
    use Slim\Factory\AppFactory;

    require __DIR__ . '/../vendor/autoload.php';

    // Create Container using PHP-DI
    $container = new Container();

    // Set container to create App with on AppFactory
    AppFactory::setContainer($container);
    $app = AppFactory::create();

Add a service to your container:

    $container->set('myService', function () {
        $settings = [...];
        return new MyService($settings);
    });

You can fetch services from your container explicitly as well as from inside a Slim application route like this:

    /**
     * Example GET route
     *
     * @param  ServerRequestInterface $request  PSR-7 request
     * @param  ResponseInterface      $response  PSR-7 response
     * @param  array                  $args Route parameters
     *
     * @return ResponseInterface
     */
    $app->get('/foo', function (Request $request, Response $response, $args) {
        $myService = $this->get('myService');

        // ...do something with $myService...

        return $response;
    });

To test if a service exists in the container before using it, use the `has()` method, like this:

    /**
     * Example GET route
     *
     * @param  ServerRequestInterface $request  PSR-7 request
     * @param  ResponseInterface      $response  PSR-7 response
     * @param  array                  $args Route parameters
     *
     * @return ResponseInterface
     */
    $app->get('/foo', function (Request $request, Response $response, $args) {
        if ($this->has('myService')) {
            $myService = $this->get('myService');
        }
        return $response;
    });

## Configuring the application via a container

In case you want to create the `App` with dependencies already defined in your container, you can use the `AppFactory::createFromContainer()` method.

**Example**

    <?php

    use App\Factory\MyResponseFactory;
    use DI\Container;
    use Psr\Container\ContainerInterface;
    use Psr\Http\Message\ResponseFactoryInterface;
    use Slim\Factory\AppFactory;

    require_once __DIR__ . '/../vendor/autoload.php';

    // Create Container using PHP-DI
    $container = new Container();

    // Add custom response factory
    $container->set(ResponseFactoryInterface::class, function (ContainerInterface $container) {
        return new MyResponseFactory(...);
    });

    // Configure the application via container
    $app = AppFactory::createFromContainer($container);

    // ...

    $app->run();

Supported `App` dependencies are:

- Psr\\Http\\Message\\ResponseFactoryInterface
- Slim\\Interfaces\\CallableResolverInterface
- Slim\\Interfaces\\RouteCollectorInterface
- Slim\\Interfaces\\RouteResolverInterface
- Slim\\Interfaces\\MiddlewareDispatcherInterface

- [Concepts: Middleware](/docs/v4/concepts/middleware.html)
- [The Application: Overview](/docs/v4/objects/application.html)

# Application - Slim Framework

# Application

[Edit This Page](https://github.com/slimphp/Slim-Website/tree/gh-pages/docs/v4/objects/application.md)

The Application `Slim\App` is the entry point to your Slim application and is used to register the routes that link to your callbacks or controllers.

    <?php
    use Psr\Http\Message\ResponseInterface as Response;
    use Psr\Http\Message\ServerRequestInterface as Request;
    use Slim\Factory\AppFactory;

    require __DIR__ . '/../vendor/autoload.php';

    // Instantiate app
    $app = AppFactory::create();

    // Add Error Handling Middleware
    $app->addErrorMiddleware(true, false, false);

    // Add route callbacks
    $app->get('/', function (Request $request, Response $response, array $args) {
        $response->getBody()->write('Hello World');
        return $response;
    });

    // Run application
    $app->run();

## Advanced Notices and Warnings Handling

**Warnings** and **Notices** are not caught by default. If you wish your application to display an error page when they happen, you will need to implement code similar to the following `index.php`.

    <?php

    use MyApp\Handlers\HttpErrorHandler;
    use MyApp\Handlers\ShutdownHandler;
    use Slim\Exception\HttpInternalServerErrorException;
    use Slim\Factory\AppFactory;
    use Slim\Factory\ServerRequestCreatorFactory;

    require __DIR__ . '/../vendor/autoload.php';

    // Set that to your needs
    $displayErrorDetails = true;

    $app = AppFactory::create();
    $callableResolver = $app->getCallableResolver();
    $responseFactory = $app->getResponseFactory();

    $serverRequestCreator = ServerRequestCreatorFactory::create();
    $request = $serverRequestCreator->createServerRequestFromGlobals();

    $errorHandler = new HttpErrorHandler($callableResolver, $responseFactory);
    $shutdownHandler = new ShutdownHandler($request, $errorHandler, $displayErrorDetails);
    register_shutdown_function($shutdownHandler);

    // Add Routing Middleware
    $app->addRoutingMiddleware();

    // Add Error Handling Middleware
    $errorMiddleware = $app->addErrorMiddleware($displayErrorDetails, false, false);
    $errorMiddleware->setDefaultErrorHandler($errorHandler);

    $app->run();

## Advanced Custom Error Handler

    <?php
    namespace MyApp\Handlers;

    use Psr\Http\Message\ResponseInterface;
    use Slim\Exception\HttpBadRequestException;
    use Slim\Exception\HttpException;
    use Slim\Exception\HttpForbiddenException;
    use Slim\Exception\HttpMethodNotAllowedException;
    use Slim\Exception\HttpNotFoundException;
    use Slim\Exception\HttpNotImplementedException;
    use Slim\Exception\HttpUnauthorizedException;
    use Slim\Handlers\ErrorHandler;
    use Exception;
    use Throwable;

    class HttpErrorHandler extends ErrorHandler
    {
        public const BAD_REQUEST = 'BAD_REQUEST';
        public const INSUFFICIENT_PRIVILEGES = 'INSUFFICIENT_PRIVILEGES';
        public const NOT_ALLOWED = 'NOT_ALLOWED';
        public const NOT_IMPLEMENTED = 'NOT_IMPLEMENTED';
        public const RESOURCE_NOT_FOUND = 'RESOURCE_NOT_FOUND';
        public const SERVER_ERROR = 'SERVER_ERROR';
        public const UNAUTHENTICATED = 'UNAUTHENTICATED';

        protected function respond(): ResponseInterface
        {
            $exception = $this->exception;
            $statusCode = 500;
            $type = self::SERVER_ERROR;
            $description = 'An internal error has occurred while processing your request.';

            if ($exception instanceof HttpException) {
                $statusCode = $exception->getCode();
                $description = $exception->getMessage();

                if ($exception instanceof HttpNotFoundException) {
                    $type = self::RESOURCE_NOT_FOUND;
                } elseif ($exception instanceof HttpMethodNotAllowedException) {
                    $type = self::NOT_ALLOWED;
                } elseif ($exception instanceof HttpUnauthorizedException) {
                    $type = self::UNAUTHENTICATED;
                } elseif ($exception instanceof HttpForbiddenException) {
                    $type = self::UNAUTHENTICATED;
                } elseif ($exception instanceof HttpBadRequestException) {
                    $type = self::BAD_REQUEST;
                } elseif ($exception instanceof HttpNotImplementedException) {
                    $type = self::NOT_IMPLEMENTED;
                }
            }

            if (
                !($exception instanceof HttpException)
                && ($exception instanceof Exception || $exception instanceof Throwable)
                && $this->displayErrorDetails
            ) {
                $description = $exception->getMessage();
            }

            $error = [
                'statusCode' => $statusCode,
                'error' => [
                    'type' => $type,
                    'description' => $description,
                ],
            ];

            $payload = json_encode($error, JSON_PRETTY_PRINT);

            $response = $this->responseFactory->createResponse($statusCode);
            $response->getBody()->write($payload);

            return $response;
        }
    }

## Advanced Shutdown Handler

    <?php

    namespace MyApp\Handlers;

    use MyApp\Handlers\HttpErrorHandler;
    use Psr\Http\Message\ServerRequestInterface as Request;
    use Slim\Exception\HttpInternalServerErrorException;
    use Slim\ResponseEmitter;

    class ShutdownHandler
    {
        /**
         * @var Request
         */
        private $request;

        /**
         * @var HttpErrorHandler
         */
        private $errorHandler;

        /**
         * @var bool
         */
        private $displayErrorDetails;

        /**
         * ShutdownHandler constructor.
         *
         * @param Request           $request
         * @param HttpErrorHandler  $errorHandler
         * @param bool              $displayErrorDetails
         */
        public function __construct(Request $request, HttpErrorHandler $errorHandler, bool $displayErrorDetails) {
            $this->request = $request;
            $this->errorHandler = $errorHandler;
            $this->displayErrorDetails = $displayErrorDetails;
        }

        public function __invoke()
        {
            $error = error_get_last();
            if ($error) {
                $errorFile = $error['file'];
                $errorLine = $error['line'];
                $errorMessage = $error['message'];
                $errorType = $error['type'];
                $message = 'An error while processing your request. Please try again later.';

                if ($this->displayErrorDetails) {
                    switch ($errorType) {
                        case E_USER_ERROR:
                            $message = "FATAL ERROR: {$errorMessage}. ";
                            $message .= " on line {$errorLine} in file {$errorFile}.";
                            break;

                        case E_USER_WARNING:
                            $message = "WARNING: {$errorMessage}";
                            break;

                        case E_USER_NOTICE:
                            $message = "NOTICE: {$errorMessage}";
                            break;

                        default:
                            $message = "ERROR: {$errorMessage}";
                            $message .= " on line {$errorLine} in file {$errorFile}.";
                            break;
                    }
                }

                $exception = new HttpInternalServerErrorException($this->request, $message);
                $response = $this->errorHandler->__invoke($this->request, $exception, $this->displayErrorDetails, false, false);

                if (ob_get_length()) {
                  ob_clean();
                }

                $responseEmitter = new ResponseEmitter();
                $responseEmitter->emit($response);
            }
        }
    }

- [Concepts: Dependency Container](/docs/v4/concepts/di.html)
- [The Request: Overview](/docs/v4/objects/request.html)

# Request - Slim Framework

# Request

[Edit This Page](https://github.com/slimphp/Slim-Website/tree/gh-pages/docs/v4/objects/request.md)

Your Slim app’s routes and middleware are given a PSR-7 request object that represents the current HTTP request received by your web server. The request object implements the [PSR-7 ServerRequestInterface](https://www.php-fig.org/psr/psr-7/#321-psrhttpmessageserverrequestinterface) with which you can inspect and manipulate the HTTP request method, headers, and body.

## How to get the Request object

The PSR-7 request object is injected into your Slim application routes as the first argument to the route callback like this:

    <?php

    use Psr\Http\Message\ResponseInterface as Response;
    use Psr\Http\Message\ServerRequestInterface as Request;
    use Slim\Factory\AppFactory;

    require __DIR__ . '/../vendor/autoload.php';

    $app = AppFactory::create();

    $app->get('/hello', function (Request $request, Response $response) {
        $response->getBody()->write('Hello World');
        return $response;
    });

    $app->run();

The PSR-7 request object is injected into your Slim application _middleware_ as the first argument of the middleware callable like this:

Inject PSR-7 request into application middleware:

    <?php

    use Psr\Http\Message\ServerRequestInterface as Request;
    use Psr\Http\Server\RequestHandlerInterface as RequestHandler;
    use Slim\Factory\AppFactory;

    require __DIR__ . '/../vendor/autoload.php';

    $app = AppFactory::create();

    $app->add(function (Request $request, RequestHandler $handler) {
       return $handler->handle($request);
    });

    // ...define app routes...

    $app->run();

## The Request HTTP-Method

Every HTTP request has a method that is typically one of:

- GET
- POST
- PUT
- DELETE
- HEAD
- PATCH
- OPTIONS

You can inspect the HTTP request’s method with the Request object method appropriately named `getMethod()`.

    $method = $request->getMethod();

It is possible to fake or _override_ the HTTP request method. This is useful if, for example, you need to mimic a `PUT` request using a traditional web browser that only supports `GET` or `POST` requests.

**Note:** To enable request method overriding the [Method Overriding Middleware](/docs/v4/middleware/method-overriding.html) must be injected into your application.

There are two ways to override the HTTP request method. You can include a `METHOD` parameter in a `POST` request’s body. The HTTP request must use the `application/x-www-form-urlencoded` content type.

Override HTTP method with \_METHOD parameter:

    POST /path HTTP/1.1
    Host: example.com
    Content-type: application/x-www-form-urlencoded
    Content-length: 22

    data=value&_METHOD=PUT

You can also override the HTTP request method with a custom `X-Http-Method-Override` HTTP request header. This works with any HTTP request content type.

    POST /path HTTP/1.1
    Host: example.com
    Content-type: application/json
    Content-length: 16
    X-Http-Method-Override: PUT

    {"data":"value"}

### Server Parameters

To fetch data related to the incoming request environment, you will need to use `getServerParams()`. For example, to get a single Server Parameter:

    $params = $request->getServerParams();
    $authorization = $params['HTTP_AUTHORIZATION'] ?? null;

### POST Parameters

If the request method is `POST` and the `Content-Type` is either `application/x-www-form-urlencoded` or `multipart/form-data`, you can retrieve all `POST` parameters as follows:

    // Get all POST parameters
    $params = (array)$request->getParsedBody();

    // Get a single POST parameter
    $foo = $params['foo'];

## The Request URI

Every HTTP request has a URI that identifies the requested application resource. The HTTP request URI has several parts:

- Scheme (e.g. `http` or `https`)
- Host (e.g. `example.com`)
- Port (e.g. `80` or `443`)
- Path (e.g. `/users/1`)
- Query string (e.g. `sort=created&dir=asc`)

You can fetch the PSR-7 Request object’s [URI object](https://www.php-fig.org/psr/psr-7/#35-psrhttpmessageuriinterface) with its `getUri()` method:

    $uri = $request->getUri();

The PSR-7 Request object’s URI is itself an object that provides the following methods to inspect the HTTP request’s URL parts:

- getScheme()
- getAuthority()
- getUserInfo()
- getHost()
- getPort()
- getPath()
- getQuery() (returns the full query string, e.g. `a=1&b=2`)
- getFragment()

You can get the query parameters as an associative array on the Request object using `getQueryParams()`.

### Query String Parameters

The `getQueryParams()` method retrieves all query parameters from the URI of an HTTP request as an associative array.

If there are no query parameters, it returns an empty array.

Internally, the method uses [parse_str](https://www.php.net/manual/en/function.parse-str.php) to parse the query string into an array.

**Usage**

    // URL: https://example.com/search?key1=value1&key2=value2
    $queryParams = $request->getQueryParams();


    Array
    (
        [key1] => value1
        [key2] => value2
    )

To read a single value from the query parameters array, you can use the parameter’s name as the key.

    // Output: value1
    $key1 = $queryParams['key1'] ?? null;

    // Output: value2
    $key2 = $queryParams['key2'] ?? null;

    // Output: null
    $key3 = $queryParams['key3'] ?? null;

**Note:** `?? null` ensures that if the query parameter does not exist, `null` is returned instead of causing a warning.

## The Request Headers

Every HTTP request has headers. These are metadata that describe the HTTP request but are not visible in the request’s body. Slim’s PSR-7 Request object provides several methods to inspect its headers.

### Get All Headers

You can fetch all HTTP request headers as an associative array with the PSR-7 Request object’s `getHeaders()` method. The resultant associative array’s keys are the header names and its values are themselves a numeric array of string values for their respective header name.

    $headers = $request->getHeaders();
    foreach ($headers as $name => $values) {
        echo $name . ": " . implode(", ", $values);
    }

Figure 5: Fetch and iterate all HTTP request headers as an associative array.

### Get One Header

You can get a single header’s value(s) with the PSR-7 Request object’s `getHeader($name)` method. This returns an array of values for the given header name. Remember, _a single HTTP header may have more than one value!_

Get values for a specific HTTP header:

    $headerValueArray = $request->getHeader('Accept');

You may also fetch a comma-separated string with all values for a given header with the PSR-7 Request object’s `getHeaderLine($name)` method. Unlike the `getHeader($name)` method, this method returns a comma-separated string.

Get single header’s values as comma-separated string:

    $headerValueString = $request->getHeaderLine('Accept');

### Detect Header

You can test for the presence of a header with the PSR-7 Request object’s `hasHeader($name)` method.

    if ($request->hasHeader('Accept')) {
        // Do something
    }

### Detect XHR requests

You can detect XHR requests by checking if the header `X-Requested-With` is `XMLHttpRequest` using the Request’s `getHeaderLine()` method.

Example XHR request:

    POST /path HTTP/1.1
    Host: example.com
    Content-type: application/x-www-form-urlencoded
    Content-length: 7
    X-Requested-With: XMLHttpRequest

    foo=bar


    if ($request->getHeaderLine('X-Requested-With') === 'XMLHttpRequest') {
        // Do something
    }

### Content Type

You can fetch the HTTP request content type with the Request object’s `getHeaderLine()` method.

    $contentType = $request->getHeaderLine('Content-Type');

### Content Length

You can fetch the HTTP request content length with the Request object’s `getHeaderLine()` method.

    $length = $request->getHeaderLine('Content-Length');

## The Request Body

Every HTTP request has a body. If you are building a Slim application that consumes JSON or XML data, you can use the PSR-7 Request object’s `getParsedBody()` method to parse the HTTP request body into a native PHP format. Note that body parsing differs from one PSR-7 implementation to another.

You may need to implement middleware in order to parse the incoming input depending on the PSR-7 implementation you have installed. Here is an example for parsing incoming `JSON` input:

    <?php

    use Psr\Http\Message\ResponseInterface as Response;
    use Psr\Http\Message\ServerRequestInterface as Request;
    use Psr\Http\Server\MiddlewareInterface;
    use Psr\Http\Server\RequestHandlerInterface as RequestHandler;

    class JsonBodyParserMiddleware implements MiddlewareInterface
    {
        public function process(Request $request, RequestHandler $handler): Response
        {
            $contentType = $request->getHeaderLine('Content-Type');

            if (strstr($contentType, 'application/json')) {
                $contents = json_decode(file_get_contents('php://input'), true);
                if (json_last_error() === JSON_ERROR_NONE) {
                    $request = $request->withParsedBody($contents);
                }
            }

            return $handler->handle($request);
        }
    }

Parse HTTP request body into native PHP format:

    $parsedBody = $request->getParsedBody();

Technically speaking, the PSR-7 Request object represents the HTTP request body as an instance of `Psr\Http\Message\StreamInterface`. You can get the HTTP request body `StreamInterface` instance with the PSR-7 Request object’s `getBody()` method. The `getBody()` method is preferable if the incoming HTTP request size is unknown or too large for available memory.

Get HTTP request body:

    $body = $request->getBody();

The resultant `Psr\Http\Message\StreamInterface` instance provides the following methods to read and iterate its underlying PHP `resource`.

- getSize()
- tell()
- eof()
- isSeekable()
- seek()
- rewind()
- isWritable()
- write($string)
- isReadable()
- read($length)
- getContents()
- getMetadata($key = null)

## Uploaded Files

The file uploads in `$_FILES` are available from the Request object’s `getUploadedFiles()` method. This returns an array keyed by the name of the `input` element.

Get uploaded files:

    $files = $request->getUploadedFiles();

Each object in the `$files` array is an instance of `Psr\Http\Message\UploadedFileInterface` and supports the following methods:

- getStream()
- moveTo($targetPath)
- getSize()
- getError()
- getClientFilename()
- getClientMediaType()

See the [cookbook](/docs/v4/cookbook/uploading-files.html) on how to upload files using a POST form.

## Attributes

With PSR-7 it is possible to inject objects/values into the request object for further processing. In your applications middleware often need to pass along information to your route closure and the way to do it is to add it to the request object via an attribute.

Example, setting a value on your request object.

    use Psr\Http\Message\ServerRequestInterface as Request;
    use Psr\Http\Server\RequestHandlerInterface as RequestHandler;

    $app->add(function (Request $request, RequestHandler $handler) {
        // Add the session storage to your request as [READ-ONLY]
        $request = $request->withAttribute('session', $_SESSION);

        return $handler->handle($request);
    });

Example, how to retrieve the value.

    use Psr\Http\Message\ResponseInterface as Response;
    use Psr\Http\Message\ServerRequestInterface as Request;

    $app->get('/test', function (Request $request, Response $response) {
        // Get the session from the request
        $session = $request->getAttribute('session');

        $response->getBody()->write('Yay, ' . $session['name']);

        return $response;
    });

The request object also has bulk functions as well. `$request->getAttributes()` and `$request->withAttributes()`

- [The Application: Overview](/docs/v4/objects/application.html)
- [The Response: Overview](/docs/v4/objects/response.html)

# Response - Slim Framework

# Response

[Edit This Page](https://github.com/slimphp/Slim-Website/tree/gh-pages/docs/v4/objects/response.md)

Your Slim app’s routes and middleware are given a PSR-7 response object that represents the current HTTP response to be returned to the client. The response object implements the [PSR-7 ResponseInterface](https://www.php-fig.org/psr/psr-7/#33-psrhttpmessageresponseinterface) with which you can inspect and manipulate the HTTP response status, headers, and body.

## How to get the Response object

The PSR-7 response object is injected into your Slim application routes as the second argument to the route callback like this:

    <?php

    use Psr\Http\Message\ResponseInterface as Response;
    use Psr\Http\Message\ServerRequestInterface as Request;
    use Slim\Factory\AppFactory;

    require __DIR__ . '/../vendor/autoload.php';

    $app = AppFactory::create();

    $app->get('/hello', function (Request $request, Response $response) {
        $response->getBody()->write('Hello World');
        return $response;
    });

    $app->run();

Figure 1: Inject PSR-7 response into application route callback.

## The Response Status

Every HTTP response has a numeric [status code](http://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html). The status code identifies the _type_ of HTTP response to be returned to the client. The PSR-7 Response object’s default status code is `200` (OK). You can get the PSR-7 Response object’s status code with the `getStatusCode()` method like this.

    $status = $response->getStatusCode();

Figure 3: Get response status code.

You can copy a PSR-7 Response object and assign a new status code like this:

    $newResponse = $response->withStatus(302);

Figure 4: Create response with new status code.

## The Response Headers

Every HTTP response has headers. These are metadata that describe the HTTP response but are not visible in the response’s body. The PSR-7 Response object provides several methods to inspect and manipulate its headers.

### Get All Headers

You can fetch all HTTP response headers as an associative array with the PSR-7 Response object’s `getHeaders()` method. The resultant associative array’s keys are the header names and its values are themselves a numeric array of string values for their respective header name.

    $headers = $response->getHeaders();
    foreach ($headers as $name => $values) {
        echo $name . ": " . implode(", ", $values);
    }

Figure 5: Fetch and iterate all HTTP response headers as an associative array.

### Get One Header

You can get a single header’s value(s) with the PSR-7 Response object’s `getHeader($name)` method. This returns an array of values for the given header name. Remember, _a single HTTP header may have more than one value!_

    $headerValueArray = $response->getHeader('Vary');

Figure 6: Get values for a specific HTTP header.

You may also fetch a comma-separated string with all values for a given header with the PSR-7 Response object’s `getHeaderLine($name)` method. Unlike the `getHeader($name)` method, this method returns a comma-separated string.

    $headerValueString = $response->getHeaderLine('Vary');

Figure 7: Get single header's values as comma-separated string.

### Detect Header

You can test for the presence of a header with the PSR-7 Response object’s `hasHeader($name)` method.

    if ($response->hasHeader('Vary')) {
        // Do something
    }

Figure 8: Detect presence of a specific HTTP header.

### Set Header

You can set a header value with the PSR-7 Response object’s `withHeader($name, $value)` method.

    $newResponse = $oldResponse->withHeader('Content-type', 'application/json');

Figure 9: Set HTTP header

**Reminder**

The Response object is immutable. This method returns a _copy_ of the Response object that has the new header value. **This method is destructive**, and it _replaces_ existing header values already associated with the same header name.

### Append Header

You can append a header value with the PSR-7 Response object’s `withAddedHeader($name, $value)` method.

    $newResponse = $oldResponse->withAddedHeader('Allow', 'PUT');

Figure 10: Append HTTP header

**Reminder**

Unlike the `withHeader()` method, this method _appends_ the new value to the set of values that already exist for the same header name. The Response object is immutable. This method returns a _copy_ of the Response object that has the appended header value.

### Remove Header

You can remove a header with the Response object’s `withoutHeader($name)` method.

    $newResponse = $oldResponse->withoutHeader('Allow');

Figure 11: Remove HTTP header

**Reminder**

The Response object is immutable. This method returns a _copy_ of the Response object without the specified header.

## The Response Body

An HTTP response typically has a body.

Just like the PSR-7 Request object, the PSR-7 Response object implements the body as an instance of `Psr\Http\Message\StreamInterface`. You can get the HTTP response body `StreamInterface` instance with the PSR-7 Response object’s `getBody()` method. The `getBody()` method is preferable if the outgoing HTTP response length is unknown or too large for available memory.

    $body = $response->getBody();

Figure 12: Get HTTP response body

The resultant `Psr\Http\Message\StreamInterface` instance provides the following methods to read from, iterate, and write to its underlying PHP `resource`.

- getSize()
- tell()
- eof()
- isSeekable()
- seek()
- rewind()
- isWritable()
- write($string)
- isReadable()
- read($length)
- getContents()
- getMetadata($key = null)

Most often, you’ll need to write to the PSR-7 Response object. You can write content to the `StreamInterface` instance with its `write()` method like this:

    $body = $response->getBody();
    $body->write('Hello');

Figure 13: Write content to the HTTP response body

You can also _replace_ the PSR-7 Response object’s body with an entirely new `StreamInterface` instance. This is particularly useful when you want to pipe content from a remote destination (e.g. the filesystem or a remote API) into the HTTP response. You can replace the PSR-7 Response object’s body with its `withBody(StreamInterface $body)` method. Its argument **MUST** be an instance of `Psr\Http\Message\StreamInterface`.

    use GuzzleHttp\Psr7\LazyOpenStream;

    $newStream = new LazyOpenStream('/path/to/file', 'r');
    $newResponse = $oldResponse->withBody($newStream);

Figure 14: Replace the HTTP response body

**Reminder**

The Response object is immutable. This method returns a _copy_ of the Response object that contains the new body.

## Returning JSON

In it’s simplest form, JSON data can be returned with a default 200 HTTP status code.

    $data = array('name' => 'Bob', 'age' => 40);
    $payload = json_encode($data);

    $response->getBody()->write($payload);
    return $response
              ->withHeader('Content-Type', 'application/json');

Figure 15: Returning JSON with a 200 HTTP status code.

We can also return JSON data with a custom HTTP status code.

    $data = array('name' => 'Rob', 'age' => 40);
    $payload = json_encode($data);

    $response->getBody()->write($payload);
    return $response
              ->withHeader('Content-Type', 'application/json')
              ->withStatus(201);

Figure 16: Returning JSON with a 201 HTTP status code.

**Reminder**

The Response object is immutable. This method returns a _copy_ of the Response object that has a new Content-Type header. **This method is destructive**, and it _replaces_ the existing Content-Type header.

## Returning a Redirect

You can redirect the HTTP client by using the `Location` header.

    return $response
      ->withHeader('Location', 'https://www.example.com')
      ->withStatus(302);

Figure 17: Returning a redirect to https://www.example.com

- [The Request: Overview](/docs/v4/objects/request.html)
- [Routing: Overview](/docs/v4/objects/routing.html)

# Routing - Slim Framework

# Routing

[Edit This Page](https://github.com/slimphp/Slim-Website/tree/gh-pages/docs/v4/objects/routing.md)

The Slim Framework’s router is built on top of the [Fast Route](https://github.com/nikic/FastRoute) component, and it is remarkably fast and stable. While we are using this component to do all our routing, the app’s core has been entirely decoupled from it and interfaces have been put in place to pave the way for using other routing libraries.

## How to create routes

You can define application routes using proxy methods on the `Slim\App` instance. The Slim Framework provides methods for the most popular HTTP methods.

### GET Route

You can add a route that handles only `GET` HTTP requests with the Slim application’s `get()` method. It accepts two arguments:

1.  The route pattern (with optional named placeholders)
2.  The route callback

    $app->get('/books/{id}', function ($request, $response, array $args) {
    // Show book identified by $args['id']
    });

### POST Route

You can add a route that handles only `POST` HTTP requests with the Slim application’s `post()` method. It accepts two arguments:

1.  The route pattern (with optional named placeholders)
2.  The route callback

    $app->post('/books', function ($request, $response, array $args) {
    // Create new book
    });

### PUT Route

You can add a route that handles only `PUT` HTTP requests with the Slim application’s `put()` method. It accepts two arguments:

1.  The route pattern (with optional named placeholders)
2.  The route callback

    $app->put('/books/{id}', function ($request, $response, array $args) {
    // Update book identified by $args['id']
    });

### DELETE Route

You can add a route that handles only `DELETE` HTTP requests with the Slim application’s `delete()` method. It accepts two arguments:

1.  The route pattern (with optional named placeholders)
2.  The route callback

    $app->delete('/books/{id}', function ($request, $response, array $args) {
    // Delete book identified by $args['id']
    });

### OPTIONS Route

You can add a route that handles only `OPTIONS` HTTP requests with the Slim application’s `options()` method. It accepts two arguments:

1.  The route pattern (with optional named placeholders)
2.  The route callback

    $app->options('/books/{id}', function ($request, $response, array $args) {
    // Return response headers
    });

### PATCH Route

You can add a route that handles only `PATCH` HTTP requests with the Slim application’s `patch()` method. It accepts two arguments:

1.  The route pattern (with optional named placeholders)
2.  The route callback

    $app->patch('/books/{id}', function ($request, $response, array $args) {
    // Apply changes to book identified by $args['id']
    });

### Any Route

You can add a route that handles all HTTP request methods with the Slim application’s `any()` method. It accepts two arguments:

1.  The route pattern (with optional named placeholders)
2.  The route callback

    $app->any('/books/[{id}]', function ($request, $response, array $args) {
    // Apply changes to books or book identified by $args['id'] if specified.
    // To check which method is used: $request->getMethod();
    });

Note that the second parameter is a callback. You could specify a Class which implements the `__invoke()` method instead of a Closure. You can then do the mapping somewhere else:

    $app->any('/user', 'MyRestfulController');

### Custom Route

You can add a route that handles multiple HTTP request methods with the Slim application’s `map()` method. It accepts three arguments:

1.  Array of HTTP methods
2.  The route pattern (with optional named placeholders)
3.  The route callback

    $app->map(['GET', 'POST'], '/books', function ($request, $response, array $args) {
    // Create new book or list all books
    });

## Route callbacks

Each routing method described above accepts a callback routine as its final argument. This argument can be any PHP callable, and by default it accepts three arguments.

- `Request` The first argument is a `Psr\Http\Message\ServerRequestInterface` object that represents the current HTTP request.
- `Response` The second argument is a `Psr\Http\Message\ResponseInterface` object that represents the current HTTP response.
- `Arguments` The third argument is an associative array that contains values for the current route’s named placeholders.

### Writing content to the response

There are two ways to write content to the HTTP response:

1.  Using the `$response->getBody()->write('my content');` method of the Response object.
2.  Simply `echo()` content from the route callback. This content will be appended or prepended to the current HTTP response object if you add the [Output Buffering Middleware](/docs/v4/middleware/output-buffering.html).

Please note that since Slim 4, you must return a `Psr\Http\Message\ResponseInterface` object.

### Closure binding

If you use a [dependency container](/docs/v4/concepts/di.html) and a `Closure` instance as the route callback, the closure’s state is bound to the `Container` instance. This means you will have access to the DI container instance _inside_ of the Closure via the `$this` keyword:

    $app->get('/hello/{name}', function ($request, $response, array $args) {
        // Use app HTTP cookie service
        $this->get('cookies')->set('name', [
            'value' => $args['name'],
            'expires' => '7 days'
        ]);
    });

**Heads Up!**

Slim does not support `static` closures.

## Redirect helper

You can add a route that redirects `GET` HTTP requests to a different URL with the Slim application’s `redirect()` method. It accepts three arguments:

1.  The route pattern (with optional named placeholders) to redirect `from`
2.  The location to redirect `to`, which may be a `string` or a [Psr\\Http\\Message\\UriInterface](https://www.php-fig.org/psr/psr-7/#35-psrhttpmessageuriinterface)
3.  The HTTP status code to use (optional; `302` if unset)

    $app->redirect('/books', '/library', 301);

`redirect()` routes respond with the status code requested and a `Location` header set to the second argument.

## Route strategies

The route callback signature is determined by a route strategy. By default, Slim expects route callbacks to accept the request, response, and an array of route placeholder arguments. This is called the RequestResponse strategy. However, you can change the expected route callback signature by simply using a different strategy. As an example, Slim provides an alternative strategy called `RequestResponseArgs` that accepts request and response, plus each route placeholder as a separate argument.

Here is an example of using this alternative strategy:

    <?php
    use Slim\Factory\AppFactory;
    use Slim\Handlers\Strategies\RequestResponseArgs;

    require __DIR__ . '/../vendor/autoload.php';

    $app = AppFactory::create();

    /**
     * Changing the default invocation strategy on the RouteCollector component
     * will change it for every route being defined after this change being applied
     */
    $routeCollector = $app->getRouteCollector();
    $routeCollector->setDefaultInvocationStrategy(new RequestResponseArgs());

    $app->get('/hello/{name}', function ($request, $response, $name) {
        $response->getBody()->write($name);

        return $response;
    });

Alternatively you can set a different invocation strategy on a per route basis:

    <?php
    use Slim\Factory\AppFactory;
    use Slim\Handlers\Strategies\RequestResponseArgs;

    require __DIR__ . '/../vendor/autoload.php';

    $app = AppFactory::create();
    $routeCollector = $app->getRouteCollector();

    $route = $app->get('/hello/{name}', function ($request, $response, $name) {
        $response->getBody()->write($name);

        return $response;
    });
    $route->setInvocationStrategy(new RequestResponseArgs());

You can provide your own route strategy by implementing the `Slim\Interfaces\InvocationStrategyInterface`.

## Route placeholders

Each routing method described above accepts a URL pattern that is matched against the current HTTP request URI. Route patterns may use named placeholders to dynamically match HTTP request URI segments.

### Format

A route pattern placeholder starts with a `{`, followed by the placeholder name, ending with a `}`. This is an example placeholder named `name`:

    use Psr\Http\Message\ResponseInterface;
    use Psr\Http\Message\ServerRequestInterface;
    // ...

    $app->get('/hello/{name}', function (ServerRequestInterface $request, ResponseInterface $response, array $args) {
        $name = $args['name'];
        $response->getBody()->write("Hello, $name");

        return $response;
    });

### Optional segments

To make a section optional, simply wrap in square brackets:

    $app->get('/users[/{id}]', function ($request, $response, array $args) {
        // responds to both `/users` and `/users/123`
        // but not to `/users/`

        return $response;
    });

Multiple optional parameters are supported by nesting:

    $app->get('/news[/{year}[/{month}]]', function ($request, $response, array $args) {
        // responds to `/news`, `/news/2016` and `/news/2016/03`
        // ...

        return $response;
    });

For “Unlimited” optional parameters, you can do this:

    $app->get('/news[/{params:.*}]', function ($request, $response, array $args) {
        // $params is an array of all the optional segments
        $params = explode('/', $args['params']);
        // ...

        return $response;
    });

In this example, a URI of `/news/2016/03/20` would result in the `$params` array containing three elements: `['2016', '03', '20']`.

### Regular expression matching

By default the placeholders are written inside `{}` and can accept any values. However, placeholders can also require the HTTP request URI to match a particular regular expression. If the current HTTP request URI does not match a placeholder regular expression, the route is not invoked. This is an example placeholder named `id` that requires one or more digits.

    $app->get('/users/{id:[0-9]+}', function ($request, $response, array $args) {
        // Find user identified by $args['id']
        // ...

        return $response;
    });

## Route names

Application routes can be assigned a name. This is useful if you want to programmatically generate a URL to a specific route with the RouteParser’s `urlFor()` method. Each routing method described above returns a `Slim\Route` object, and this object exposes a `setName()` method.

    $app->get('/hello/{name}', function ($request, $response, array $args) {
        $response->getBody()->write("Hello, " . $args['name']);
        return $response;
    })->setName('hello');

You can generate a URL for this named route with the application RouteParser’s `urlFor()` method.

    $routeParser = $app->getRouteCollector()->getRouteParser();
    echo $routeParser->urlFor('hello', ['name' => 'Josh'], ['example' => 'name']);

    // Outputs "/hello/Josh?example=name"

The RouteParser’s `urlFor()` method accepts three arguments:

- `$routeName` The route name. A route’s name can be set via `$route->setName('name')`. Route mapping methods return an instance of `Route` so you can set the name directly after mapping the route. e.g.: `$app->get('/', function () {...})->setName('name')`
- `$data` Associative array of route pattern placeholders and replacement values.
- `$queryParams` Associative array of query parameters to be appended to the generated url.

## Route groups

To help organize routes into logical groups, the `Slim\App` also provides a `group()` method. Each group’s route pattern is prepended to the routes or groups contained within it, and any placeholder arguments in the group pattern are ultimately made available to the nested routes:

    use Slim\Routing\RouteCollectorProxy;
    // ...

    $app->group('/users/{id:[0-9]+}', function (RouteCollectorProxy $group) {
        $group->map(['GET', 'DELETE', 'PATCH', 'PUT'], '', function ($request, $response, array $args) {
            // Find, delete, patch or replace user identified by $args['id']
            // ...

            return $response;
        })->setName('user');

        $group->get('/reset-password', function ($request, $response, array $args) {
            // Route for /users/{id:[0-9]+}/reset-password
            // Reset the password for user identified by $args['id']
            // ...

            return $response;
        })->setName('user-password-reset');
    });

The group pattern can be empty, enabling the logical grouping of routes that do not share a common pattern.

    use Slim\Routing\RouteCollectorProxy;
    // ...

    $app->group('', function (RouteCollectorProxy $group) {
        $group->get('/billing', function ($request, $response, array $args) {
            // Route for /billing
            return $response;
        });

        $group->get('/invoice/{id:[0-9]+}', function ($request, $response, array $args) {
            // Route for /invoice/{id:[0-9]+}
            return $response;
        });
    })->add(new GroupMiddleware());

Note inside the group closure, Slim binds the closure to the container instance.

- inside route closure, `$this` is bound to the instance of `Psr\Container\ContainerInterface`

## Route middleware

You can also attach middleware to any route or route group.

    use Slim\Routing\RouteCollectorProxy;
    // ...

    $app->group('/foo', function (RouteCollectorProxy $group) {
        $group->get('/bar', function ($request, $response, array $args) {
            // ...
            return $response;
        })->add(new RouteMiddleware());
    })->add(new GroupMiddleware());

## Route expressions caching

It’s possible to enable router cache via `RouteCollector::setCacheFile()`. See examples below:

    <?php
    use Slim\Factory\AppFactory;

    require __DIR__ . '/../vendor/autoload.php';

    $app = AppFactory::create();

    /**
     * To generate the route cache data, you need to set the file to one that does not exist in a writable directory.
     * After the file is generated on first run, only read permissions for the file are required.
     *
     * You may need to generate this file in a development environment and committing it to your project before deploying
     * if you don't have write permissions for the directory where the cache file resides on the server it is being deployed to
     */
    $routeCollector = $app->getRouteCollector();
    $routeCollector->setCacheFile('/path/to/cache.file');

## Container Resolution

You are not limited to defining a function for your routes. In Slim there are a few different ways to define your route action functions.

In addition to a function, you may use:

- container_key:method
- Class:method
- Class implementing `__invoke()` method
- container_key

This functionality is enabled by Slim’s Callable Resolver Class. It translates a string entry into a function call. Example:

    $app->get('/', '\HomeController:home');

Alternatively, you can take advantage of PHP’s `::class` operator which works well with IDE lookup systems and produces the same result:

    $app->get('/', \HomeController::class . ':home');

You can also pass an array, the first element of which will contain the name of the class, and the second will contain the name of the method being called:

    $app->get('/', [\HomeController::class, 'home']);

In this code above we are defining a `/` route and telling Slim to execute the `home()` method on the `HomeController` class.

Slim first looks for an entry of `HomeController` in the container, if it’s found it will use that instance, otherwise it will call its constructor with the container as the first argument. Once an instance of the class is created it will then call the specified method using whatever Strategy you have defined.

### Registering a controller with the container

Create a controller with the `home` action method. The constructor should accept the dependencies that are required. For example:

    <?php

    use Psr\Http\Message\ResponseInterface;
    use Psr\Http\Message\ServerRequestInterface;
    use Slim\Views\Twig;

    class HomeController
    {
        private $view;

        public function __construct(Twig $view)
        {
            $this->view = $view;
        }

        public function home(ServerRequestInterface $request, ResponseInterface $response, array $args): ResponseInterface
        {
          // your code here
          // use $this->view to render the HTML
          // ...

          return $response;
        }
    }

Create a factory in the container that instantiates the controller with the dependencies:

    use Psr\Container\ContainerInterface;
    // ...

    $container = $app->getContainer();

    $container->set(\HomeController::class, function (ContainerInterface $container) {
        // retrieve the 'view' from the container
        $view = $container->get('view');

        return new HomeController($view);
    });

This allows you to leverage the container for dependency injection and so you can inject specific dependencies into the controller.

### Allow Slim to instantiate the controller

Alternatively, if the class does not have an entry in the container, then Slim will pass the container’s instance to the constructor. You can construct controllers with many actions instead of an invokable class which only handles one action.

    <?php

    use Psr\Container\ContainerInterface;
    use Psr\Http\Message\ResponseInterface;
    use Psr\Http\Message\ServerRequestInterface;

    class HomeController
    {
       private $container;

       // constructor receives container instance
       public function __construct(ContainerInterface $container)
       {
           $this->container = $container;
       }

       public function home(ServerRequestInterface $request, ResponseInterface $response, array $args): ResponseInterface
       {
            // your code to access items in the container... $this->container->get('');

            return $response;
       }

       public function contact(ServerRequestInterface $request, ResponseInterface $response, array $args): ResponseInterface
       {
            // your code to access items in the container... $this->container->get('');

            return $response;
       }
    }

You can use your controller methods like so.

    $app->get('/', \HomeController::class . ':home');
    $app->get('/contact', \HomeController::class . ':contact');

### Using an invokable class

You do not have to specify a method in your route callable and can just set it to be an invokable class such as:

    <?php

    use Psr\Container\ContainerInterface;
    use Psr\Http\Message\ResponseInterface;
    use Psr\Http\Message\ServerRequestInterface;

    class HomeAction
    {
       private $container;

       public function __construct(ContainerInterface $container)
       {
           $this->container = $container;
       }

       public function __invoke(ServerRequestInterface $request, ResponseInterface $response, array $args): ResponseInterface
       {
            // your code to access items in the container... $this->container->get('');

            return $response;
       }
    }

You can use this class like so.

    $app->get('/', \HomeAction::class);

Again, as with controllers, if you register the class name with the container, then you can create a factory and inject just the specific dependencies that you require into your action class.

## Route Object

Sometimes in middleware you require the parameter of your route.

In this example we are checking first that the user is logged in and second that the user has permissions to view the particular video they are attempting to view.

    $app->get('/course/{id}', Video::class . ':watch')
        ->add(PermissionMiddleware::class);


    <?php

    use Psr\Http\Message\ServerRequestInterface as Request;
    use Psr\Http\Server\RequestHandlerInterface as RequestHandler;
    use Slim\Routing\RouteContext;

    class PermissionMiddleware
    {
        public function __invoke(Request $request, RequestHandler $handler)
        {
            $routeContext = RouteContext::fromRequest($request);
            $route = $routeContext->getRoute();

            $courseId = $route->getArgument('id');

            // do permission logic...

            return $handler->handle($request);
        }
    }

## Obtain Base Path From Within Route

To obtain the base path from within a route simply do the following:

    <?php

    use Psr\Http\Message\ResponseInterface as Response;
    use Psr\Http\Message\ServerRequestInterface as Request;
    use Slim\Factory\AppFactory;
    use Slim\Routing\RouteContext;

    require __DIR__ . '/../vendor/autoload.php';

    $app = AppFactory::create();

    $app->get('/', function(Request $request, Response $response) {
        $routeContext = RouteContext::fromRequest($request);
        $basePath = $routeContext->getBasePath();
        // ...

        return $response;
    });

- [The Response: Overview](/docs/v4/objects/response.html)
- [Packaged Middleware: Routing](/docs/v4/middleware/routing.html)

# Routing Middleware - Slim Framework

# Routing Middleware

[Edit This Page](https://github.com/slimphp/Slim-Website/tree/gh-pages/docs/v4/middleware/routing.md)

The routing has been implemented as middleware. We are still using [FastRoute](https://github.com/nikic/FastRoute) as the default router but it is not tightly coupled to it. If you wanted to implement another routing library you could by creating your own implementations of the routing interfaces. `DispatcherInterface`, `RouteCollectorInterface`, `RouteParserInterface` and `RouteResolverInterface` create a bridge between Slim’s components and the routing library. If you were using `determineRouteBeforeAppMiddleware`, you need to add the `Middleware\RoutingMiddleware` middleware to your application just before your call to `run()` to maintain the previous behaviour.

## Usage

    <?php

    use Slim\Factory\AppFactory;

    require __DIR__ . '/../vendor/autoload.php';

    $app = AppFactory::create();

    // Add Routing Middleware
    $app->addRoutingMiddleware();

    // ...

    $app->run();

- [Routing: Overview](/docs/v4/objects/routing.html)
- [Packaged Middleware: Error Handling](/docs/v4/middleware/error-handling.html)

# Error Middleware - Slim Framework

# Error Middleware

[Edit This Page](https://github.com/slimphp/Slim-Website/tree/gh-pages/docs/v4/middleware/error-handling.md)

Things go wrong. You can’t predict errors, but you can anticipate them. Each Slim Framework application has an error handler that receives all uncaught PHP exceptions. This error handler also receives the current HTTP request and response objects, too. The error handler must prepare and return an appropriate Response object to be returned to the HTTP client.

## Usage

    <?php

    use Slim\Factory\AppFactory;

    require __DIR__ . '/../vendor/autoload.php';

    $app = AppFactory::create();

    /**
     * The routing middleware should be added earlier than the ErrorMiddleware
     * Otherwise exceptions thrown from it will not be handled by the middleware
     */
    $app->addRoutingMiddleware();

    /**
     * Add Error Middleware
     *
     * @param bool                  $displayErrorDetails -> Should be set to false in production
     * @param bool                  $logErrors -> Parameter is passed to the default ErrorHandler
     * @param bool                  $logErrorDetails -> Display error details in error log
     * @param LoggerInterface|null  $logger -> Optional PSR-3 Logger
     *
     * Note: This middleware should be added last. It will not handle any exceptions/errors
     * for middleware added after it.
     */
    $errorMiddleware = $app->addErrorMiddleware(true, true, true);

    // ...

    $app->run();

## Adding Custom Error Handlers

You can now map custom handlers for any type of Exception or Throwable.

    <?php

    use Monolog\Handler\RotatingFileHandler;
    use Monolog\Logger;
    use Psr\Http\Message\ServerRequestInterface;
    use Psr\Log\LoggerInterface;
    use Slim\Factory\AppFactory;

    require __DIR__ . '/../vendor/autoload.php';

    $app = AppFactory::create();

    // Add Routing Middleware
    $app->addRoutingMiddleware();

    // Optional: Define custom error logger
    $logger = new Logger('error');
    $logger->pushHandler(new RotatingFileHandler('error.log'));

    // Define Custom Error Handler
    $customErrorHandler = function (
        ServerRequestInterface $request,
        Throwable $exception,
        bool $displayErrorDetails,
        bool $logErrors,
        bool $logErrorDetails
    ) use ($app, $logger) {
        if ($logger) {
            $logger->error($exception->getMessage());
        }

        $payload = ['error' => $exception->getMessage()];

        $response = $app->getResponseFactory()->createResponse();
        $response->getBody()->write(
            json_encode($payload, JSON_UNESCAPED_UNICODE)
        );

        return $response;
    };

    // Add Error Middleware
    $errorMiddleware = $app->addErrorMiddleware(true, true, true, $logger);
    $errorMiddleware->setDefaultErrorHandler($customErrorHandler);

    // ...

    $app->run();

## Error Logging

If you would like to pipe in custom error logging to the default `ErrorHandler` that ships with Slim, there are two ways to do it.

With the first method, you can simply extend `ErrorHandler` and stub the `logError()` method.

    <?php
    namespace MyApp\Handlers;

    use Slim\Handlers\ErrorHandler;

    class MyErrorHandler extends ErrorHandler
    {
        protected function logError(string $error): void
        {
            // Insert custom error logging function.
        }
    }


    <?php

    use MyApp\Handlers\MyErrorHandler;
    use Slim\Factory\AppFactory;

    require __DIR__ . '/../vendor/autoload.php';

    $app = AppFactory::create();

    // Add Routing Middleware
    $app->addRoutingMiddleware();

    // Instantiate Your Custom Error Handler
    $myErrorHandler = new MyErrorHandler($app->getCallableResolver(), $app->getResponseFactory());

    // Add Error Middleware
    $errorMiddleware = $app->addErrorMiddleware(true, true, true);
    $errorMiddleware->setDefaultErrorHandler($myErrorHandler);

    // ...

    $app->run();

With the second method, you can supply a logger that conforms to the [PSR-3 standard](https://www.php-fig.org/psr/psr-3/), such as one from the popular [Monolog](https://github.com/Seldaek/monolog/) library.

    <?php

    use Monolog\Handler\StreamHandler;
    use Monolog\Logger;
    use MyApp\Handlers\MyErrorHandler;
    use Slim\Factory\AppFactory;

    require __DIR__ . '/../vendor/autoload.php';

    $app = AppFactory::create();

    // Add Routing Middleware
    $app->addRoutingMiddleware();

    // Monolog Example
    $logger = new Logger('app');
    $streamHandler = new StreamHandler(__DIR__ . '/var/log', 100);
    $logger->pushHandler($streamHandler);

    // Add Error Middleware with Logger
    $errorMiddleware = $app->addErrorMiddleware(true, true, true, $logger);

    // ...

    $app->run();

## Error Handling/Rendering

The rendering is finally decoupled from the handling. It will still detect the content-type and render things appropriately with the help of `ErrorRenderers`. The core `ErrorHandler` extends the `AbstractErrorHandler` class which has been completely refactored. By default it will call the appropriate `ErrorRenderer` for the supported content types. The core `ErrorHandler` defines renderers for the following content types:

- `application/json`
- `application/xml` and `text/xml`
- `text/html`
- `text/plain`

For any content type you can register your own error renderer. So first define a new error renderer that implements `\Slim\Interfaces\ErrorRendererInterface`.

    <?php

    use Slim\Interfaces\ErrorRendererInterface;
    use Throwable;

    class MyCustomErrorRenderer implements ErrorRendererInterface
    {
        public function __invoke(Throwable $exception, bool $displayErrorDetails): string
        {
            return 'My awesome format';
        }
    }

And then register that error renderer in the core error handler. In the example below we will register the renderer to be used for `text/html` content types.

    <?php

    use MyApp\Handlers\MyErrorHandler;
    use Slim\Factory\AppFactory;

    require __DIR__ . '/../vendor/autoload.php';

    $app = AppFactory::create();

    // Add Routing Middleware
    $app->addRoutingMiddleware();

    // Add Error Middleware
    $errorMiddleware = $app->addErrorMiddleware(true, true, true);

    // Get the default error handler and register my custom error renderer.
    $errorHandler = $errorMiddleware->getDefaultErrorHandler();
    $errorHandler->registerErrorRenderer('text/html', MyCustomErrorRenderer::class);

    // ...

    $app->run();

### Force a specific content type for error rendering

By default, the error handler tries to detect the error renderer using the `Accept` header of the request. If you need to force the error handler to use a specific error renderer you can write the following.

    $errorHandler->forceContentType('application/json');

## New HTTP Exceptions

We have added named HTTP exceptions within the application. These exceptions work nicely with the native renderers. They can each have a `description` and `title` attribute as well to provide a bit more insight when the native HTML renderer is invoked.

The base class `HttpSpecializedException` extends `Exception` and comes with the following sub classes:

- HttpBadRequestException
- HttpForbiddenException
- HttpInternalServerErrorException
- HttpMethodNotAllowedException
- HttpNotFoundException
- HttpNotImplementedException
- HttpUnauthorizedException

You can extend the `HttpSpecializedException` class if they need any other response codes that we decide not to provide with the base repository. Example if you wanted a 504 gateway timeout exception that behaves like the native ones you would do the following:

    class HttpGatewayTimeoutException extends HttpSpecializedException
    {
        protected $code = 504;
        protected $message = 'Gateway Timeout.';
        protected string $title = '504 Gateway Timeout';
        protected string $description = 'Timed out before receiving response from the upstream server.';
    }

To throw HTTP exceptions, use the following code:

    use Slim\Exception\HttpNotFoundException;
    // ...

    throw new HttpNotFoundException($request);

Ensure you pass the `$request` object when throwing the exception.

- [Packaged Middleware: Routing](/docs/v4/middleware/routing.html)
- [Packaged Middleware: Method Overriding](/docs/v4/middleware/method-overriding.html)

# Method Overriding Middleware - Slim Framework

# Method Overriding Middleware

[Edit This Page](https://github.com/slimphp/Slim-Website/tree/gh-pages/docs/v4/middleware/method-overriding.md)

The Method Overriding Middleware enables you to use the `X-Http-Method-Override` request header or the request body parameter `_METHOD` to override an incoming request’s method. The middleware should be placed after the routing middleware has been added.

## Usage

    <?php

    use Slim\Factory\AppFactory;
    use Slim\Middleware\MethodOverrideMiddleware;

    require __DIR__ . '/../vendor/autoload.php';

    $app = AppFactory::create();

    // Add RoutingMiddleware before we add the MethodOverrideMiddleware so the method is overridden before routing is done
    $app->addRoutingMiddleware();

    // Add MethodOverride middleware
    $methodOverrideMiddleware = new MethodOverrideMiddleware();
    $app->add($methodOverrideMiddleware);

    // ...

    $app->run();

- [Packaged Middleware: Error Handling](/docs/v4/middleware/error-handling.html)
- [Packaged Middleware: Output Buffering](/docs/v4/middleware/output-buffering.html)

# Output Buffering Middleware - Slim Framework

# Output Buffering Middleware

[Edit This Page](https://github.com/slimphp/Slim-Website/tree/gh-pages/docs/v4/middleware/output-buffering.md)

The Output Buffering Middleware enables you to switch between two modes of output buffering: `APPEND` (default) and `PREPEND` mode. The `APPEND` mode will use the existing response body to append the contents. The `PREPEND` mode will create a new response body object and prepend the contents to the output from the existing response body. This middleware should be placed in the center of the middleware stack so it gets executed last.

## Usage

    <?php

    use Slim\Factory\AppFactory;
    use Slim\Middleware\OutputBufferingMiddleware;
    use Slim\Psr7\Factory\StreamFactory;

    require __DIR__ . '/../vendor/autoload.php';

    $app = AppFactory::create();

    $streamFactory = new StreamFactory();

    /**
     * The two modes available are
     * OutputBufferingMiddleware::APPEND (default mode) - Appends to existing response body
     * OutputBufferingMiddleware::PREPEND - Creates entirely new response body
     */
    $mode = OutputBufferingMiddleware::APPEND;
    $outputBufferingMiddleware = new OutputBufferingMiddleware($streamFactory, $mode);
    $app->add($outputBufferingMiddleware);

    // ...

    $app->run();

- [Packaged Middleware: Method Overriding](/docs/v4/middleware/method-overriding.html)
- [Packaged Middleware: Body Parsing](/docs/v4/middleware/body-parsing.html)

# Body Parsing Middleware - Slim Framework

# Body Parsing Middleware

[Edit This Page](https://github.com/slimphp/Slim-Website/tree/gh-pages/docs/v4/middleware/body-parsing.md)

It’s very common in web APIs to send data in JSON or XML format. Out of the box, PSR-7 implementations do not support these formats, you have to decode the Request object’s getBody() yourself. As this is a common requirement, Slim 4 provides `BodyParsingMiddleware` to handle this task.

## Usage

It’s recommended to put the body parsing middleware before the call to `addErrorMiddlware`, so that the stack looks like this:

    <?php

    use Slim\Factory\AppFactory;

    require_once __DIR__ . '/../vendor/autoload.php';

    $app = AppFactory::create();

    // Parse json, form data and xml
    $app->addBodyParsingMiddleware();

    $app->addRoutingMiddleware();

    $app->addErrorMiddleware(true, true, true);

    // ...

    $app->run();

## Posted JSON, form or XML data

No changes are required to the POST handler because the `BodyParsingMiddleware` detects that the `Content-Type` is set to a `JSON` media type and so places the decoded body into the Request’s parsed body property.

For data posted to the website from a browser, you can use the $request’s `getParsedBody()` method.

This will return an array of the posted data.

    $app->post('/', function (Request $request, Response $response, $args): Response {
        $data = $request->getParsedBody();

        $html = var_export($data, true);
        $response->getBody()->write($html);

        return $response;
    });

## Media type detection

- The middleware reads the `Content-Type` from the request header to detect the media type.
- Checks if this specific media type has a parser registered.
- If not, look for a media type with a structured syntax suffix (RFC 6839), e.g. `application/`.

## Supported media types

- application/json
- application/x-www-form-urlencoded
- application/xml
- text/xml

- [Packaged Middleware: Output Buffering](/docs/v4/middleware/output-buffering.html)
- [Packaged Middleware: Content Length](/docs/v4/middleware/content-length.html)

# Content Length Middleware - Slim Framework

# Content Length Middleware

[Edit This Page](https://github.com/slimphp/Slim-Website/tree/gh-pages/docs/v4/middleware/content-length.md)

The Content Length Middleware will automatically append a `Content-Length` header to the response. This is to replace the `addContentLengthHeader` setting that was removed from Slim 3. This middleware should be placed on the end of the middleware stack so that it gets executed first and exited last.

## Usage

    <?php

    use Slim\Factory\AppFactory;
    use Slim\Middleware\ContentLengthMiddleware;

    require __DIR__ . '/../vendor/autoload.php';

    $app = AppFactory::create();

    // Add any middleware which may modify the response body before adding the ContentLengthMiddleware

    $contentLengthMiddleware = new ContentLengthMiddleware();
    $app->add($contentLengthMiddleware);

    // ...

    $app->run();

- [Packaged Middleware: Body Parsing](/docs/v4/middleware/body-parsing.html)
- [Cook book: Trailing / in routes](/docs/v4/cookbook/route-patterns.html)

# Trailing / in route patterns - Slim Framework

# Trailing / in route patterns

[Edit This Page](https://github.com/slimphp/Slim-Website/tree/gh-pages/docs/v4/cookbook/route-patterns.md)

Slim treats a URL pattern with a trailing slash as different to one without. That is, `/user` and `/user/` are different and so can have different callbacks attached.

For GET requests a permanent redirect is fine, but for other request methods like POST or PUT the browser will send the second request with the GET method. To avoid this you simply need to remove the trailing slash in the `Request` object and pass the manipulated url to the next middleware.

If you want to redirect/rewrite all URLs that end in a `/` to the non-trailing `/` equivalent, consider [middlewares/trailing-slash](//github.com/middlewares/trailing-slash) middleware. Alternatively, the middlware also allows you to force a trailing slash to be appended to all URLs.

    use Middlewares\TrailingSlash;

    $app->add(new TrailingSlash(trailingSlash: true)); // true adds the trailing slash (false removes it)

- [Packaged Middleware: Content Length](/docs/v4/middleware/content-length.html)
- [Cook book: Retrieving Current Route](/docs/v4/cookbook/retrieving-current-route.html)

# Retrieving Current Route - Slim Framework

# Retrieving Current Route

[Edit This Page](https://github.com/slimphp/Slim-Website/tree/gh-pages/docs/v4/cookbook/retrieving-current-route.md)

If you ever need to get access to the current route within your application, you will need to instantiate the `RouteContext` object using the incoming `ServerRequestInterface`.

From there you can get the route via `$routeContext->getRoute()` and access the route’s name by using `getName()` or get the methods supported by this route via `getMethods()`, etc.

**Note:** If you need to access the `RouteContext` object during the middleware cycle before reaching the route handler you will need to add the `RoutingMiddleware` as the outermost middleware before the error handling middleware (See example below).

Example:

    <?php

    use Slim\Exception\HttpNotFoundException;
    use Slim\Factory\AppFactory;
    use Slim\Routing\RouteContext;

    require __DIR__ . '/../vendor/autoload.php';

    $app = AppFactory::create();

    // Via this middleware you could access the route and routing results from the resolved route
    $app->add(function (Request $request, RequestHandler $handler) {
        $routeContext = RouteContext::fromRequest($request);
        $route = $routeContext->getRoute();

        // return NotFound for non-existent route
        if (empty($route)) {
            throw new HttpNotFoundException($request);
        }

        $name = $route->getName();
        $groups = $route->getGroups();
        $methods = $route->getMethods();
        $arguments = $route->getArguments();

        // ... do something with the data ...

        return $handler->handle($request);
    });

    // The RoutingMiddleware should be added after our CORS middleware so routing is performed first
    $app->addRoutingMiddleware();

    // The ErrorMiddleware should always be the outermost middleware
    $app->addErrorMiddleware(true, true, true);

    // ...

    $app->run();

- [Cook book: Trailing / in routes](/docs/v4/cookbook/route-patterns.html)
- [Cook book: Using Doctrine with Slim](/docs/v4/cookbook/database-doctrine.html)

# Setting up CORS - Slim Framework

# Setting up CORS

[Edit This Page](https://github.com/slimphp/Slim-Website/tree/gh-pages/docs/v4/cookbook/enable-cors.md)

Cross-Origin Resource Sharing ([CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)) is a security feature implemented in web browsers that allows or restricts web pages from making requests to a domain different from the one that served the web page.

It is a mechanism that enables controlled access to resources located outside of a given domain.

CORS is essential for enabling secure communication between different web applications while preventing malicious cross-origin requests.

By specifying certain headers, servers can indicate which origins are permitted to access their resources, thus maintaining a balance between usability and security.

A good flowchart for implementing CORS support: [CORS Server Flowchart](https://www.html5rocks.com/static/images/cors_server_flowchart.png)

You can read the specification here: [https://fetch.spec.whatwg.org/#cors-protocol](https://fetch.spec.whatwg.org/#cors-protocol)

## The simple solution

For simple CORS requests, the server only needs to add the following header to its response:

    Access-Control-Allow-Origin: <domain>, ...

The following code should enable lazy CORS.

    $app->options('/{routes:.+}', function ($request, $response, $args) {
        return $response;
    });

    $app->add(function ($request, $handler) {
        $response = $handler->handle($request);
        return $response
                ->withHeader('Access-Control-Allow-Origin', 'http://mysite')
                ->withHeader('Access-Control-Allow-Headers', 'X-Requested-With, Content-Type, Accept, Origin, Authorization')
                ->withHeader('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE, PATCH, OPTIONS');
    });

**Optional:** Add the following route as the last route:

    use Slim\Exception\HttpNotFoundException;

    /**
     * Catch-all route to serve a 404 Not Found page if none of the routes match
     * NOTE: make sure this route is defined last
     */
    $app->map(['GET', 'POST', 'PUT', 'DELETE', 'PATCH'], '/{routes:.+}', function ($request, $response) {
        throw new HttpNotFoundException($request);
    });

## CORS example application

Here is a complete CORS example application that uses a CORS middleware:

    <?php

    use Psr\Http\Message\ResponseInterface;
    use Psr\Http\Message\ServerRequestInterface;
    use Psr\Http\Server\RequestHandlerInterface;
    use Slim\Factory\AppFactory;

    require_once __DIR__ . '/../vendor/autoload.php';

    $app = AppFactory::create();

    $app->addBodyParsingMiddleware();

    // Add the RoutingMiddleware before the CORS middleware
    // to ensure routing is performed later
    $app->addRoutingMiddleware();

    // Add the ErrorMiddleware before the CORS middleware
    // to ensure error responses contain all CORS headers.
    $app->addErrorMiddleware(true, true, true);

    // This CORS middleware will append the response header
    // Access-Control-Allow-Methods with all allowed methods
    $app->add(function (ServerRequestInterface $request, RequestHandlerInterface $handler) use ($app): ResponseInterface {
        if ($request->getMethod() === 'OPTIONS') {
            $response = $app->getResponseFactory()->createResponse();
        } else {
            $response = $handler->handle($request);
        }

        $response = $response
            ->withHeader('Access-Control-Allow-Credentials', 'true')
            ->withHeader('Access-Control-Allow-Origin', '*')
            ->withHeader('Access-Control-Allow-Headers', '*')
            ->withHeader('Access-Control-Allow-Methods', 'GET, POST, PUT, PATCH, DELETE, OPTIONS')
            ->withHeader('Cache-Control', 'no-store, no-cache, must-revalidate, max-age=0')
            ->withHeader('Pragma', 'no-cache');

        if (ob_get_contents()) {
            ob_clean();
        }

        return $response;
    });

    // Define app routes
    $app->get('/', function (ServerRequestInterface $request, ResponseInterface $response) {
        $response->getBody()->write('Hello, World!');

        return $response;
    });

    // ...

    $app->run();

## Access-Control-Allow-Credentials

If the request contains credentials (cookies, authorization headers or TLS client certificates), you might need to add an `Access-Control-Allow-Credentials` header to the response object.

    $response = $response->withHeader('Access-Control-Allow-Credentials', 'true');

- [Cook book: Using Doctrine with Slim](/docs/v4/cookbook/database-doctrine.html)
- [Cook book: Uploading Files using POST forms](/docs/v4/cookbook/uploading-files.html)

# Uploading files using POST forms - Slim Framework

# Uploading files using POST forms

[Edit This Page](https://github.com/slimphp/Slim-Website/tree/gh-pages/docs/v4/cookbook/uploading-files.md)

Files that are uploaded using forms in POST requests can be retrieved with the Request method `getUploadedFiles()`.

When uploading files using a POST request, make sure your file upload form has the attribute `enctype="multipart/form-data"` otherwise `getUploadedFiles()` will return an empty array.

If multiple files are uploaded for the same input name, add brackets after the input name in the HTML, otherwise only one uploaded file will be returned for the input name by `getUploadedFiles()`.

Below is an example HTML form that contains both single and multiple file uploads.

    <!-- make sure the attribute enctype is set to multipart/form-data -->
    <form method="post" enctype="multipart/form-data">
        <!-- upload of a single file -->
        <p>
            <label>Add file (single): </label><br/>
            <input type="file" name="example1"/>
        </p>

        <!-- multiple input fields for the same input name, use brackets -->
        <p>
            <label>Add files (up to 2): </label><br/>
            <input type="file" name="example2[]"/><br/>
            <input type="file" name="example2[]"/>
        </p>

        <!-- one file input field that allows multiple files to be uploaded, use brackets -->
        <p>
            <label>Add files (multiple): </label><br/>
            <input type="file" name="example3[]" multiple="multiple"/>
        </p>

        <p>
            <input type="submit"/>
        </p>
    </form>

Figure 1: Example HTML form for file uploads

Uploaded files can be moved to a directory using the `moveTo` method. Below is an example application that handles the uploaded files of the HTML form above.

    <?php

    use DI\ContainerBuilder;
    use Psr\Http\Message\ResponseInterface;
    use Psr\Http\Message\ServerRequestInterface;
    use Psr\Http\Message\UploadedFileInterface;
    use Slim\Factory\AppFactory;

    require __DIR__ . '/../vendor/autoload.php';

    $containerBuilder = new ContainerBuilder();
    $container = $containerBuilder->build();

    $container->set('upload_directory', __DIR__ . '/uploads');

    AppFactory::setContainer($container);
    $app = AppFactory::create();

    $app->post('/', function (ServerRequestInterface $request, ResponseInterface $response) {
        $directory = $this->get('upload_directory');
        $uploadedFiles = $request->getUploadedFiles();

        // handle single input with single file upload
        $uploadedFile = $uploadedFiles['example1'];
        if ($uploadedFile->getError() === UPLOAD_ERR_OK) {
            $filename = moveUploadedFile($directory, $uploadedFile);
            $response->getBody()->write('Uploaded: ' . $filename . '<br/>');
        }

        // handle multiple inputs with the same key
        foreach ($uploadedFiles['example2'] as $uploadedFile) {
            if ($uploadedFile->getError() === UPLOAD_ERR_OK) {
                $filename = moveUploadedFile($directory, $uploadedFile);
                $response->getBody()->write('Uploaded: ' . $filename . '<br/>');
            }
        }

        // handle single input with multiple file uploads
        foreach ($uploadedFiles['example3'] as $uploadedFile) {
            if ($uploadedFile->getError() === UPLOAD_ERR_OK) {
                $filename = moveUploadedFile($directory, $uploadedFile);
                $response->getBody()->write('Uploaded: ' . $filename . '<br/>');
            }
        }

        return $response;
    });

    /**
     * Moves the uploaded file to the upload directory and assigns it a unique name
     * to avoid overwriting an existing uploaded file.
     *
     * @param string $directory The directory to which the file is moved
     * @param UploadedFileInterface $uploadedFile The file uploaded file to move
     *
     * @return string The filename of moved file
     */
    function moveUploadedFile(string $directory, UploadedFileInterface $uploadedFile)
    {
        $extension = pathinfo($uploadedFile->getClientFilename(), PATHINFO_EXTENSION);

        // see http://php.net/manual/en/function.random-bytes.php
        $basename = bin2hex(random_bytes(8));
        $filename = sprintf('%s.%0.8s', $basename, $extension);

        $uploadedFile->moveTo($directory . DIRECTORY_SEPARATOR . $filename);

        return $filename;
    }

    $app->run();

Figure 2: Example Slim application to handle the uploaded files

- [Cook book: Enabling CORS](/docs/v4/cookbook/enable-cors.html)
- [Add Ons: Templates](/docs/v4/features/templates.html)

# Twig Templates - Slim Framework

# Twig Templates

[Edit This Page](https://github.com/slimphp/Slim-Website/tree/gh-pages/docs/v4/features/twig-view.md)

## The slim/twig-view component

The [Twig-View](https://github.com/slimphp/Twig-View) PHP component helps you render [Twig](https://twig.symfony.com/) templates in your application. This component is available on Packagist, and it’s easy to install with Composer like this:

## Installation

    composer require slim/twig-view

## Usage

Next, you need to add the middleware to the Slim app:

    <?php

    use Slim\Factory\AppFactory;
    use Slim\Views\Twig;
    use Slim\Views\TwigMiddleware;

    require __DIR__ . '/../vendor/autoload.php';

    // Create App
    $app = AppFactory::create();

    // Create Twig
    $twig = Twig::create(__DIR__ . '/../templates', ['cache' => false]);

    // Add Twig-View Middleware
    $app->add(TwigMiddleware::create($app, $twig));

**Note:** For production scenarios, `cache` should be set to some `'path/to/cache'` to store compiled templates (thus avoiding recompilation on every request). For more information, see [Twig environment options](https://twig.symfony.com/doc/3.x/api.html#environment-options)

Now you can use the `slim/twig-view` component service inside an app route to render a template and write it to a PSR-7 Response object like this:

    $app->get('/', function ($request, $response) {
        $view = Twig::fromRequest($request);

        return $view->render($response, 'home.html.twig', [
            'name' => 'John',
        ]);
    });

    // Run app
    $app->run();

In this example, `$view` invoked inside the route callback is a reference to the `\Slim\Views\Twig` instance returned by the `fromRequest` method. The `\Slim\Views\Twig` instance’s `render()` method accepts a PSR-7 Response object as its first argument, the Twig template path as its second argument, and an array of template variables as its final argument. The `render()` method returns a new PSR-7 Response object whose body is the rendered Twig template.

Create a directory in your project root: `templates/`

Create a Twig template file within the templates directory: `templates/home.html.twig`

    <!DOCTYPE html>
    <html>
    <head>
        <title>Welcome to Slim!</title>
    </head>
    <body>
    <h1>Hello {{ name }}</h1>
    </body>
    </html>

### The url_for() method

The `slim/twig-view` component exposes a custom `url_for()` function to your Twig templates. You can use this function to generate complete URLs to any named route in your Slim application. The `url_for()` function accepts two arguments:

1.  A route name
2.  A hash of route placeholder names and replacement values

The second argument’s keys should correspond to the selected route’s pattern placeholders. This is an example Twig template that draws a link URL for the “profile” named route shown in the example Slim application above.

    <a href="{{ url_for('profile', { 'name': 'josh' }) }}">Josh</a></li>

## Read more

- [Twig-View documentation](https://github.com/slimphp/Twig-View)
- [Twig documentation](https://twig.symfony.com/)

- [Add Ons: Templates](/docs/v4/features/templates.html)
- [Add Ons: PHP Templates](/docs/v4/features/php-view.html)
