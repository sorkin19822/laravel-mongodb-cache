# Laravel MongoDB Cache Driver — Постановка задачі

> **Репозиторій:** https://github.com/sorkin19822/laravel-mongodb-cache  
> **Статус:** в розробці  
> **Стек:** PHP 8.5 · Laravel 12 · MongoDB 7 · Docker  

---

## Мета проекту

Створити відкритий Composer-пакет, який додає MongoDB як повноцінний кеш-бекенд до Laravel 12. Пакет має бути сучаснішим за всі наявні аналоги на Packagist — підтримувати PHP 8.5, Laravel 12, офіційний драйвер `mongodb/laravel-mongodb ^5`, та реалізовувати Cache Locks (чого немає в конкурентів).

**Ціль для портфоліо:** демонструє знання архітектури Laravel (Cache contracts, ServiceProvider, auto-discovery), сучасного PHP 8.5 (pipe operator `|>`, readonly properties), MongoDB (TTL indexes, атомарні операції), Docker, PHPUnit та CI/CD.

---

## Бізнес-логіка

### Що робить пакет

Після встановлення через `composer require sorkin/laravel-mongodb-cache` розробник може використовувати MongoDB як кеш-драйвер в Laravel без жодного ручного налаштування провайдерів:

```php
// config/cache.php
'stores' => [
    'mongodb' => [
        'driver'     => 'mongodb',
        'connection' => 'mongodb',
        'collection' => 'cache',
        'prefix'     => '',
    ],
],

// Використання — стандартний Laravel API, нічого нового вчити не треба
Cache::store('mongodb')->put('key', $value, 3600);
Cache::store('mongodb')->get('key');
Cache::store('mongodb')->remember('key', 60, fn() => DB::table('...')->get());
Cache::store('mongodb')->lock('process_order', 10)->get(fn() => ...);
```

### Чому MongoDB для кешу

- Проект вже використовує MongoDB як основну БД → не потрібно піднімати Redis
- TTL indexes в MongoDB видаляють прострочені записи нативно (без cron / artisan schedule)
- Атомарний `findOneAndUpdate` дозволяє реалізувати distributed locks без race condition
- Горизонтальне масштабування "з коробки" (replica set, sharding)

---

## Структура пакету

```
laravel-mongodb-cache/
├── src/
│   ├── MongoDbStore.php                  # Основний Store (implements Cache\Store)
│   ├── MongoDbLock.php                   # Distributed Lock (implements Cache\Lock)
│   └── MongoDbCacheServiceProvider.php   # Реєстрація драйвера в Laravel
├── config/
│   └── mongodb-cache.php                 # Публікований конфіг
├── tests/
│   ├── Unit/
│   │   ├── MongoDbStoreTest.php
│   │   └── MongoDbLockTest.php
│   └── Feature/
│       └── CacheIntegrationTest.php
├── docker/
│   ├── php/
│   │   └── Dockerfile                    # PHP 8.5 + ext-mongodb
│   └── mongo/
│       └── init.js                       # Створення TTL index при старті
├── docker-compose.yml                    # php + mongo + mongo-express
├── composer.json
├── phpunit.xml
├── .github/
│   └── workflows/
│       └── tests.yml                     # GitHub Actions CI
└── README.md
```

---

## Клас `MongoDbStore` — детальна специфікація

**Імплементує:** `Illuminate\Contracts\Cache\Store`  
**Також імплементує:** `Illuminate\Contracts\Cache\LockProvider` (для підтримки `Cache::lock()`)

### Схема документа в MongoDB

```json
{
  "_id": "cache_prefix_mykey",
  "value": "<serialized PHP value>",
  "expiration": "<ISODate — MongoDB TTL field>"
}
```

- `_id` = `{prefix}{key}` — унікальний ключ
- `value` — серіалізоване значення (через `serialize()` / `unserialize()`)
- `expiration` — поле типу `UTCDateTime`, по якому MongoDB автоматично видаляє документ через TTL index
- Для `forever` — `expiration` не встановлюється (або встановлюється дуже далеко в майбутньому)

### Обов'язкові методи

