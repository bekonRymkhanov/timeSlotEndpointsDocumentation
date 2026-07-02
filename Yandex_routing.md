# Маршрутизация через Yandex MVRP — документация

> Ручное (по запросу диспетчера) построение маршрутов через Yandex Routing (MVRP).
> Диспетчер сам выбирает машины и заказы (заборы/доставки) на дату — система строит
> задачу для Yandex, отправляет её, а готовый результат забирает **очередной задачей
> (job), которая опрашивает Yandex сама, переставляя себя в очередь**.
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
2. **Получение результата.** Сразу после отправки запускается **очередная задача
   (job)** `PollRoutePlanResultCommand`. Она опрашивает Yandex по `task_id`; пока
   результат не готов, задача **сама переставляет себя** в очередь с задержкой (первый
   опрос через ~2 мин, далее каждые 10 мин). Как только Yandex посчитал — мы разбираем
   ответ, раскладываем точки по машинам в порядке объезда, помечаем невписавшиеся
   заказы и выставляем маршруту статус «Готов».

Из-за этого фронтенд после отправки **опрашивает** маршрут (эндпоинт `show`), пока
статус не станет `2` (готов) или `3` (ошибка).

> Раньше результат забирал крон каждые 5 минут (`route-plan:check-status`). Теперь это
> делает самоперезапускающаяся job — крон из расписания убран, но сама консольная
> команда сохранена как ручной/резервный «сборщик» (см. раздел 7).

### Автоподхват уже назначенных заказов курьера

Если у водителя выбранной машины **уже есть** заборы/доставки, назначенные на эту дату,
система **сама добавляет их в маршрут и жёстко закрепляет за этой машиной** — Yandex обязан
оставить эти точки именно на этом курьере, а не переносить их на другую машину и не
выбрасывать. Пример: выбрана машина `1` с заборами `[2]` и доставками `[3]`, а у её курьера
на этот день уже назначен забор `6` — в Yandex уйдут заборы `2, 6` и доставка `3`, причём
`6` привязан к машине `1`.

Что подхватывается (на дату маршрута):
- **заборы** курьера с `take_date` = дата маршрута и статусом «ещё не завершён»
  (`StatusType::ORDER_TAKE_INCOMPLETED_STATUSES`);
- **доставки** курьера, у которых `invoice.delivery_date` = дата маршрута и статус ≠ «доставлен»
  (`StatusType::ID_DELIVERED`).

Явно выбранные диспетчером заказы **не** закрепляются — Yandex распределяет их свободно.
Заказы, уже указанные в запросе, повторно не добавляются. Автоподхваченный заказ без координат
или с нулевым весом **молча пропускается** (в отличие от явно выбранных — те дают ошибку
валидации). Технически закрепление реализовано через теги Yandex (см. раздел 6).

> Обратного назначения курьера на заказы **после** расчёта не делается: привязка нужна только
> для того, чтобы Yandex построил маршрут с учётом уже висящей на курьере работы. Статусы
> заборов/доставок и `courier_id` этой фичей не меняются.

### Статусы маршрута

