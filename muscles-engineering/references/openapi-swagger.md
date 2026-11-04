# OpenAPI и Swagger UI в Muscles ASGI

Применяй при разработке или исследовании документации HTTP API. Описано наблюдаемое поведение локальных библиотек Muscles и Muscles ASGI; в целевом проекте сначала проверь версию, код и тесты. Отличай OpenAPI JSON — контракт API — от Swagger UI, который этот документ отображает и отправляет запросы.

## Источники и генерация

- `muscles-asgi/src/muscles/asgi/restful/restful.py` — конфигурация `RestApi`, маршруты схемы и регистрация операций.
- `muscles-asgi/src/muscles/asgi/restful/swagger.py` — `Swagger.dump()`, paths, модели и security schemes.
- `muscles/src/muscles/core/schema/` — преобразование схем и метаданные.
- Документация и тесты пакета `muscles-asgi` (`docs/openapi-and-routing.md`, `docs/swagger.md`, `tests/restful/`) — примеры и проверки.

`RestApi` регистрирует обработчики; `Swagger.dump()` строит документ по их маршрутам и метаданным. Он не выводит полный контракт из тела Python-функции. Модели из request/response добавляются в `components.schemas` и используются через `$ref`; схемы безопасности собираются в `components.securitySchemes`. Отдельно проверяй реальную обработку запросов и защиту.

## Общие настройки и адреса

`RestApi` принимает `prefix`, `name`, `title`, `version`, `openapi_version`, `description`, `termsOfService`, `contact_email`, `servers`, а также настройки путей документации. Версия API и версия OpenAPI различны; по умолчанию OpenAPI — 3.0.3. Для `prefix='/api/v1'` документированные пути: UI `/api/v1/docs`, `/api/v1/swagger`; JSON `/api/v1/openapi.json`, `/api/v1/schema`. Учитывай фактические настройки прокси.

В локальном эксперименте при `prefix='/api/v1'` одновременно сформированы `servers: [{url: '/api/v1'}]` и paths с `/api/v1`. Проверь URL, который Swagger UI или клиент формирует при запросе: префикс может продублироваться. При необходимости явно настрой `servers` с учётом развёртывания. Шаблон Swagger UI этой версии загружает CSS и JS с внешнего CDN; доступность JSON не доказывает доступности интерфейса.

## Операции и параметры

`api.init` принимает `tags`, `summary`, `description`, `parameters`, `request`, `response`, `security` и `auth` в дополнение к пути, методу и ключу. Генератор создаёт операции в `paths`, приводит методы к нижнему регистру; без описанных ответов подставляет `default`. Не предполагай, что `key` автоматически становится `operationId`.

Для параметров используй `PathParameter`, `QueryParameter`, `HeaderParameter`, `CookieParameter` с именем, типом, `required`, `description` и при необходимости `explode`; `QueryParameter` поддерживает `multiple`. Если `{id}` в пути не описан, генератор добавляет обязательный строковый параметр; явно задавай нужный тип. `FormParameter` в этой реализации формирует `in: formData`: для OpenAPI 3 проверяй валидность и рассматривай form `requestBody` по возможностям версии.

## Запросы, ответы и модели

Указывай тело как список, например `request=[JsonRequestBody(description='Данные', model=CreateProductRequest)]`, а ответы как словарь кодов, например `response={200: JsonResponseBody(description='OK', model=ProductResponse), 404: JsonResponseBody(description='Не найден')}`. Для массивов request/response поддерживают `is_list`, `min_items`, `max_items`, `unique_items`; у одного кода ответа может быть список разных типов содержимого. Сверяй метаданные с реальным статусом, телом и Content-Type.

В локальном генераторе `requestBody.required` не формируется. В проверенном примере `Column(String, nullable=False, required=True, description=..., example=..., min_length=2)` итоговая схема содержала только `type: string`. Не обещай автоматический перенос `required`, описаний, примеров, default и ограничений: проверь полученный JSON, особенно вложенные модели, enum, массивы и `$ref`. Само описание схемы не гарантирует runtime-валидацию. Для подробностей поведения `Model` и `Column` читай [muscles-models.md](muscles-models.md).

## Группы и безопасность

`api.group` может наследовать общий префикс, `tags`, `security` и ответы операциям. Тесты демонстрируют `BearerAuthSecurity()` и общий `401`; `auth=False` на операции отключает унаследованную защиту в проверенном сценарии. Учитывай влияние изменений группы на все операции. Проверяй как сформированные security requirements и security schemes, так и реальные запросы с отсутствующими, неверными и допустимыми учётными данными: значок замка не подтверждает защиту сервера.

## Проверка результата

Проверяй сериализованный OpenAPI на корректность через существующий в проекте валидатор, наличие paths и методов, schemas и `$ref`, параметры, request body, статусы, ошибки, безопасность, фактический URL сервера. Сверяй контракт с HTTP-поведением; при доступном приложении проверь Swagger UI и формируемый им запрос в браузере. Не утверждай, что UI или сервер проверены, если исследован только `Swagger.dump()` или код. После изменений контракта учитывай потребителей API.
