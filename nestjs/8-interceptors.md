# Interceptors

Interceptor - класс, позволяющий оборачивать методы для обработки запросов в дополнительную логику.
-  К interceptor должен быть применен декоратор @Injectable().
-  Interceptor должен реализовывать интерфейс NestInterceptor<T, R> и его метод intercept(context, next), T - тип 
   значения, возвращаемого методом, к которому применен interceptor, R - результат выполнения intercept (intercept 
   возвращает Observable\<R\>), context - текущий ExecutionContext, next - объект, реализующий интерфейс CallHandler с
   единственным методом handle. next.handle() вызывает метод, к которому применен interceptor, и возвращает Observable. 
   Этот Observable возвращает значение, которое вернул исходный метод после его вызова с помощью next.handle(). 
   
```typescript
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';

@Injectable()
export class MyInterceptor implements NestInterceptor {
   intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
      return next.handle();
   }
}
```

Interceptor может быть использован 3-мя способами:
1. Применение декоратора @UseInterceptors(...interceptors) (interceptors - классы или экземпляры interceptor-ов) к 
   классу контроллера.
2. Применение декоратора @UseInterceptors(...interceptors) к методу контроллера.
3. Глобальный interceptor - применяется ко всем контроллерам и их методам в приложении:
   ```typescript
   const app = await NestFactory.create(AppModule);
   app.useGlobalInterceptors(new LoggingInterceptor());
   ```

### Применение interceptor

1. Добавление дополнительной логики до и/или после вызова метода:
   
   ```typescript
   import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
   import { Observable } from 'rxjs';
   import { tap } from 'rxjs/operators';
   
   @Injectable()
   export class LoggingInterceptor implements NestInterceptor {
      intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
         console.log('Before...');
         const now = Date.now();
         return next
            .handle()
            .pipe(
               tap(() => console.log(`After... ${Date.now() - now}ms`)),
            );
      }
   }
   ```
   
2. Трансформация результата вызова метода или исключения, выброшенного методом:
   
   ```typescript
   import { Injectable, NestInterceptor, ExecutionContext, CallHandler, BadGatewayException } from '@nestjs/common';
   import { Observable, throwError } from 'rxjs';
   import { map, catchError } from 'rxjs/operators';
    
   export interface Response<T> {
      data: T;
   }
   
   @Injectable()
   export class TransformInterceptor<T> implements NestInterceptor<T, Response<T>> {
      intercept(context: ExecutionContext, next: CallHandler): Observable<Response<T>> {
         return next.handle().pipe(map(data => ({ data })));
      }
   }
   
   @Injectable()
   export class ExcludeNullInterceptor implements NestInterceptor {
      intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
         return next
            .handle()
            .pipe(map(value => value === null ? '' : value ));
      }
   }
    
   @Injectable()
   export class ErrorsInterceptor implements NestInterceptor {
      intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
         return next
            .handle()
            .pipe(
               catchError(err => throwError(new BadGatewayException())),
            );
      } 
   }
   ```

3. Расширение или полное переопределение изначального поведения метода:
   
   ```typescript
   import { Injectable, NestInterceptor, ExecutionContext, CallHandler, RequestTimeoutException } from '@nestjs/common';
   import { Observable, of, throwError, TimeoutError } from 'rxjs';
   import { catchError, timeout } from 'rxjs/operators';
   
   @Injectable()
   export class TimeoutInterceptor implements NestInterceptor {
      intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
         return next.handle().pipe(
            timeout(5000),
            catchError(err => {
               if (err instanceof TimeoutError) {
                  return throwError(new RequestTimeoutException());
               }
               return throwError(err);
            }),
           );
      };
   };
    
   @Injectable()
   export class CacheInterceptor implements NestInterceptor {
      intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
         // computing isCached
         if (isCached) {
            return of([]);
         }
         return next.handle();
      }
   }
   ```