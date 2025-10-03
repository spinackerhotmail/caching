# Поддержка распределённого кеша в ASP.NET Core

## Оглавление
1. [Введение](#введение)
2. [Встроенная поддержка](#встроенная-поддержка)
3. [Доступные провайдеры кеша](#доступные-провайдеры-кеша)
4. [Настройка и конфигурация](#настройка-и-конфигурация)
5. [Примеры использования](#примеры-использования)
6. [Лучшие практики](#лучшие-практики)
7. [Полезные ссылки](#полезные-ссылки)

## Введение

ASP.NET Core предоставляет мощную встроенную поддержку распределённого кеширования через интерфейс `IDistributedCache`. Эта абстракция позволяет легко переключаться между различными провайдерами кеша без изменения кода приложения.

## Встроенная поддержка

### Основной интерфейс: IDistributedCache

ASP.NET Core предоставляет интерфейс `IDistributedCache` со следующими методами:

```csharp
public interface IDistributedCache
{
    byte[]? Get(string key);
    Task<byte[]?> GetAsync(string key, CancellationToken token = default);
    
    void Set(string key, byte[] value, DistributedCacheEntryOptions options);
    Task SetAsync(string key, byte[] value, DistributedCacheEntryOptions options, CancellationToken token = default);
    
    void Refresh(string key);
    Task RefreshAsync(string key, CancellationToken token = default);
    
    void Remove(string key);
    Task RemoveAsync(string key, CancellationToken token = default);
}
```

### Методы расширения для работы со строками

Для удобства работы ASP.NET Core предоставляет методы расширения:

```csharp
// Получение и установка строковых значений
string value = await cache.GetStringAsync("key");
await cache.SetStringAsync("key", "value", options);

// Работа с JSON объектами (требует Microsoft.Extensions.Caching.Abstractions)
T obj = await cache.GetAsync<T>("key");
await cache.SetAsync("key", obj, options);
```

## Доступные провайдеры кеша

### 1. In-Memory Distributed Cache

**Пакет:** Встроен в ASP.NET Core  
**Назначение:** Для разработки и тестирования

```csharp
// Startup.cs или Program.cs
services.AddDistributedMemoryCache();
```

**Особенности:**
- ✅ Быстрая настройка
- ✅ Не требует внешних зависимостей
- ❌ Данные не сохраняются между перезапусками
- ❌ Не подходит для продакшена с несколькими экземплярами

### 2. SQL Server Distributed Cache

**Пакет:** `Microsoft.Extensions.Caching.SqlServer`

```bash
dotnet add package Microsoft.Extensions.Caching.SqlServer
```

**Настройка:**
```csharp
services.AddDistributedSqlServerCache(options =>
{
    options.ConnectionString = connectionString;
    options.SchemaName = "dbo";
    options.TableName = "TestCache";
});
```

**Создание таблицы:**
```bash
dotnet sql-cache create "connectionString" dbo TestCache
```

**Особенности:**
- ✅ Персистентность данных
- ✅ Использует существующую инфраструктуру БД
- ❌ Медленнее in-memory решений
- ❌ Дополнительная нагрузка на БД

### 3. Redis Distributed Cache

**Пакет:** `Microsoft.Extensions.Caching.StackExchangeRedis`

```bash
dotnet add package Microsoft.Extensions.Caching.StackExchangeRedis
```

**Настройка:**
```csharp
services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = "localhost:6379";
    options.InstanceName = "SampleInstance";
});
```

**Расширенная настройка:**
```csharp
services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = "localhost:6379";
    options.InstanceName = "MyApp";
    options.ConfigurationOptions = new ConfigurationOptions
    {
        EndPoints = { "localhost:6379" },
        Password = "your-password",
        AbortOnConnectFail = false,
        ConnectRetry = 3,
        ConnectTimeout = 5000
    };
});
```

**Особенности:**
- ✅ Высокая производительность
- ✅ Богатая функциональность
- ✅ Горизонтальное масштабирование
- ✅ Персистентность (опционально)
- ❌ Требует дополнительную инфраструктуру


## Настройка и конфигурация

### 1. Базовая настройка в Program.cs (NET 6+)

```csharp
var builder = WebApplication.CreateBuilder(args);

// Выбор провайдера кеша
builder.Services.AddDistributedMemoryCache(); // Для разработки

// Или Redis для продакшена
// builder.Services.AddStackExchangeRedisCache(options =>
// {
//     options.Configuration = builder.Configuration.GetConnectionString("Redis");
// });

builder.Services.AddControllers();

var app = builder.Build();

app.MapControllers();
app.Run();
```

### 2. Конфигурация через appsettings.json

```json
{
  "ConnectionStrings": {
    "Redis": "localhost:6379",
    "SqlServer": "Server=.;Database=CacheDb;Trusted_Connection=true;"
  },
  "DistributedCache": {
    "Provider": "Redis",
    "Redis": {
      "InstanceName": "MyApplication",
      "Configuration": "localhost:6379"
    },
    "SqlServer": {
      "SchemaName": "dbo",
      "TableName": "Cache"
    }
  }
}
```

### 3. Условная настройка провайдера

```csharp
var cacheProvider = builder.Configuration["DistributedCache:Provider"];

switch (cacheProvider?.ToLower())
{
    case "redis":
        builder.Services.AddStackExchangeRedisCache(options =>
        {
            options.Configuration = builder.Configuration.GetConnectionString("Redis");
            options.InstanceName = builder.Configuration["DistributedCache:Redis:InstanceName"];
        });
        break;
    
    case "sqlserver":
        builder.Services.AddDistributedSqlServerCache(options =>
        {
            options.ConnectionString = builder.Configuration.GetConnectionString("SqlServer");
            options.SchemaName = builder.Configuration["DistributedCache:SqlServer:SchemaName"];
            options.TableName = builder.Configuration["DistributedCache:SqlServer:TableName"];
        });
        break;
    
    default:
        builder.Services.AddDistributedMemoryCache();
        break;
}
```

## Примеры использования

### 1. Базовое использование в контроллере

```csharp
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    private readonly IDistributedCache _cache;
    private readonly IUserService _userService;

    public UsersController(IDistributedCache cache, IUserService userService)
    {
        _cache = cache;
        _userService = userService;
    }

    [HttpGet("{id}")]
    public async Task<ActionResult<User>> GetUser(int id)
    {
        string cacheKey = $"user_{id}";
        
        // Попытка получить из кеша
        string cachedUser = await _cache.GetStringAsync(cacheKey);
        if (!string.IsNullOrEmpty(cachedUser))
        {
            return Ok(JsonSerializer.Deserialize<User>(cachedUser));
        }

        // Получение из базы данных
        var user = await _userService.GetUserAsync(id);
        if (user == null)
        {
            return NotFound();
        }

        // Сохранение в кеш
        var cacheOptions = new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(15),
            SlidingExpiration = TimeSpan.FromMinutes(5)
        };

        await _cache.SetStringAsync(cacheKey, JsonSerializer.Serialize(user), cacheOptions);

        return Ok(user);
    }
}
```

### 2. Создание сервиса кеширования

```csharp
public interface ICacheService
{
    Task<T> GetAsync<T>(string key) where T : class;
    Task SetAsync<T>(string key, T value, TimeSpan? expiration = null) where T : class;
    Task RemoveAsync(string key);
    Task RemoveByPatternAsync(string pattern);
}

public class CacheService : ICacheService
{
    private readonly IDistributedCache _distributedCache;
    private readonly ILogger<CacheService> _logger;

    public CacheService(IDistributedCache distributedCache, ILogger<CacheService> logger)
    {
        _distributedCache = distributedCache;
        _logger = logger;
    }

    public async Task<T> GetAsync<T>(string key) where T : class
    {
        try
        {
            var cachedValue = await _distributedCache.GetStringAsync(key);
            if (string.IsNullOrEmpty(cachedValue))
                return null;

            return JsonSerializer.Deserialize<T>(cachedValue);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error getting cached value for key: {Key}", key);
            return null;
        }
    }

    public async Task SetAsync<T>(string key, T value, TimeSpan? expiration = null) where T : class
    {
        try
        {
            var serializedObject = JsonSerializer.Serialize(value);
            var options = new DistributedCacheEntryOptions();

            if (expiration.HasValue)
                options.SetAbsoluteExpiration(expiration.Value);
            else
                options.SetAbsoluteExpiration(TimeSpan.FromMinutes(30)); // По умолчанию 30 минут

            await _distributedCache.SetStringAsync(key, serializedObject, options);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error setting cached value for key: {Key}", key);
        }
    }

    public async Task RemoveAsync(string key)
    {
        try
        {
            await _distributedCache.RemoveAsync(key);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error removing cached value for key: {Key}", key);
        }
    }

    public async Task RemoveByPatternAsync(string pattern)
    {
        // Примечание: Не все провайдеры поддерживают удаление по паттерну
        // Для Redis можно использовать StackExchange.Redis напрямую
        throw new NotImplementedException("Pattern-based removal requires provider-specific implementation");
    }
}
```

### 3. Декоратор для автоматического кеширования

```csharp
public class CachedUserService : IUserService
{
    private readonly IUserService _userService;
    private readonly IDistributedCache _cache;
    private readonly TimeSpan _cacheDuration = TimeSpan.FromMinutes(15);

    public CachedUserService(IUserService userService, IDistributedCache cache)
    {
        _userService = userService;
        _cache = cache;
    }

    public async Task<User> GetUserAsync(int id)
    {
        string cacheKey = $"user_{id}";
        
        var cachedUser = await _cache.GetStringAsync(cacheKey);
        if (!string.IsNullOrEmpty(cachedUser))
        {
            return JsonSerializer.Deserialize<User>(cachedUser);
        }

        var user = await _userService.GetUserAsync(id);
        if (user != null)
        {
            var options = new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = _cacheDuration
            };
            await _cache.SetStringAsync(cacheKey, JsonSerializer.Serialize(user), options);
        }

        return user;
    }

    public async Task<User> UpdateUserAsync(User user)
    {
        var updatedUser = await _userService.UpdateUserAsync(user);
        
        // Инвалидация кеша после обновления
        string cacheKey = $"user_{user.Id}";
        await _cache.RemoveAsync(cacheKey);
        
        return updatedUser;
    }
}
```

### 4. Гибридное кеширование (L1 + L2)

```csharp
public class HybridCacheService
{
    private readonly IMemoryCache _l1Cache;
    private readonly IDistributedCache _l2Cache;
    private readonly ILogger<HybridCacheService> _logger;

    public HybridCacheService(
        IMemoryCache l1Cache, 
        IDistributedCache l2Cache,
        ILogger<HybridCacheService> logger)
    {
        _l1Cache = l1Cache;
        _l2Cache = l2Cache;
        _logger = logger;
    }

    public async Task<T> GetAsync<T>(string key) where T : class
    {
        // L1 Cache (Memory)
        if (_l1Cache.TryGetValue(key, out T cachedValue))
        {
            _logger.LogDebug("Cache hit (L1): {Key}", key);
            return cachedValue;
        }

        // L2 Cache (Distributed)
        var distributedValue = await _l2Cache.GetStringAsync(key);
        if (!string.IsNullOrEmpty(distributedValue))
        {
            var deserializedValue = JsonSerializer.Deserialize<T>(distributedValue);
            
            // Кешируем в L1 на короткое время
            _l1Cache.Set(key, deserializedValue, TimeSpan.FromMinutes(2));
            
            _logger.LogDebug("Cache hit (L2): {Key}", key);
            return deserializedValue;
        }

        _logger.LogDebug("Cache miss: {Key}", key);
        return null;
    }

    public async Task SetAsync<T>(string key, T value, TimeSpan expiration) where T : class
    {
        // Сохраняем в L1
        _l1Cache.Set(key, value, TimeSpan.FromMinutes(2));

        // Сохраняем в L2
        var serializedValue = JsonSerializer.Serialize(value);
        var options = new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = expiration
        };
        
        await _l2Cache.SetStringAsync(key, serializedValue, options);
    }
}
```

## Лучшие практики

### 1. Стратегии именования ключей

- Используйте префиксы (app_entity_id) для группировки.
- Избегайте чрезмерно длинных ключей.
- Добавляйте версионность схемы (schema_v2_product_42) при изменении формата данных.

```csharp
public static class CacheKeys
{
    private const string USER_PREFIX = "user";
    private const string PRODUCT_PREFIX = "product";
    private const string CATEGORY_PREFIX = "category";

    public static string User(int id) => $"{USER_PREFIX}_{id}";
    public static string UserProfile(int userId) => $"{USER_PREFIX}_profile_{userId}";
    public static string Product(int id) => $"{PRODUCT_PREFIX}_{id}";
    public static string ProductsByCategory(int categoryId) => $"{PRODUCT_PREFIX}_category_{categoryId}";
}
```

### 2. Настройка времени жизни кеша

```csharp
public static class CacheOptions
{
    public static DistributedCacheEntryOptions ShortTerm => new()
    {
        AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5)
    };

    public static DistributedCacheEntryOptions MediumTerm => new()
    {
        AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(30),
        SlidingExpiration = TimeSpan.FromMinutes(10)
    };

    public static DistributedCacheEntryOptions LongTerm => new()
    {
        AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(2),
        SlidingExpiration = TimeSpan.FromMinutes(30)
    };
}
```

### 3. Обработка ошибок кеширования

```csharp
public class SafeCacheService
{
    private readonly IDistributedCache _cache;
    private readonly ILogger<SafeCacheService> _logger;

    public async Task<T> GetAsync<T>(string key, Func<Task<T>> factory) where T : class
    {
        try
        {
            // Попытка получить из кеша
            var cached = await _cache.GetStringAsync(key);
            if (!string.IsNullOrEmpty(cached))
            {
                return JsonSerializer.Deserialize<T>(cached);
            }
        }
        catch (Exception ex)
        {
            _logger.LogWarning(ex, "Failed to get from cache, key: {Key}", key);
        }

        // Если кеш недоступен, получаем данные из источника
        var data = await factory();
        
        if (data != null)
        {
            // Пытаемся сохранить в кеш, но не блокируем выполнение при ошибке
            _ = Task.Run(async () =>
            {
                try
                {
                    var serialized = JsonSerializer.Serialize(data);
                    await _cache.SetStringAsync(key, serialized, CacheOptions.MediumTerm);
                }
                catch (Exception ex)
                {
                    _logger.LogWarning(ex, "Failed to set cache, key: {Key}", key);
                }
            });
        }

        return data;
    }
}
```

### 4. Мониторинг кеша

```csharp
public class CacheMetrics
{
    private readonly ILogger<CacheMetrics> _logger;
    private long _hits = 0;
    private long _misses = 0;

    public void RecordHit() => Interlocked.Increment(ref _hits);
    public void RecordMiss() => Interlocked.Increment(ref _misses);

    public double GetHitRatio()
    {
        var totalRequests = _hits + _misses;
        return totalRequests > 0 ? (double)_hits / totalRequests : 0;
    }

    public void LogStatistics()
    {
        _logger.LogInformation("Cache Statistics - Hits: {Hits}, Misses: {Misses}, Hit Ratio: {HitRatio:P2}", 
            _hits, _misses, GetHitRatio());
    }
}
```

### 5. Конфигурация для разных сред

```csharp
// Для Development
if (builder.Environment.IsDevelopment())
{
    builder.Services.AddDistributedMemoryCache();
}
// Для Production
else
{
    builder.Services.AddStackExchangeRedisCache(options =>
    {
        options.Configuration = builder.Configuration.GetConnectionString("Redis");
        options.InstanceName = builder.Configuration["ApplicationName"];
    });
}
```

## Полезные ссылки

- [Официальная документация ASP.NET Core - Distributed caching](https://docs.microsoft.com/en-us/aspnet/core/performance/caching/distributed)
- [Microsoft.Extensions.Caching.Abstractions](https://www.nuget.org/packages/Microsoft.Extensions.Caching.Abstractions/)
- [Microsoft.Extensions.Caching.StackExchangeRedis](https://www.nuget.org/packages/Microsoft.Extensions.Caching.StackExchangeRedis/)
- [Microsoft.Extensions.Caching.SqlServer](https://www.nuget.org/packages/Microsoft.Extensions.Caching.SqlServer/)
- [Redis Documentation](https://redis.io/documentation)
- [StackExchange.Redis](https://github.com/StackExchange/StackExchange.Redis)

## Заключение

ASP.NET Core предоставляет мощную и гибкую систему распределённого кеширования:

- ✅ Можно реализовать гибкую стратегию кеширования, переключаться между провайдерами кеша
- ✅ Используется единый API для работы с кешем
- ✅ Интеграция на уровне DI
- ✅ Поддержка асинхронных операции
- ✅ Настройка различных стратегии срока жизни кеша

Правильное использование распределённого кеша может значительно улучшить производительность вашего приложения и снизить нагрузку на базу данных.