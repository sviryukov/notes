# Контроллеры

@Controller(prefix?: string) - дескриптор класса, использующийся для создания контроллеров. prefix - префикс маршрутов
методов, обрабатываемых этим контроллером. Контроллер отвечает за обработку HTTP-запросов.

### Дескрипторы методов контроллера

1. @Get(path?: string), @Post(path?: string), @Pout(path?: string), @Delete(path?: string), @Patch(path?: string),
   @Options(path?: string), @Head(path?: string) - дескрипторы методов контроллера, отвечающих за обработку
   конкретных запросов. @All(path?: string) - дескриптор метода, отвечающего за обработку всех запросов по данному
   маршруту. Итоговый маршрут выглядит как '/prefix/path', '/prefix', '/path', '' в зависимости от того, переданы ли
   аргументы в дескрипторы класса-контроллера и его метода.

    -  Дескрипторы методов контроллера предоставляют доступ к параметрам маршрута:

       ```typescript
       import { Controller, Get, Param } from '@nest/common'; 
       
       @Controller()
       export class MyController {
          @Get(':id')
          method(@Param('id') id: string): string {
             return `response for ${id}`;
          }
       }
       ```

    -  Методы по умолчанию возвращают код 200 (POST - 201), в ответ на запрос возвращают сериализованный JSON, если
       метод возвращает объект или массив, или значение, если метод возвращает примитив.
    -  Методы могут быть асинхронными или возвращать Observable (nest подпишется и примет последнее значение в потоке):

       ```typescript
       import { Controller, Get } from '@nest/common';
       import { Observable, of } from 'rxjs';
       
       @Controller()
       export class MyController {
          @Get()
          async method1(): Promise<any[]> {
             return [];
          }
       
          @Get()
          method2(): Observable<any[]> {
             return of([]);
          }
       }
       ```

2. Дескриптор метода @HttpCode(code: number) позволяет задать код ответа для метода.
3. Дескриптор метода @Header(name, value) позволяет задать заголовок ответа для метода.
4. Дескриптор метода @Redirect(url?, statusCode?) позволяет задавать перенаправление запроса. statusCode по
   умолчанию 302. Для динамического переопределения адреса и кода перенаправления метод может возвращать объект вида:

   ```
   {
      "url": string,
      "statusCode": number
   }
   ```

### Дескрипторы параметров методов контроллера

Дескрипторы параметров, предоставляющие доступ к объектам запроса и ответа, и соответствующие им сущности express:
1. @Req (@Request) - req;
2. @Res (@Response) - res - позволяет напрямую отправлять ответ, код и т.д.;
3. @Next - next;
4. @Session() - req.session;
5. @Param(key?: string) - req.params / req.params[key];
6. @Body(key?: string) - req.body / req.body[key];
7. @Query(key?: string) - req.query / req.query[key];
8. @Headers(name?: string) - req.headers / req.headers[name],
9. @Ip() - req.ip;
10. @HostParam() - req.hosts.

Кастомные декораторы параметров методов контроллера создаются с помощью функции 
createParamDecorator((data, ctx) => any), data - значение опционального аргумента, передаваемого в декоратор, ctx - 
текущий ExecutionContext:

```typescript
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const User = createParamDecorator(
   (data: unknown, ctx: ExecutionContext) => {
      const request = ctx.switchToHttp().getRequest();
      const user = request.user;
      
      return data ? user?.[data] : user;
   },
);

@Controller()
export class MyController {
   @Get()
   method(@User('id') id: string): string {
      return `response for ${id}`;
   }
}
```

Декораторы, объединяющие в себе композицию декораторов, создаются с помощью функции applyDecorators:

```typescript
import { applyDecorators } from '@nestjs/common';

export function Auth(...roles: Role[]) {
  return applyDecorators(
    SetMetadata('roles', roles),
    UseGuards(AuthGuard, RolesGuard),
    ApiBearerAuth(),
    ApiUnauthorizedResponse({ description: 'Unauthorized' }),
  );
}


@Controller()
export class MyController {
    @Get()
    @Auth('admin')
    method() {}
}
```