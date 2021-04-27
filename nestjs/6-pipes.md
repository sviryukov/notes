# Pipes

Pipe - класс, используемый для трансформации и валидации аргументов методов контроллера. Pipe обрабатывает аргументы
перед тем, как вызывается метод, и при необходимости выбрасывает исключение, отлавливаемое фильтром исключений
(встроенным или любым другим).

-  К pipe должен быть применен декоратор @Injectable().
-  Pipe должен реализовывать интерфейс PipeTransform<T, R> и его метод transform(value, metadata), T - тип значения
   аргумента, R - тип возвращаемого transform значения, value - значение аргумента, metadata - объект, реализующий
   интерфейс ArgumentMetadata. ArgumentMetadata имеет три поля: type - вариант передаваемого параметра, metatype - 
   тип передаваемого параметра, data - аргумент, передаваемый в декоратор аргумента (например, для @Body('string') 
   data будет 'string').
   
   ```typescript
   interface ArgumentMetadata {
      type: 'body' | 'query' | 'param' | 'custom';
      metatype?: Type<unknown>;
      data?: string;
   }
   ```
   
-  Кастомные декораторы параметров могут использоваться вместе с pipes так же, как встроенные.

```typescript
import { PipeTransform, Injectable, ArgumentMetadata, Controller, Get, Param, UsePipes } from '@nestjs/common';

@Injectable()
export class MyPipe implements PipeTransform<string, string> {
   transform(value: string, metadata: ArgumentMetadata) {
      return value;
   }
}

@Controller()
export class MyController {
   @Get()
   method1(@Param('param', MyPipe) param: string) {
      // ...
   }
   
   @Post()
   @UsePipes(MyPipe)
   method2(@Param('param') param: string) {
      // ...
   }
}
```

Pipe может быть использован 3-мя способами:
1. Передача аргументов в декоратор аргумента метода контроллера (после первого аргумента декоратора при его наличии):
   ```typescript
   @Param('param', ...pipes)
   ```
   pipes - классы pipe или их экземпляры.
2. Применение декоратора @UsePipes(...pipes) к методу контроллера, pipes - классы pipe или их экземпляры.
3. Глобальный pipe - применяется ко всем контроллерам и их методам в приложении:
   ```typescript
   const app = await NestFactory.create(AppModule);
   app.useGlobalPipes(new ValidationPipe());
   ```

### Встроенные pipe

В nest есть 6 встроенных pipe:
1. ValidationPipe;
2. ParseIntPipe;
3. ParseBoolPipe;
4. ParseArrayPipe;
5. ParseUUIDPipe;
6. DefaultValuePipe.

Parse... pipe проверяют тип значения аргумента (или приводимость к определенному типу), возвращают значение данного
типа или выбрасывают исключение. С помощью DefaultValuePipe можно определить значение аргумента по умолчанию в том
случае, когда он равен null или undefined.

```typescript
import { PipeTransform, ParseIntPipe, HttpStatus, DefaultValuePipe } from '@nestjs/common';

@Controller()
export class MyController {
   @Get()
   method1(@Param('param', new DefaultValuePipe(0), ParseIntPipe) param: string) {
      // ...
   }
   
   @Get()
   method2(@Param('param', new ParseIntPipe({ errorHttpStatusCode: HttpStatus.NOT_ACCEPTABLE })) param: string) {
      // ...
   }
}
```

В первом случае при ошибке валидации будет выброшено такое исключение:

```json
{
  "statusCode": 400,
  "message": "Validation failed (numeric string is expected)",
  "error": "Bad Request"
}
```

Во втором случае переопределен код ошибки для выбрасываемого исключения:

```json
{
   "statusCode": 406,
   "message": "Validation failed (numeric string is expected)",
   "error": "Bad Request"
}
```

### Пример валидации тела post-запроса с помощью библиотеки joi

```
$ npm install --save joi
$ npm install --save-dev @types/joi
```

```typescript
import {
    PipeTransform, 
    ParseIntPipe,
    HttpStatus,
    Post,
    Body,
    Injectable,
    ArgumentMetadata,
    BadRequestException,
    UsePipes
} from '@nestjs/common';
import { ObjectSchema } from 'joi';
import { MyDto, myCreateSchema } from '...';

@Injectable()
export class JoiValidationPipe implements PipeTransform {
   constructor(private schema: ObjectSchema) {}

   transform(value: any, metadata: ArgumentMetadata) {
      const { error } = this.schema.validate(value);
      if (error) {
         throw new BadRequestException('Validation failed');
      }
      return value;
   }
}

@Controller()
export class MyController {
   @Post()
   @UsePipes(new JoiValidationPipe(myCreateSchema))
   async method(@Body() myDto: MyDto) {
      // ...
   }
}
```

myCreateSchema - функция, возвращающая объект ObjectSchema.

### Пример валидации класса тела post-запроса с помощью библиотеки class-validator

```
$ npm i --save class-validator class-transformer
```

```typescript
import {
    PipeTransform, 
    ParseIntPipe,
    HttpStatus,
    Post,
    Body,
    Injectable,
    ArgumentMetadata,
    BadRequestException,
    UsePipes
} from '@nestjs/common';
import { IsString, IsInt, validate } from 'class-validator';
import { plainToClass } from 'class-transformer';

export class MyDto {
   @IsString()
   name: string;

   @IsInt()
   age: number;
}

@Injectable()
export class ValidationPipe implements PipeTransform<any> {
   async transform(value: any, { metatype }: ArgumentMetadata) {
      if (!metatype || !this.toValidate(metatype)) {
         return value;
      }
      const object = plainToClass(metatype, value);
      const errors = await validate(object);
      if (errors.length > 0) {
         throw new BadRequestException('Validation failed');
      }
      return value;
   }

   private toValidate(metatype: Function): boolean {
      const types: Function[] = [String, Boolean, Number, Array, Object];
      return !types.includes(metatype);
   }
}

@Controller()
export class MyController {
   @Post()
   @UsePipes(new JoiValidationPipe(myCreateSchema))
   async method(@Body(new ValidationPipe()) myDto: MyDto) {
      // ...
   }
}
```