| Код | Значение | Когда выставляется |
|-----|----------|--------------------|
| `1` | В процессе (`PENDING`) | Сразу после создания и отправки в Yandex |
| `2` | Готов (`SUCCESS`) | Когда job получил и применил результат |
| `3` | Ошибка (`FAILED`) | Если Yandex не вернул `task_id` при отправке, **или** результат не пришёл за отведённое окно (~3 ч опроса) |

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
 │        • автоподхват заказов курьера + закрепление (pinned)     │
 │        • сохранение Route + CarRoute[] + RouteClient[] (PENDING)│
 │        • сборка тела запроса (RoutePlanRequestBuilder)         │
 │        • ОТПРАВКА в Yandex (POST /vrs/api/v1/add/mvrp)         │
 │        • сохранение task_id (или статус FAILED)               │
 │        • запуск job опроса: Bus::dispatch(PollRoutePlanResult) │
 └──────────────────────────────────────────────────────────────┘
        │
        │ 2. Ответ 201  { id, status: 1, date, taskId }   ← результат ещё НЕ готов
        ▼
   Фронтенд периодически опрашивает show…

 ─ ─ ─ (очередь `routing`, воркер queue:work; сама себя переставляет) ─ ─ ─

 ┌──────────────────────────────────────────────────────────────┐
 │ PollRoutePlanResultCommand → PollRoutePlanResultHandler       │
 │   (delay: 1-й опрос ~2 мин, далее каждые 10 мин)               │
 │   → флаг Settings::YANDEX_ROUTING включён?                     │
 │   → маршрут ещё PENDING и есть task_id? (findPending)          │
 │   → GET /vrs/api/v1/result/mvrp/{task_id}                     │
 │        • НЕ готов (не 200) / ошибка сети                       │
 │             → re-dispatch того же job с attempt+1 (в 10 мин)   │
 │             → после MAX_ATTEMPTS (~3 ч) → markFailed           │
 │        • готов (HTTP 200)                                      │
 │             → RoutePlanResultDTO разбирает ответ               │
 │             → RoutePlanResultApplier раскладывает точки по     │
 │               машинам, помечает невписавшиеся, итоги, SUCCESS  │
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
(не в очереди) — в очередь уходит только последующий опрос результата. Поэтому
медленный ответ Yandex на этапе submit замедляет ответ пользователю. Маршрут
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
- **Читает только базу** — Yandex не вызывается. Пока job опроса не применил результат,
  будет `status = 1` с пустыми `stops` и пустым `unrouted`.

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
| `pinnedCarId` | ID машины, за которой закреплена точка (заказ уже был назначен курьеру); `null` у обычных |
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

### Слой команды создания (запись, CQRS)

| Класс | Роль |
|-------|------|
| `DTO/CreateRoutePlanDTO` | Неизменяемый набор данных из запроса. `fromRequest` «разворачивает» `cars` в `carIds` + `carWorkHours[carId] = {start,end}`, нормализует время (`08:00` → `08:00:00`) и запоминает `userId = Auth::id()`. |
| `Commands/CreateRoutePlanCommand` | Команда-обёртка над DTO (объект-сообщение для шины команд). |
| `Handlers/CreateRoutePlanHandler` | **Главный обработчик создания.** Загружает сущности, определяет депо, валидирует, сохраняет агрегат `Route`+`CarRoute[]`+`RouteClient[]`, собирает тело, отправляет задачу в Yandex и **запускает job опроса результата**. См. раздел 5. |
| `Services/RoutePlanRequestBuilder` | Собирает тело запроса для Yandex MVRP из сохранённого маршрута (депо, машины, точки, опции). См. раздел 6. |

### Слой опроса результата (очередь / job)

| Класс | Роль |
|-------|------|
| `Commands/PollRoutePlanResultCommand` | **Очередная задача (`implements ShouldQueue`, очередь `routing`).** Хранит `routeId` и `attempt` (номер попытки). Поле `delay` = задержка перед запуском: `FIRST_DELAY_S = 120` для 1-й попытки, дальше `RETRY_DELAY_S = 600`. |
| `Handlers/PollRoutePlanResultHandler` | **Опрашивает Yandex по одному маршруту и применяет готовый результат.** Пока не готово (или сетевая ошибка) — переставляет себя в очередь с `attempt+1`; лимит `MAX_ATTEMPTS = 18` (~3 ч), после чего маршрут → `FAILED`. См. раздел 7. |

### Слой чтения (query)

| Класс | Роль |
|-------|------|
| `Queries/RoutePlanQuery` | Загрузка данных: машины (`with carType, activeDriver`), заборы (`with order.sender`), доставки (`with customer, city`); `findPending($routeId)` — один PENDING-маршрут с `task_id` для job опроса; `getPendingForStatusCheck` — все PENDING для резервного «сборщика»; `getWithResult` — маршрут с результатом для `show` (`findOrFail`, 404 при отсутствии). |

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

### Резервный «сборщик» (консольная команда)

| Класс | Роль |
|-------|------|
| `Console/Commands/CheckRoutePlanStatusCommand` | Команда `route-plan:check-status`. Опрашивает Yandex по **всем** `PENDING`-маршрутам с `task_id` и применяет готовые результаты. **Убрана из расписания** — оставлена как ручной запуск / страховочный «сборщик» (см. раздел 7, «Надёжность»). |

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
3a. **Автоподхват заказов курьера** (`attachCourierOrders`, после валидации, до транзакции):
   по каждой машине с активным водителем (`activeDriver`) через `RoutePlanQuery::getCourierTakes` /
   `getCourierDeliveries` берутся ещё не завершённые заборы (по `take_date`) и доставки (по
   `invoice.delivery_date`) на дату маршрута, которых ещё нет в запросе. Точки без координат или
   с `weight <= 0` молча пропускаются. Остальные добавляются в наборы заборов/доставок, а карта
   «id заказа → id машины» (`pins`) запоминает, за какой машиной их закрепить.
