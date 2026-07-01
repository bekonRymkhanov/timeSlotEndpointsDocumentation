# Маршрутизация через Yandex MVRP — документация

> Ручное (по запросу диспетчера) построение маршрутов через Yandex Routing (MVRP).
> Диспетчер сам выбирает машины и заказы (заборы/доставки) на дату — система строит
> задачу для Yandex, отправляет её, а готовый результат забирает по расписанию (крон).
>
> Модуль: `app/Module/Routing/`. Маршруты: `routes/routing.php`.
> Это ручной аналог старых плановых команд (`CreateMVRPYandexRoutingCommand`,
> `CheckMVRPRoutingStatusCommand`), которые работали автоматически по секторам.

---

## 1. Что это такое (бизнес-описание)

Диспетчеру нужно распределить набор заказов на день по конкретным машинам так,
чтобы маршрут каждой машины был оптимальным (минимум пробега, с учётом
грузоподъёмности, объёма и рабочего времени). Расчёт оптимального маршрута мы
не делаем сами — его выполняет **Yandex Routing (сервис MVRP — Multiple Vehicles
Routing Problem)**.

Процесс состоит из двух шагов, разнесённых во времени:

1. **Отправка задачи.** Диспетчер выбирает машины + заказы + дату и нажимает
   «Построить маршрут». Мы собираем данные, сохраняем «черновик» маршрута в базе
   и **отправляем задачу в Yandex**. Yandex сразу отвечает не результатом, а только
   **идентификатором задачи** (`task_id`) — расчёт у него идёт в фоне.
2. **Получение результата.** Каждые 5 минут крон опрашивает Yandex по `task_id`.
   Как только Yandex посчитал — мы разбираем ответ, раскладываем точки по машинам
   в порядке объезда, помечаем невписавшиеся заказы и выставляем маршруту статус
   «Готов».

Из-за этого фронтенд после отправки **опрашивает** маршрут (эндпоинт `show`), пока
статус не станет `2` (готов) или `3` (ошибка).

### Статусы маршрута

| Код | Значение | Когда выставляется |
|-----|----------|--------------------|
| `1` | В процессе (`PENDING`) | Сразу после создания и отправки в Yandex |
| `2` | Готов (`SUCCESS`) | Когда крон получил и применил результат |
| `3` | Ошибка (`FAILED`) | Если Yandex не вернул `task_id` при отправке |

---

## 2. Общая схема потока

```
   Диспетчер (фронтенд)
        │
        │ 1. POST /api/routing/plan   { cars[], orderTakeIds[], deliveryIds[], date }
        ▼
 ┌──────────────────────────────────────────────────────────────┐
 │ RoutePlanController::create                                    │
 │   → проверка прав + валидация (CreateRoutePlanRequest)         │
 │   → CreateRoutePlanCommand → CreateRoutePlanHandler            │
 │        • загрузка машин / заборов / доставок                   │
 │        • определение склада-депо по city_id                    │
 │        • проверки (координаты, вес, is_routing)                │
 │        • сохранение Route + CarRoute[] + RouteClient[] (PENDING)│
 │        • сборка тела запроса (RoutePlanRequestBuilder)         │
 │        • ОТПРАВКА в Yandex (POST /vrs/api/v1/add/mvrp)         │
 │        • сохранение task_id (или статус FAILED)               │
 └──────────────────────────────────────────────────────────────┘
        │
        │ 2. Ответ 201  { id, status: 1, date, taskId }   ← результат ещё НЕ готов
        ▼
   Фронтенд периодически опрашивает show…

 ─ ─ ─ ─ ─ ─ ─ ─ ─ (проходит время, работает крон каждые 5 мин) ─ ─ ─ ─ ─ ─ ─ ─

 ┌──────────────────────────────────────────────────────────────┐
 │ CheckRoutePlanStatusCommand  (route-plan:check-status)        │
 │   → для каждого PENDING-маршрута с task_id:                    │
 │        • GET /vrs/api/v1/result/mvrp/{task_id}                │
 │        • если HTTP 200 — результат готов                       │
 │        • RoutePlanResultDTO разбирает ответ                    │
 │        • RoutePlanResultApplier раскладывает точки по машинам, │
 │          помечает невписавшиеся, считает итоги, статус SUCCESS │
 └──────────────────────────────────────────────────────────────┘
        │
        │ 3. GET /api/routing/plan/{routeId}
        ▼
 ┌──────────────────────────────────────────────────────────────┐
 │ RoutePlanController::show                                      │
 │   → читает уже сохранённый результат из БД (Yandex не зовёт)   │
 │   → RoutePlanResource: машины со списком точек + unrouted[]    │
 └──────────────────────────────────────────────────────────────┘
```

