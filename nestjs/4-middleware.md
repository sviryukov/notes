# Middleware

Middleware в nest выполняет ту же задачу, что и в express. Middleware может быть функцией или классом с декоратором
@Injectable().

### Middleware-классы

-  К классу middleware должен быть применен декоратор @Injectable().
-  Класс middleware должен реализовывать интерфейс NestMiddleware и его метод use(req, res, next), классы аргументов — 
   из express.
-  Модуль, использующий класс middleware, должен реализовывать интерфейс NestModule и его метод configure(consumer), 
   consumer - объект, реализующий интерфейс MiddlewareConsumer. Метод configure также может быть асинхронным.

```typescript
import { Injectable, NestMiddlewar, Module, NestModule, MiddlewareConsumer, RequestMethod } from '@nest/common';
import { Request, Response, NextFunction } from 'express';

@Injectable()  
export class MyMiddleware implements NestMiddleware {
   use(req: Request, res: Response, next: NextFunction) {
      // ...
      next();
   }
}

@Module({})
export class MyModule implements NestModule {
   configure(consumer: MiddlewareConsumer) {
       consumer.apply(MyMiddleware).forRoutes({ path: 'route', method: RequestMethod.GET });
   }
}
```

### MiddlewareConsumer

-  Метод apply классса MiddlewareConsumer принимает одно или несколько middleware (классов или функций):

   ```typescript
   consumer.apply(MyMiddleware1, MyMiddleware2, myMiddleware3).forRoutes(MyController);
   ```

-  Метод forRoutes задает, к каким запросам будет применено middleware. Он может в качестве аргументов принимать
   любую комбинацию из строк с маршрутами или шаблонами маршрутов, объектов RouteInfo с полями path и method
   и классов-контроллеров:

   ```typescript
   consumer
      .apply(MyMiddleware)
      .forRoutes(
         'route1',
         'route2*',
         { path: 'route3', method: RequestMethod.GET },
         { path: 'route3', method: RequestMethod.POST },
         MyController
      );
   ```

-  Метод exclude задает, к каким запросам не будет применено middleware. Он может в качестве аргументов принимать
   любую комбинацию из строк с маршрутами или шаблонами маршрутов и объектов RouteInfo:

   ```typescript
   consumer
      .apply(MyMiddleware)
      .exclude(
         'route1*',
         { path: 'route2', method: RequestMethod.GET },
         { path: 'route2', method: RequestMethod.POST }
      )
      .forRoutes(
         'route3*',
         MyController
      );
   ```
   
   exclude должен быть вызван перед forRoutes.

### Middleware-функции:

```typescript
import { Request, Response, NextFunction } from 'express';

export function myMiddleware(req: Request, res: Response, next: NextFunction) {
   // ...
   next();
};
```
```typescript
consumer
   .apply(myMiddleware)
   .forRoutes(MyController);
```

### Глобальное middleware

myMiddleware будет применено ко всем маршрутам в приложении:

```typescript
const app = await NestFactory.create(AppModule);
app.use(myMiddleware);
```