4. **Сохранение** (в одной транзакции `DB::transaction`):
   - `Route` (`user_id`, `company_id` = у первой машины, `city_id`, `date`,
     `status = PENDING`, `depot_lat/lon`);
   - `CarRoute` на каждую машину: снимок номера, водителя (`activeDriver`) и рабочих
     часов. `resolveWorkHours`: приоритет — переопределение из запроса (если заданы и
     start, и end), иначе часы машины (`getRoutingWorkStart()/End()`, по умолчанию
     `08:00:00`/`19:00:00`);
   - `RouteClient` на каждый забор/доставку: полиморфная связь (`client_id` +
     `client_type` = `OrderTake::class`/`Delivery::class`), `type`, `status = PENDING`,
     снимок координат/веса/объёма, а также `pinned_car_id` из карты `pins` (`null` для явно
     выбранных заказов, `car_id` — для автоподхваченных).
5. **Отправка + запуск опроса** (`submit`, **после** коммита транзакции, синхронно):
   перезагружает `carRoutes.car.carType` + `routeClients`, собирает тело
   (`RoutePlanRequestBuilder`), вызывает `IntegrationRoutePlanRepository::submit`. Пишет
   строку в `route_results` (`type = mvrp_submit`, `task_id`, код, тела запроса/ответа).
   Если `task_id` пустой — `Route::markFailed(...)`. Иначе сохраняет `task_id` и
   **ставит первую задачу опроса**: `Bus::dispatch(new PollRoutePlanResultCommand($route->id))`.

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
vehicles[]: { id:car_id, ref:car_number, tags:["car-<car_id>"],
              capacity:{ weight_kg, volume_cbm, units }, shifts:[{ id:"0", time_window }] }