**Важно:** отправка в Yandex происходит **синхронно, внутри HTTP-запроса** `create`
(не в очереди). Поэтому медленный ответ Yandex замедляет ответ пользователю. Маршрут
сохраняется в БД **до** отправки, поэтому неудачная отправка оставляет строку
маршрута со статусом `FAILED` (повторная отправка пока не реализована).

---

## 3. Эндпоинты

### 3.1. `POST /api/routing/plan` — создать (отправить на расчёт)

- **Контроллер:** `RoutePlanController::create`
- **Право:** `dispatcher.routing.plan.store` (`PermissionList::ROUTING_PLAN_STORE`)
- **Валидация:** `CreateRoutePlanRequest`
- **Ответ:** `201`, сообщение «Маршрут поставлен в очередь на расчёт»

**Тело запроса**

| Поле               | Тип           | Правила |
|--------------------|---------------|---------|
| `cars`             | массив (≥1)   | обязательно |
| `cars[].id`        | int           | обязательно, без повторов, `exists:cars,id` |
| `cars[].workStart` | строка/null   | время `H:i` или `H:i:s`, необязательно (переопределяет рабочие часы машины) |
| `cars[].workEnd`   | строка/null   | то же |
| `orderTakeIds`     | int[]         | обязательно, если нет `deliveryIds`; без повторов; `exists:order_takes,id` |
| `deliveryIds`      | int[]         | обязательно, если нет `orderTakeIds`; без повторов; `exists:deliveries,id` |
| `date`             | дата          | обязательно, не раньше сегодня (`after_or_equal:today`) |

Пример:

```json
{
  "cars": [
    { "id": 1, "workStart": "09:00:00", "workEnd": "18:00:00" }
  ],
  "orderTakeIds": [10, 11],
  "deliveryIds": [20, 21],
  "date": "2026-06-24"
}
```

**Ответ `201` (`RoutePlanCreatedResource`)**

```json
{ "id": 12, "status": 1, "date": "2026-06-24", "taskId": "abcd-...-1234" }
```

`taskId` равен `null` только если отправка в Yandex не удалась (тогда `status = 3`).

### 3.2. `GET /api/routing/plan/{routeId}` — получить результат

- **Контроллер:** `RoutePlanController::show`
- **Право:** `dispatcher.routing.plan.show` (`PermissionList::ROUTING_PLAN_SHOW`)
- **Только число** в `{routeId}` (`whereNumber`); неизвестный id → `404`.
- **Читает только базу** — Yandex не вызывается. Пока крон не отработал, будет
  `status = 1` с пустыми `stops` и пустым `unrouted`.

**Ответ `200` (`RoutePlanResource`)**

```json
{
  "id": 12,
  "date": "2026-06-24",
  "status": 2,
  "taskId": "abcd-...-1234",
  "error": null,
  "cars": [
    {
      "carRouteId": 5, "carId": 1, "carNumber": "123ABC01",
      "driver": { "id": 7, "name": "...", "phone": "..." },
      "workStart": "08:00:00", "workEnd": "19:00:00",
      "totalWeight": 540.0, "totalVolume": 3.2, "stopsCount": 6,
      "stops": [ /* точки в порядке объезда */ ]
    }
  ],
  "unrouted": [ /* заказы, которые не удалось включить в маршрут */ ]
}
```

