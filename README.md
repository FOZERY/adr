# Переход с AjvSchema<T> на типы схем OpenApi (SchemaObject из @nestjs/swagger)

## Контекст

На данный момент JsonSchemaType<T>, которой по сути является AjvSchema<T> некорекктно из коробки выводит тип схемы из переданного в дженерик типа TypeScript'а.  
Проблеме уже несколько лет, существует несколько до сих пор актуальных issues на GitHub:

1. https://github.com/ajv-validator/ajv/issues/2163 - <T | null> не определяется в схеме как nullable: true - возникает ошибка.
2. https://github.com/ajv-validator/ajv/issues/1375 - <T | undefined> должен определяться в схеме, как nullable: true, иначе возникает ошибка.

Данные проблемы абсолютно не были решены в Ajv. Проблема не в определении схем, проблема конкретно в выводе типов из TypeScript'а.  
Эти два несоответствия кардинально ошибочны в определении типов JavaScript и TypeScript, ведь null !== undefined.  
Опциональные поля должны были значить то, что мы их вообще можем не ожидать (т.е. их отсутствие в required: []), а поля с типом null - конкретно ожидаем значение равное null, что должно было бы соответствовать nullable: true.

Причем "строгое" определение типов схемы можно обойти, просто объявив поля в $defs:

1. AjvSchema<T> ругается на nullable:

```
type OptionalFields = {
  offset?: number;
  limit?: number;
};

// AjvSchema ожидает, что опциональные поля undefined, будут указаны как null
const optionalFieldsSchema: AjvSchema<OptionalFields> = {
  type: 'object',
  additionalProperties: false,
  properties: {
    offset: {
      type: 'integer',
    },
    limit: {
      type: 'integer',
    },
  },
}; // ...Property 'nullable' is missing in type '{ type: "integer"; }' but required in type '{ nullable: true; const?: null | undefined; enum?: readonly (number | null | undefined)[] | undefined; default?: number | null | undefined; }'.
```

2. Обход типизации AjvSchema<T>, через $defs:

```
type OptionalFields = {
  offset?: number;
  limit?: number;
};

const optionalFieldsSchema: AjvSchema<OptionalFields> = {
  type: 'object',
  additionalProperties: false,
  $defs: {
    offset: {
      type: 'integer',
    },
    limit: {
      type: 'integer',
    },
  },
  properties: {
    offset: {
      $ref: '#/$defs/offset',
    },
    limit: {
      $ref: '#/$defs/limit',
    },
  },
};
```

Причём, если передать в дженерик типы с null, то AjvSchema это проглотит и никак не будет ругаться - наоборот, он выдаст ошибку, если мы укажем nullable: true:

```
type OptionalFields = {
  offset: number | null;
  limit: number | null;
};

const optionalFieldsSchema: AjvSchema<OptionalFields> = {
  type: 'object',
  additionalProperties: false,
  required: ['offset', 'limit'],
  properties: {
    offset: {
      type: 'integer',
      nullable: true,
    },
    limit: {
      type: 'integer',
      nullable: true,
    },
  },
}; // ...Type '{ type: "integer"; nullable: true; }' is not assignable to type '{ nullable?: false | undefined; const?: number | null | undefined; enum?: readonly (number | null)[] | undefined; default?: number | null | undefined; }'.
Types of property 'nullable' are incompatible.
Type 'true' is not assignable to type 'false'.
```

Из-за этого всего приходится использовать "костыли":

1. Либо использовать $defs и тогда мы теряем всю типизацию, кроме проверки присутствия полей
2. Либо кастить к нужному типу через as

Также есть проблемы с правильным определением типов при использовании Union Types. Об этом также сказано и в документации Ajv: https://ajv.js.org/guide/typescript.html#type-safe-unions. В этом случае мы также чаще всего теряем правильную типизацию схемы.

Учитывая всё вышесказанное напрашивается решение проблемы.

## Варианты решения

### 1. Использовать каст типов через as и $defs

Плюсы:

1. Мы не меняем кардинально текущие схемы
2. Сможем обойти ошибки, через $defs и as any/as false/as true

Минусы:

1. Мы полностью теряем типизацию
2. Используя as мы можем привести тип к any и потерять не только типизацию, но и определить её неправильно, что будет ещё хуже.
3. Мы продолжаем использовать AjvSchema<T>, но пытаемся обойти её всевозможными способами. Типы теряются, допустить ошибку в схеме становится намного проще.
4. AjvSchema не маппится 1 в 1 на схему OpenApi 3.0, поэтому, используя, например, if/then/else мы не сможем использовать эту схему в Swagger.

### 2. Отказаться от AjvSchema<T> и перейти на типы схем для OpenApi

Плюсы:

1. Мы получаем полную типизацию схемы OpenApi (properties, type, description, example и т.д.). Невозможно будет пропустить какое-то поле, нужное для определения самой схемы.
2. Мы получаем возможность **полностью** отказаться от классов DTO из NestJS, дублирующих sdk-types. Одна схема будет использоваться и для AJV-валидации и для Swagger. Мы избегаем неявного дублирования, ошибок, когда кто-то поменял sdk-тип и забыл поменять DTO для Swagger.
3. Мы избегаем проблем с реализацией сложных типов в классах DTO для Swagger, когда нам нужно прокидывать дженерики в DTO и составлять сложные декораторы.
4. Получаем возможность кодогенерации типов TypeScript из OpenApi-схем (если такое вообще понадобится, ибо здесь также могут возникнуть проблемы с неправильным выводом типов).

Минусы:

1. Тип TypeScript не будет маппиться на схему OpenAPI. Схема становится первичным источником правды, под неё пишется тип.
2. Мы не сможем использовать if/then/else и другие фишки Ajv и JsonSchema, которых нет в OpenApi (но их и не так много).

Дополнение:

1. Мы завязываемся на схемы, вместо типов TypeScript.
2. Схемы начинают выступать единственным источником правды. На основе схем мы составляем типы для TypeScript.
3. Схемы с описанием для валидации и описанием для Swagger видны как фронтенду, так и бэкенду (находятся в sdk-types).
4. На каждую схему обязательно придется реализовывать модульные тесты для строгого контроля валидации и ожидаемых полей.

Появляется возможность реализовать декоратор @RouteSchema() со схемой, как в fastify:

```
  @Post('step1')
  @RouteSchema({
    body: spacesStep1CreateRequestSchema,
    response: {
      '2XX': {
        type: 'string',
        description: 'Success',
      },
      '4XX': {
        description: 'Bad Request',
        type: 'object',
        properties: {
          success: { type: 'boolean' },
        },
        example: {
          success: false,
        },
      },
    },
  })
  public async step1Create(
    @AjvBody(spacesStep1CreateRequestSchema) request: SpacesStep1CreateRequest,
  ): Promise<SpacesStep1CreateResponse> {
    const result = await this.spacesService.step1Create(request);
    return apiResponse(result);
  }
```

Причем, мы в одном месте (sdk-types) создаём и типы и схемы для body, query, params, response и затем используем их в декораторе. Количество шаблонного кода сокращается в несколько раз, ибо не нужно писать классы DTO, схемы становятся единным источником правды.
