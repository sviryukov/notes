# Guards

Guard - класс, используемый для определения, должен ли метод обработать запрос. В отличие от middleware, guard имеет 
доступ к ExecutionContext - интерфейсу, расширяющему ArgumentHost.
-  К guard должен быть применен декоратор @Injectable().
-  Guard должен реализовывать интерфейс CanActivate и его метод canActivate(context), context - текущий
   ExecutionContext.
-  Guard возвращает boolean-значение (или Promise\<boolean\>, или Observable\<boolean\>), обозначающее, должен ли метод
   обработать запрос.

```typescript
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Observable } from 'rxjs';
import { validateRequest } from '...';

@Injectable()
export class AuthGuard implements CanActivate {
   canActivate(
      context: ExecutionContext,
   ): boolean | Promise<boolean> | Observable<boolean> {
      const request = context.switchToHttp().getRequest();
      return validateRequest(request);
   }
}
```

Порядок выполнения операций перед выполнением метода обработки запроса:
1. Middleware;
2. Guard;
3. Interceptor, Pipe.

Guard может быть использован 3-мя способами:
   1. Применение декоратора @UseGuards(...guards) (guards - классы или экземпляры guard-ов) к классу контроллера.
   2. Применение декоратора @UseGuards(...guards) к методу контроллера.
   3. Глобальный guard - применяется ко всем контроллерам и их методам в приложении:
   ```typescript
   const app = await NestFactory.create(AppModule);
   app.useGlobalGuards(new RolesGuard());
   ```