- `stops` — точки одной машины, **уже отсортированы по `position`** (порядок объезда).
- `unrouted` — заказы со статусом «не вписан» (`DROPPED`), с причиной `dropReason`.

**Одна точка (`RouteClientResource`):**

| Поле | Значение |
|------|----------|
| `id` | ID точки маршрута (`route_clients.id`) |
| `type` | `1` — забор, `2` — доставка |
| `clientId` | ID забора или доставки |
| `position` | Порядок объезда |
| `status` | `0` — в ожидании, `1` — в маршруте, `2` — не вписан |
| `dropReason` | Причина, если не вписан |
| `latitude`, `longitude` | Координаты точки |
| `weight`, `volume` | Вес и объём |
| `arrivalAt` | Расчётное время прибытия (от Yandex) |

---

## 4. Классы и их роли

### Слой HTTP (вход/выход)

| Класс | Роль |
|-------|------|
| `Controllers/RoutePlanController` | Тонкий контроллер. Проверяет права, валидирует, диспатчит команду или зовёт query, оборачивает результат в Resource. Логики не содержит. |
| `Requests/CreateRoutePlanRequest` | Валидация тела запроса (правила выше). Отдаёт готовый DTO через `getDTO()`. |
| `Resources/RoutePlanCreatedResource` | Формат ответа на `create`: `id`, `status`, `date`, `taskId`. |
| `Resources/RoutePlanResource` | Формат ответа на `show`: маршрут, машины со списком точек, `unrouted[]`. |
| `Resources/RouteClientResource` | Формат одной точки маршрута. |

### Слой команды (запись, CQRS)

| Класс | Роль |
|-------|------|
| `DTO/CreateRoutePlanDTO` | Неизменяемый набор данных из запроса. `fromRequest` «разворачивает» `cars` в `carIds` + `carWorkHours[carId] = {start,end}`, нормализует время (`08:00` → `08:00:00`) и запоминает `userId = Auth::id()`. |
| `Commands/CreateRoutePlanCommand` | Команда-обёртка над DTO (объект-сообщение для шины команд). |
| `Handlers/CreateRoutePlanHandler` | **Главный обработчик.** Загружает сущности, определяет депо, валидирует, сохраняет агрегат `Route`+`CarRoute[]`+`RouteClient[]`, собирает тело и отправляет задачу в Yandex. См. раздел 5. |
| `Services/RoutePlanRequestBuilder` | Собирает тело запроса для Yandex MVRP из сохранённого маршрута (депо, машины, точки, опции). См. раздел 6. |

### Слой чтения (query)

| Класс | Роль |
|-------|------|
| `Queries/RoutePlanQuery` | Загрузка данных: машины (`with carType, activeDriver`), заборы (`with order.sender`), доставки (`with customer, city`); список `PENDING`-маршрутов для крона; маршрут с результатом для `show` (`findOrFail`, 404 при отсутствии). |

### Слой интеграции с Yandex

| Класс | Роль |
|-------|------|
| `Repositories/IntegrationRoutePlanRepository` | **HTTP-клиент Yandex.** `submit()` → `POST /vrs/api/v1/add/mvrp` (возвращает `task_id`, код, тело); `fetchResult()` → `GET /vrs/api/v1/result/mvrp/{taskId}`. Заголовок `Authorization: Bearer <token>`. |
| `DTO/RoutePlanResultDTO` | **Разбор ответа Yandex.** `fromResponse()` парсит `result.routes[]` → по каждой машине упорядоченный список точек, и `result.dropped_locations[]` → причины невключения. `isReady()` = `HTTP 200` (иначе Yandex ещё считает). |
| `Services/RoutePlanResultApplier` | **Применение результата к базе.** Раскладывает точки по машинам с `position`, ставит статус `ROUTED` и время прибытия, считает итоги по машине (вес/объём/кол-во), помечает остальные точки `DROPPED` с причиной, ставит маршруту `SUCCESS`. См. раздел 7. |

