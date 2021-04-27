# Исключения

В nest есть встроенный слой для обработки всех необработанных в приложении исключений. По умолчанию это выполняется
глобальным фильтром исключений, обрабатывающим исключения, реализующие класс HttpException или его дочерние классы.
В случае, если исключение не реализует HttpException или его дочерний класс, встроенный фильтр исключений вернет в
ответ на запрос следующий JSON:

```json
{
  "statusCode": 500,
  "message": "Internal server error"
}
```

### Классы исключений

-  HttpException(response, status) - основной класс исключений в nest. response - строка (если необходимо определить
   только текст ошибки) или объект с полями status и error (если необходимо переопределить и статус), status - код
   HTTP-ответа (лучше использовать enum HttpStatus).

   ```typescript
   import { Controller, Get, HttpException, HttpStatus } from '@nest/common';
   
   @Controller()
   export class MyController {
      @Get()
      method1() {
         throw new HttpException('Forbidden', HttpStatus.FORBIDDEN);
      }
      
      @Get()
      method2() {
         throw new HttpException({ 
            status: HttpStatus.FORBIDDEN,
            error: 'Custom error message'
         }, HttpStatus.FORBIDDEN);
      }
   }
   ```
-  Можно также создавать кастомные классы исключений:

   ```typescript
   import { Controller, Get, HttpException, HttpStatus } from '@nest/common';
   
   export class ForbiddenException extends HttpException {
      constructor() {
         super('Forbidden', HttpStatus.FORBIDDEN);
      }
   }
   
   @Controller()
   export class MyController {
      @Get()
      method1() {
         throw new HttpException('Forbidden', HttpStatus.FORBIDDEN);
      }
      
      @Get()
      method2() {
         throw new ForbiddenException();
      }
   }
   ```

-  В nest есть классы исключений для основных 4xx и 5xx HTTP-ошибок (BadRequestException, UnauthorizedException и т.д).

### Фильтры исключений

Фильтр исключений — класс, позволяющий переопределять обработку исключений, производимую встроенным глобальным фильтром.
-  К фильтру исключений должен быть применен декоратор @Catch(...exceptions), exceptions - классы исключений,
   обрабатываемых данным фильтром. Если extensions не указаны, фильтр будет обрабатывать все исключения.
-  Фильтр исключений должен реализовывать интерфейс ExceptionFilter и его метод catch(exception, host), exception -
   объект обрабатываемого исключения, host - объект интерфейса ArgumentHost, в данном случае предоставляющий доступ к
   объектам Request и Response запроса, при обработке которого было выброшено исключение.

```typescript
import { ExceptionFilter, Catch, ArgumentsHost, HttpException } from '@nestjs/common';
import { BaseExceptionFilter } from '@nestjs/core';
import { Request, Response } from 'express';

@Catch(HttpException)
export class MyExceptionFilter implements ExceptionFilter {
   catch(exception: HttpException, host: ArgumentsHost) {
      const ctx = host.switchToHttp();
      const response = ctx.getResponse<Response>();
      const request = ctx.getRequest<Request>();
      const status = exception.getStatus();
      
      response
         .status(status)
         .json({
           statusCode: status,
           timestamp: new Date().toISOString(),
           path: request.url,
      });
   }
}
```

Фильтры исключений могут быть использованы в приложении 3 способами:

1. Применение декоратора @UseFilters(...filters) (filters - классы или экземпляры фильтров исключений) к классу 
   контроллера.
2. Применение декоратора @UseFilters(...filters) к методу контроллера.
3. Глобальные фильтры исключений обрабатывают все исключения, выбрасываемые в приложении:
   ```typescript
   const app = await NestFactory.create(AppModule);
   app.useGlobalFilters(new MyExceptionFilter());
   ```