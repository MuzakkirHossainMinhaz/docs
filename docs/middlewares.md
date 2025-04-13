---
title: Middlewares
description:
image:
---
# Middlewares

Middleware is a function which is called before the route handler method. Middleware functions have access to `request` and `response` objects,
and `next()` function which calls the next middleware or the route method in the applications' request response life cycle.

You can use middlewares to handle the following use cases
1. execute some piece of code
2. modify the request response life cycle.
3. Pre-Validate the request headers, or on payload.
4. Call the `next` middleware function in the stack, etc.

:::info
Middlewares are dumb in nature, which means they don't know which route handler will be called to handle the request. If you are looking to
access the context of route handler, you can take a look at [Guards](./guards.md).
:::

## Creating Middlewares
To create a middleware, let's create a file `api-key-middleware.ts` inside the `app/http/middlewares` directory, you will need to create a class like shown below inside the file we just created.

```ts
import { HttpMiddleware } from '@intentjs/core/http';
import { MiddlewareNext, Request, Response } from '@intentjs/hyper-express'; 

export class ApiKeyMiddleware extends HttpMiddleware {
  use(req: Request, res: Response, next: MiddlewareNext): void {
    const payload = req.all();
    // your code here.
    next();
  }
}
```

By default nature of the middleware is supposed to be synchronous, but if would like to make an asynchronous middleware then you don't need to call the `next` method.

```ts
import { HttpMiddleware } from '@intentjs/core/http';
import { MiddlewareNext, Request, Response } from '@intentjs/hyper-express'; 

export class ApiKeyMiddleware extends HttpMiddleware {
  async use(req: Request, res: Response, next: MiddlewareNext): Promise<void> {
    new Promise((resolve, reject) => {
      // your code here
      resolve(1);
    })
  }
}
```

We will write our logic inside the `use` method, which will run whenever there is any request coming in.

Middlewares are fully compatible with our application containers and service providers, which means you can inject some dependencies inside the `ApiKeyMiddleware` like below.

```ts
import { HttpMiddleware } from '@intentjs/core/http';
import { MiddlewareNext, Request, Response } from '@intentjs/hyper-express'; 

@Injectable()
export class ApiKeyMiddleware extends HttpMiddleware {
  constructor(private service: UserService) {}

  async use(req: Request, res: Response, next: MiddlewareNext): Promise<void> {
    next();
  }
}
```

Let's say you want to throw some error from the middleware itself, you can throw an `HttpException` error. This would automatically get captured by our error filter.

```ts
async use(req: Request, res: Response, next: NextFunction): Promise<void> {
  const isApiKeyMatching = false; // add your logic here.
  if (!isApiKeyMatching) {
    throw new HttpException('API Keys mismatch. Please double check once!');
  }
  next();
}
```

## Applying Middlewares

After you are done creating the middlewares, we will now need to start applying it to our controllers.
Intent supports following type of applications for the middlewares,

1. Global Middleware - gets applied on all incoming routes
2. Route Based Middleware - gets applied on specific routes, group of routes, or excluding some routes.

To apply the middleware globally, you can simply add the `ApiKeyMiddleware` we built in the `app/http/kernel.ts`.

```ts
import {
  CorsMiddleware,
  HttpGuard,
  HttpMethods,
  HttpMiddleware,
  Kernel,
  MiddlewareConfigurator,
} from '@intentjs/core/http';
import { ApiKeyMiddleware } from '#http/middlewares/api-key';

export class HttpKernel extends Kernel {
  /**
   * Register all of your global middlewares here.
   * Middlewares added in the return array will be
   * applied to all routes by default.
   */
  public middlewares(): Type<HttpMiddleware>[] {
    return [
      // existing middlewares here...
      ApiKeyMiddleware
    ];
  }
}
```

What if you don't want to apply a middleware globally, and instead just apply it on certain routes. For this, you can simply use the `routeMiddlewares` available inside the `Kernel` class.

```ts [app/http/kernel.ts]
import { Kernel, HttpMiddleware, Type, MiddlewareConfigurator } from '@intentjs/core/http';
import { ApiKeyMiddleware } from '#http/middlewares/api-key';

export class HttpKernel extends Kernel {
  /**
   * Register all of your route specific middlewares here.
   */
  public routeMiddlewares(configurator: MiddlewareConfigurator): Type<HttpMiddleware>[] {
    configurator
      .use(ApiKeyMiddleware)
      .for(UserController);
  }
}
```

This would apply the `ApiKeyMiddleware` on all of the routes mentioned in the `UserController`.
If you don't want to pass the controller to the `for` method, you can also pass a route. The route passed will be considered
just a prefix.

```ts
configurator
  .use(ApiKeyMiddleware)
  .for('/users');
```

You can also exclude certain routes from this. This would then run the middleware on all of the routes except the ones mentioned explicitly.

```ts
configurator
  .use(ApiKeyMiddleware)
  .for('/users')
  .exclude('/users/:id');
```

The above example applies the `ApiKeyMiddleware` on all routes starting with `/users` prefix, except the `/users/:id` route.

You might have some routes with different HTTP Methods, and you wish to exclude a certain route with a certain method, you can pass
`{ path: '/users/:id', method: RequestMethod.POST }`.

```ts
configurator
  .use(ApiKeyMiddleware)
  .for('/users')
  .exclude({ path: '/users/:id', method: RequestMethod.POST });
```

The above configuration will run the `ApiKeyMiddleware` on all routes starting with `/users`, except the `/users/:id` routes with POST method.