### Слой хранения (запись в БД)

| Класс | Роль |
|-------|------|
| `Repositories/RoutePlanRepository` | Сохранение/обновление `Route`, `CarRoute`, `RouteClient` и запись аудит-лога `RouteResult`. |

### Крон

| Класс | Роль |
|-------|------|
| `Console/Commands/CheckRoutePlanStatusCommand` | Плановая команда `route-plan:check-status`. Опрашивает Yandex по всем `PENDING`-маршрутам с `task_id`, применяет готовые результаты. Ошибка одного маршрута не останавливает остальные. |

### Модели (таблицы БД)

| Модель | Таблица | Роль |
|--------|---------|------|
| `Models/Route` | `routes` | Маршрут-агрегат: дата, статус, `task_id`, координаты депо, ошибка. |
| `Models/CarRoute` | `car_routes` | Маршрут одной машины: снимок номера/водителя/рабочих часов, итоги (вес/объём/кол-во точек). |
| `Models/RouteClient` | `route_clients` | Точка маршрута (забор или доставка): координаты, вес/объём, позиция, статус, причина невключения. |
| `Models/RouteResult` | `route_results` | Аудит-лог обмена с Yandex (запрос отправки и полученный результат). |

---

## 5. Обработчик `CreateRoutePlanHandler::handle` (по шагам)

1. **Загрузка данных** (через `RoutePlanQuery`): машины, заборы, доставки по id из DTO.
2. **Определение депо** (`resolveWarehouse`): берётся `city_id` первого забора, иначе
   первой доставки → запрос склада `HttpWarehouseQuery::getByCityId` (модуль
   `DispatcherSector`, вызов через `Contracts`) → `WarehouseDTO{cityId, latitude,
   longitude}`. Координаты депо сохраняются в `routes.depot_lat/depot_lon`.
3. **Валидация** (`validate`, при ошибке — `422`):
   - машина: `is_routing` должно быть `true`;
   - забор: есть координаты отправителя (`order.sender.latitude/longitude`) и `weight > 0`;
   - доставка: есть координаты (`getLatitude()/getLongitude()`) и `weight > 0`;
   - должен найтись склад-депо.
   - Координаты проходят через `toCoordinate`: `null`/пусто/не-число/`0.0` считаются
     отсутствующими.
4. **Сохранение** (в одной транзакции `DB::transaction`):
   - `Route` (`user_id`, `company_id` = у первой машины, `city_id`, `date`,
     `status = PENDING`, `depot_lat/lon`);
   - `CarRoute` на каждую машину: снимок номера, водителя (`activeDriver`) и рабочих
     часов. `resolveWorkHours`: приоритет — переопределение из запроса (если заданы и
     start, и end), иначе часы машины (`getRoutingWorkStart()/End()`, по умолчанию
     `08:00:00`/`19:00:00`);
   - `RouteClient` на каждый забор/доставку: полиморфная связь (`client_id` +
     `client_type` = `OrderTake::class`/`Delivery::class`), `type`, `status = PENDING`,
     снимок координат/веса/объёма.
5. **Отправка** (`submit`, **после** коммита транзакции, синхронно): перезагружает
   `carRoutes.car.carType` + `routeClients`, собирает тело (`RoutePlanRequestBuilder`),
   вызывает `IntegrationRoutePlanRepository::submit`. Пишет строку в `route_results`
   (`type = mvrp_submit`, `task_id`, код, тела запроса/ответа). Если `task_id` пустой —
   `Route::markFailed(...)`, иначе сохраняет `task_id`.