| Метод | Поведінка |
|---|---|
| `get(string $key): mixed` | Повертає значення або `null`. Перевіряє `expiration` вручну як fallback |
| `put(string $key, mixed $value, int $seconds): bool` | Upsert документа з `expiration` |
| `putMany(array $values, int $seconds): bool` | Bulk upsert через `bulkWrite` |
| `add(string $key, mixed $value, int $seconds): bool` | Атомарний insert (тільки якщо не існує). Повертає `false` якщо вже є |
| `increment(string $key, mixed $value = 1): int\|bool` | Атомарний `$inc` |
| `decrement(string $key, mixed $value = 1): int\|bool` | Атомарний `$inc` з від'ємним значенням |
| `forever(string $key, mixed $value): bool` | Put без `expiration` |
| `forget(string $key): bool` | `deleteOne` |
| `flush(): bool` | `deleteMany([])` — видаляє всі документи колекції |
| `getPrefix(): string` | Повертає prefix |

### PHP 8.5 фічі

```php
// Pipe operator для ланцюжка десеріалізації
private function deserialize(mixed $value): mixed
{
    return $value |> base64_decode(...) |> unserialize(...);
}

// Readonly properties в конструкторі
public function __construct(
    private readonly Collection $collection,
    private readonly string $prefix = '',
) {}
```

---

## Клас `MongoDbLock` — детальна специфікація

**Імплементує:** `Illuminate\Cache\Lock` (extends абстрактний клас)  

### Схема документа lock в MongoDB

```json
{
  "_id": "lock_mylock",
  "owner": "<unique random string>",
  "expiration": "<ISODate>"
}
```

### Логіка acquire

```
findOneAndUpdate(
  filter:  { _id: lockKey, $or: [ {expiration: не існує}, {expiration: {$lte: now}} ] },
  update:  { $set: { owner: $this->owner, expiration: now + seconds } },
  options: { upsert: true }
)
```

Якщо операція успішна — lock захоплено. Якщо документ вже існує і не прострочений — повертає `false`.

### Обов'язкові методи

| Метод | Поведінка |
|---|---|
| `acquire(): bool` | Атомарний findOneAndUpdate з upsert |
| `release(): bool` | `deleteOne` тільки якщо `owner` співпадає |
| `forceRelease(): void` | `deleteOne` без перевірки owner |
| `owner(): string` | Повертає `$this->owner` |

---

## Клас `MongoDbCacheServiceProvider`

```php
public function boot(): void
{
    // Реєстрація кастомного драйвера
    Cache::extend('mongodb', function ($app, $config) {
        $connection = $app['db']->connection('mongodb');
        $collection = $connection->getCollection($config['collection'] ?? 'cache');

        return Cache::repository(
            new MongoDbStore($collection, $config['prefix'] ?? '')
        );
    });

    // Публікація конфігу
    $this->publishes([
        __DIR__.'/../config/mongodb-cache.php' => config_path('mongodb-cache.php'),
    ], 'mongodb-cache-config');
}
```

**Auto-discovery** через `composer.json`:
```json
"extra": {
    "laravel": {
        "providers": [
            "Sorkin\\MongoDbCache\\MongoDbCacheServiceProvider"
        ]
    }
}
```

---

## `composer.json` — залежності

```json
{
    "name": "sorkin/laravel-mongodb-cache",
    "description": "MongoDB cache driver for Laravel 12. PHP 8.5+, TTL indexes, Cache Locks.",
    "type": "library",
    "license": "MIT",
    "require": {
        "php": "^8.5",
        "illuminate/cache": "^12.0",
        "illuminate/support": "^12.0",
        "mongodb/laravel-mongodb": "^5.0"
    },
    "require-dev": {
        "orchestra/testbench": "^10.0",
        "phpunit/phpunit": "^11.0"
    },
    "autoload": {
        "psr-4": {
            "Sorkin\\MongoDbCache\\": "src/"
        }
    },
    "extra": {
        "laravel": {
            "providers": [
                "Sorkin\\MongoDbCache\\MongoDbCacheServiceProvider"
            ]
        }
    }
}
```

---

## Docker-середовище