locations[]:{ id:route_client.id, ref, time_window, point:{lat,lon},
              shipment_size:{ weight_kg, volume_cbm, units:1 }, service_duration_s,
              required_tags:["car-<pinned_car_id>"]   // ← ТОЛЬКО у закреплённых точек }
options:    { time_zone, quality, date }
```

**Закрепление точки за машиной (теги).** У каждой машины есть уникальный тег `car-<car_id>`
(`vehicles[].tags`). Точка может быть обслужена машиной, только если один из тегов машины
совпадает с `required_tags` точки. Поэтому автоподхваченным (закреплённым) точкам проставляется
`required_tags: ["car-<pinned_car_id>"]` — и Yandex обязан оставить их на этом курьере. У явно
выбранных заказов `required_tags` **нет**, поэтому Yandex распределяет их свободно. Префикс тега —
`RoutePlanRequestBuilder::VEHICLE_TAG_PREFIX` (`car-`); тег машины и `required_tags` точки должны
использовать один и тот же префикс.

Грузоподъёмность машины берётся из `Car`: `getRoutingWeightCapacity()` =
`carType.capacity`, `getRoutingVolume()` = `cubature` или `carType.volume`,
`getRoutingMaxOrders()` = `carType.max_orders` (по умолчанию `30`). `time_window`
машины = её рассчитанные рабочие часы.

⚠️ **Захардкоженные значения** (помечены на доработку) в `RoutePlanRequestBuilder`:
`DEPOT_TIME_WINDOW = 07:00:00-22:00:00`, `CLIENT_TIME_WINDOW = 08:00:00-18:00:00`,
`SERVICE_DURATION_S = 600` (ожидание на точке), `TIME_ZONE = 5`, `QUALITY = normal`.
Временные окна клиентов пока **не** берутся из тайм-слотов заборов/доставок.

---

## 7. Опрос результата (очередь / job)

### `PollRoutePlanResultCommand` — сообщение очереди

- `implements ShouldQueue`, очередь `routing` (`public string $queue = 'routing'`).
- Полезная нагрузка: `routeId` и `attempt` (номер попытки, с 1).
- `delay` (секунды до запуска) вычисляется в конструкторе: `attempt <= 1` → `120`
  (первый опрос ~2 мин после отправки), иначе `600` (далее каждые 10 мин).

### `PollRoutePlanResultHandler::handle` — что делает один опрос

1. Если флаг `Settings::YANDEX_ROUTING` выключен (`SettingsService::isEnabled`) — выходим.
2. `RoutePlanQuery::findPending($routeId)` → маршрут в статусе `PENDING` с `task_id`.
   Если `null` (уже применён/провален/удалён) — выходим, ничего не делаем.
3. `IntegrationRoutePlanRepository::fetchResult(task_id)` → `GET .../result/mvrp/{task_id}`.
   `RoutePlanResultDTO::fromResponse(code, body)`.
   - **Сетевая ошибка** (исключение) → перепланировать (`rescheduleOrFail`).
   - **Не готов** (`isReady()` = `code !== 200`) → перепланировать.
   - **Готов** (`code === 200`) → записать `route_results` (`type = mvrp_result`) и
     применить (`RoutePlanResultApplier::apply`).
4. `rescheduleOrFail`: если `attempt >= MAX_ATTEMPTS (18)` → `Route::markFailed(...)`
   (результат не пришёл за ~3 ч). Иначе — `Bus::dispatch(new PollRoutePlanResultCommand(
   $routeId, $attempt + 1))` (следующий опрос через 10 мин).

### Почему именно так (важно для правок)

- Каждый опрос — **отдельный, новый job** (не «зависший» долгоживущий). Поэтому воркер
  `routing` с `--tries=1` (см. `supervisord.conf`) не рвёт цепочку: неудачная попытка не
  «съедает» лимит ретраев всей задачи. Ограничение цикла — наш `MAX_ATTEMPTS`, а не очередь.
- **Ошибку Yandex ловим и перепланируем сами** (try/catch). При `--tries=1` непойманное
  исключение завершило бы job без повторной постановки → маршрут навсегда завис бы в
  `PENDING`. Ловля ошибки держит цикл живым (как это делал per-route try/catch в кроне).
- Задача рассчитана на **асинхронный драйвер очереди** — в staging/prod это redis
  (`QUEUE_CONNECTION=redis`), задержки `delay` реально работают. В `sync`-режиме (тесты)
  job выполняется сразу, инлайн.

### `RoutePlanResultDTO::fromResponse` разбирает

- `result.routes[]` → `routesByVehicle[vehicle_id] = [{ id, arrival }, …]` (только узлы
  с `node.type === 'location'`, в порядке объезда; `arrival` из `arrival_time_s` или
  `arrival_time`);
- `result.dropped_locations[]` → `droppedReasons[id] = drop_reason | reason | 'unknown'`.

### `RoutePlanResultApplier::apply` (одна транзакция)

- машины (`carRoutes`) индексируются по `car_id`, точки (`routeClients`) — по `id`;
- по каждой машине/точке: проставляется `car_route_id`, `position` (с 0),
  `status = ROUTED`, `arrival_at`; накапливаются `total_weight/total_volume/stops_count`.
  Неизвестные id машины/точки пропускаются;
- `markDropped`: каждая точка не в статусе `ROUTED` → `status = DROPPED`,
  `car_route_id = null`, причина из `droppedReasons[id]` или «Не включен в маршрут»;
- `Route::markSuccess()` → `status = SUCCESS`.

### Надёжность: очередь vs крон

Крон был «самозаживляющимся»: каждые 5 минут он пере-сканировал **все** PENDING-маршруты,
поэтому потерянная проверка всё равно бы позже подхватилась. У цепочки job такого
запаса нет: если job потеряется (сброс redis, воркер лежал всё окно опроса, чистка
`failed_jobs`) — маршрут останется `PENDING`, и подтолкнуть его будет некому.

Дешёвая страховка без возврата к «busy-poll» кроном: изредка (например, раз в час)
запускать сохранённую команду `route-plan:check-status` как «сборщик» зависших PENDING.
Это принципиально иное, чем опрос каждые 5 минут.

---

## 8. Модель данных

| Таблица | Модель | Ключевые поля |
|---------|--------|---------------|
| `routes` | `Route` | `user_id, company_id, city_id, date, status(1/2/3), task_id, depot_lat, depot_lon, error` + softDeletes |
| `car_routes` | `CarRoute` | `route_id, car_id, car_number, driver_*, work_start/end, total_weight/volume, stops_count` + softDeletes |
| `route_clients` | `RouteClient` | `route_id, car_route_id?, pinned_car_id?, type(1/2), client_id+client_type (морф), position, status(0/1/2), drop_reason, lat/lon, weight, volume, arrival_at` + softDeletes |
| `route_results` | `RouteResult` | `route_id, type(mvrp_submit/mvrp_result), task_id, status_code, request_body, response_body` (аудит-лог) |

Новые поля в существующих таблицах: `cars.is_routing` (bool, по умолчанию false),
`cars.work_start/work_end` (time, nullable); `car_types.max_orders` (uint, nullable).

**Константы:**
- `Route`: `STATUS_PENDING=1, STATUS_SUCCESS=2, STATUS_FAILED=3`.
- `RouteClient`: `TYPE_TAKE=1, TYPE_DELIVERY=2`; `STATUS_PENDING=0, STATUS_ROUTED=1, STATUS_DROPPED=2`.
- `RouteResult`: `TYPE_MVRP_SUBMIT='mvrp_submit', TYPE_MVRP_RESULT='mvrp_result'`.
- `PollRoutePlanResultCommand`: `FIRST_DELAY_S=120, RETRY_DELAY_S=600`.
- `PollRoutePlanResultHandler`: `MAX_ATTEMPTS=18` (~3 ч окна опроса).

---

## 9. Конфигурация и DI

**`config/yandex-routing.php`:** `url` (`YANDEX_ROUTING_URL`, по умолчанию
`https://courier.yandex.ru`), `token` (`YANDEX_ROUTING_TOKEN`, идёт как
`Authorization: Bearer`), плюс `api_key`, `companyId`. Пути интеграции:
отправка `/vrs/api/v1/add/mvrp`, результат `/vrs/api/v1/result/mvrp/{taskId}`.