---

## 6. Сборка запроса `RoutePlanRequestBuilder::build`

Превращает агрегат `Route` в тело для Yandex MVRP. **Соглашение об id (критично для
обратного сопоставления результата):**

- id машины (`vehicle.id`) = `car_id` (строка);
- id точки (`location.id`) = `route_client.id` (строка), `ref` = `take-{clientId}` /
  `delivery-{clientId}`;
- id депо = `"0"`.

```
depot:      { id:"0", ref:"Склад", time_window, point:{lat,lon} }        // из routes.depot_*
vehicles[]: { id:car_id, ref:car_number,
              capacity:{ weight_kg, volume_cbm, units }, shifts:[{ id:"0", time_window }] }
locations[]:{ id:route_client.id, ref, time_window, point:{lat,lon},
              shipment_size:{ weight_kg, volume_cbm, units:1 }, service_duration_s }
options:    { time_zone, quality, date }
```

Грузоподъёмность машины берётся из `Car`: `getRoutingWeightCapacity()` =
`carType.capacity`, `getRoutingVolume()` = `cubature` или `carType.volume`,
`getRoutingMaxOrders()` = `carType.max_orders` (по умолчанию `30`). `time_window`
машины = её рассчитанные рабочие часы.

⚠️ **Захардкоженные значения** (помечены на доработку) в `RoutePlanRequestBuilder`:
`DEPOT_TIME_WINDOW = 07:00:00-22:00:00`, `CLIENT_TIME_WINDOW = 08:00:00-18:00:00`,
`SERVICE_DURATION_S = 600` (ожидание на точке), `TIME_ZONE = 5`, `QUALITY = normal`.
Временные окна клиентов пока **не** берутся из тайм-слотов заборов/доставок.

---

## 7. Крон и применение результата

**Команда `route-plan:check-status` (`CheckRoutePlanStatusCommand`)**
Запланирована в `Console/Kernel`: `->everyFiveMinutes()->between('06:00','22:00')`.
Работает только если включён флаг настроек `Settings::YANDEX_ROUTING` (`yandex_routing`,
проверяется через `SettingsService::isEnabled`).

Для каждого `PENDING`-маршрута с `task_id` (`getPendingForStatusCheck`):
1. `IntegrationRoutePlanRepository::fetchResult(task_id)` → `GET .../result/mvrp/{task_id}`.
2. `RoutePlanResultDTO::fromResponse(code, body)`. `isReady()` = `code === 200`
   (иначе Yandex ещё считает — пропускаем). Пишется строка `route_results`
   (`type = mvrp_result`).
3. `RoutePlanResultApplier::apply($route, $dto)`.
4. Ошибки по конкретному маршруту логируются, один сбой не останавливает пакет.

**`RoutePlanResultDTO::fromResponse` разбирает:**
- `result.routes[]` → `routesByVehicle[vehicle_id] = [{ id, arrival }, …]` (только узлы
  с `node.type === 'location'`, в порядке объезда; `arrival` из `arrival_time_s` или
  `arrival_time`);
- `result.dropped_locations[]` → `droppedReasons[id] = drop_reason | reason | 'unknown'`.

**`RoutePlanResultApplier::apply` (одна транзакция):**
- машины (`carRoutes`) индексируются по `car_id`, точки (`routeClients`) — по `id`;
- по каждой машине/точке: проставляется `car_route_id`, `position` (с 0),
  `status = ROUTED`, `arrival_at`; накапливаются `total_weight/total_volume/stops_count`.
  Неизвестные id машины/точки пропускаются;
- `markDropped`: каждая точка не в статусе `ROUTED` → `status = DROPPED`,
  `car_route_id = null`, причина из `droppedReasons[id]` или «Не включен в маршрут»;
- `Route::markSuccess()` → `status = SUCCESS`.

---

## 8. Модель данных

