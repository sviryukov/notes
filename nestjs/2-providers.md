# Провайдеры

@Injectable() - дескриптор класса, использующийся для создания провайдеров. Провайдер — обычный TS-класс, который
вместе с другим провайдером или контроллером реализует паттерн внедрения зависимостей (Dependency injection): один класс
содержит в себе экземпляр другого класса и, таким образом, зависит от его реализации. Провайдеры внедряются в качестве
зависимостей в другие провайдеры или контроллеры. Контроллеры должны делегировать провайдерам все задачи, выходящие за
рамки обработки HTTP-запросов.

```typescript
import { Injectable, Controller } from '@nest/common';

@Injectable()
export class MyService1 {}

@Injectable()
export class MyService2 {
   constructor(private myService1: MyService1) {}
}

@Controller('route')
export class MyController {
   constructor(private myService2: MyService2) {}
}
```