**Очередь и воркер:**
- `QUEUE_CONNECTION=redis` (staging/prod), `sync` в тестах.
- Отдельный воркер очереди `routing` в `supervisord.conf`:
  `php artisan queue:work redis --queue=routing --sleep=3 --tries=1 --max-time=3600`.

**Регистрация зависимостей:**
- `CommandBusServiceProviders`: `Bus::map([CreateRoutePlanCommand => CreateRoutePlanHandler,
  PollRoutePlanResultCommand => PollRoutePlanResultHandler])`.
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
- **`show` читает только БД** — Yandex не вызывается. Пока job опроса не применил
  результат, успешно отправленный маршрут показывает `status = 1` с пустыми `stops` и
  `unrouted`.
- **Отправка синхронная** — медленный/недоступный Yandex замедляет ответ на `create`.
  Неудачная отправка оставляет маршрут в статусе `FAILED` (повторной отправки пока нет).
- **Опрос — асинхронный** и **самоперезапускающийся**: рассчитан на redis. При потере
  job страховки в виде авто-пере-сканирования нет — держите резервный «сборщик» (раздел 7).
- **Соглашение об id** (машина = `car_id`, точка = `route_client.id`) обязательно к
  соблюдению: по нему результат Yandex сопоставляется обратно с нашими записями.
- **Закрепление заказов курьера** делается тегами (`vehicles[].tags` = `car-<car_id>`,
  `locations[].required_tags` у закреплённых точек). Меняете префикс/схему тега — меняйте
  сразу с обеих сторон (см. раздел 6). `pinned_car_id != null` ⇔ точка автоподхвачена и
  закреплена; помощник модели — `RouteClient::isPinned()`.
- **Тестовые данные:** `database/seeders/RoutePlanTestSeeder.php` создаёт одну машину с
  водителем, один забор и одну доставку в одном городе. Запуск:
  `php artisan db:seed --class=RoutePlanTestSeeder` — выводит готовое тело для
  `POST /api/routing/plan`.
- **Тесты фичи:** `tests/Feature/Routing/RoutePlanTest.php` — создание/отправка, работа
  «сборщика», fallback рабочих часов, валидация, а также перепланирование job (не готов)
  и провал по `MAX_ATTEMPTS`.