| Таблица | Модель | Ключевые поля |
|---------|--------|---------------|
| `routes` | `Route` | `user_id, company_id, city_id, date, status(1/2/3), task_id, depot_lat, depot_lon, error` + softDeletes |
| `car_routes` | `CarRoute` | `route_id, car_id, car_number, driver_*, work_start/end, total_weight/volume, stops_count` + softDeletes |
| `route_clients` | `RouteClient` | `route_id, car_route_id?, type(1/2), client_id+client_type (морф), position, status(0/1/2), drop_reason, lat/lon, weight, volume, arrival_at` + softDeletes |
| `route_results` | `RouteResult` | `route_id, type(mvrp_submit/mvrp_result), task_id, status_code, request_body, response_body` (аудит-лог) |

Новые поля в существующих таблицах: `cars.is_routing` (bool, по умолчанию false),
`cars.work_start/work_end` (time, nullable); `car_types.max_orders` (uint, nullable).

**Константы:**
- `Route`: `STATUS_PENDING=1, STATUS_SUCCESS=2, STATUS_FAILED=3`.
- `RouteClient`: `TYPE_TAKE=1, TYPE_DELIVERY=2`; `STATUS_PENDING=0, STATUS_ROUTED=1, STATUS_DROPPED=2`.
- `RouteResult`: `TYPE_MVRP_SUBMIT='mvrp_submit', TYPE_MVRP_RESULT='mvrp_result'`.

---

## 9. Конфигурация и DI

**`config/yandex-routing.php`:** `url` (`YANDEX_ROUTING_URL`, по умолчанию
`https://courier.yandex.ru`), `token` (`YANDEX_ROUTING_TOKEN`, идёт как
`Authorization: Bearer`), плюс `api_key`, `companyId`. Пути интеграции:
отправка `/vrs/api/v1/add/mvrp`, результат `/vrs/api/v1/result/mvrp/{taskId}`.

**Регистрация зависимостей:**
- `CommandBusServiceProviders`: `Bus::map([CreateRoutePlanCommand => CreateRoutePlanHandler])`.
- `QueryServiceProvider`: контракт `RoutePlanQuery` → реализация `Queries\RoutePlanQuery`.
- `RepositoryServiceProvider`: `RoutePlanRepository` → Eloquent-реализация;
  `IntegrationRoutePlanRepository` → HTTP-реализация.
- Контракты (граница между модулями): `Contracts/Queries/RoutePlanQuery`,
  `Contracts/Repositories/RoutePlanRepository`, `Contracts/Repositories/IntegrationRoutePlanRepository`.

---

## 10. Важные нюансы

- **`units` (лимит заказов на машину)** = `carType.max_orders` (по умолчанию 30). Это
  ограничение может приводить к тому, что часть заказов окажется в `unrouted`.
- **Координаты:** для забора — `order.sender.lat/lon`, для доставки —
  `getLatitude()/getLongitude()`; значение `0.0` считается отсутствующим и не проходит
  валидацию.
- **`show` читает только БД** — Yandex не вызывается. До отработки крона успешно
  отправленный маршрут показывает `status = 1` с пустыми `stops` и `unrouted`.
- **Отправка синхронная** — медленный/недоступный Yandex замедляет ответ на `create`.
  Неудачная отправка оставляет маршрут в статусе `FAILED` (повторной отправки пока нет).
- **Соглашение об id** (машина = `car_id`, точка = `route_client.id`) обязательно к
  соблюдению: по нему результат Yandex сопоставляется обратно с нашими записями.
- **Тестовые данные:** `database/seeders/RoutePlanTestSeeder.php` создаёт одну машину с
  водителем, один забор и одну доставку в одном городе. Запуск:
  `php artisan db:seed --class=RoutePlanTestSeeder` — выводит готовое тело для
  `POST /api/routing/plan`.
- В модуле `Routing` мало тестов — добавляйте покрытие при изменениях.