### `docker-compose.yml` — три сервіси

| Сервіс | Image | Порт | Призначення |
|---|---|---|---|
| `app` | `php:8.5-cli` + ext-mongodb | — | Запуск тестів (`composer test`) |
| `mongo` | `mongo:7` | 27017 | MongoDB сервер |
| `mongo-express` | `mongo-express:latest` | 8081 | Web UI для перегляду кеш-документів |

### `docker/php/Dockerfile`

```dockerfile
FROM php:8.5-cli

# Встановлення ext-mongodb
RUN pecl install mongodb && docker-php-ext-enable mongodb

# Composer
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

WORKDIR /app
COPY . .
RUN composer install
```

### `docker/mongo/init.js` — TTL index

```javascript
// Створюється при першому старті MongoDB
db.cache.createIndex(
    { "expiration": 1 },
    { expireAfterSeconds: 0 }
);

db.locks.createIndex(
    { "expiration": 1 },
    { expireAfterSeconds: 0 }
);
```

---

## Тести — що покривати

### Unit тести (`tests/Unit/`)

**MongoDbStoreTest:**
- `get` повертає `null` для відсутнього ключа
- `get` повертає `null` для прострочених документів
- `put` зберігає значення з правильним `expiration`
- `put` з `$seconds = 0` → видаляє ключ
- `forever` зберігає без `expiration`
- `forget` видаляє документ
- `flush` видаляє всі документи
- `increment` / `decrement` — атомарні операції
- `add` — повертає `false` якщо ключ вже існує
- Prefix коректно додається до ключів

**MongoDbLockTest:**
- `acquire` успішно захоплює вільний lock
- `acquire` повертає `false` якщо lock вже захоплений
- `acquire` успішно захоплює прострочений lock
- `release` звільняє тільки свій lock (по owner)
- `release` повертає `false` якщо owner не співпадає
- `forceRelease` звільняє незалежно від owner

### Feature тести (`tests/Feature/`)

- `Cache::store('mongodb')->remember()` — cache miss та cache hit
- `Cache::store('mongodb')->lock()->get(fn() => ...)` — кінець до кінця
- TTL: документ зникає після закінчення TTL (мок часу)

---

## GitHub Actions CI (`.github/workflows/tests.yml`)

```yaml
name: Tests
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      mongodb:
        image: mongo:7
        ports: ['27017:27017']
    steps:
      - uses: actions/checkout@v4
      - uses: shivammathur/setup-php@v2
        with:
          php-version: '8.5'
          extensions: mongodb
      - run: composer install
      - run: vendor/bin/phpunit
```

---

## Порядок реалізації

1. `composer.json` + autoload
2. `src/MongoDbStore.php` — базові методи (get, put, forget, flush)
3. `src/MongoDbStore.php` — розширені методи (add, increment, putMany, forever)
4. `src/MongoDbLock.php` — acquire / release / forceRelease
5. `src/MongoDbCacheServiceProvider.php` — реєстрація драйвера
6. `config/mongodb-cache.php`
7. `docker-compose.yml` + `docker/php/Dockerfile` + `docker/mongo/init.js`
8. `tests/Unit/MongoDbStoreTest.php`
9. `tests/Unit/MongoDbLockTest.php`
10. `tests/Feature/CacheIntegrationTest.php`
11. `.github/workflows/tests.yml`
12. `README.md` — installation guide + usage examples
13. Публікація на Packagist

---

## Ключові диференціатори від конкурентів на Packagist

| Фіча | Цей пакет | `1ff/laravel-mongodb-cache` | `alfa6661/...` |
|---|---|---|---|
| PHP 8.5+ | ✅ | ❌ (PHP 7+) | ❌ (PHP 7+) |
| Laravel 12 | ✅ | ❌ | ❌ |
| `mongodb/laravel-mongodb` v5 | ✅ | ❌ (jenssegers — abandoned) | ❌ (jenssegers — abandoned) |
| Cache Locks (`Cache::lock()`) | ✅ | ❌ | ❌ |
| Docker середовище | ✅ | ❌ | ❌ |
| GitHub Actions CI | ✅ | ❌ | ❌ |
