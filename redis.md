# Redis

- [簡介](#introduction)
- [設定](#configuration)
    - [叢集](#clusters)
    - [Predis](#predis)
    - [PhpRedis](#phpredis)
- [與 Redis 互動](#interacting-with-redis)
    - [交易](#transactions)
    - [管線化指令](#pipelining-commands)
- [發佈 / 訂閱](#pubsub)

<a name="introduction"></a>
## 簡介

[Redis](https://redis.io) 是一個開源且進階的鍵值儲存系統 (key-value store)。它經常被稱為資料結構伺服器，因為鍵 (keys) 可以包含 [字串](https://redis.io/docs/latest/develop/data-types/strings/)、[雜湊 (hashes)](https://redis.io/docs/latest/develop/data-types/hashes/)、[清單 (lists)](https://redis.io/docs/latest/develop/data-types/lists/)、[集合 (sets)](https://redis.io/docs/latest/develop/data-types/sets/) 以及 [排序集合 (sorted sets)](https://redis.io/docs/latest/develop/data-types/sorted-sets/)。

在將 Redis 與 Laravel 一起使用之前，我們建議您透過 PECL 安裝並使用 [PhpRedis](https://github.com/phpredis/phpredis) PHP 擴充功能。與「使用者層級 (user-land)」的 PHP 套件相比，雖然此擴充功能的安裝較為複雜，但對於大量使用 Redis 的應用程式來說，可以帶來更好的效能。如果您使用的是 [Laravel Sail](/docs/{{version}}/sail)，該擴充功能已經預先安裝在您應用程式的 Docker 容器中。

如果您無法安裝 PhpRedis 擴充功能，您可以透過 Composer 安裝 `predis/predis` 套件。Predis 是一個完全由 PHP 撰寫的 Redis 客戶端，不需要安裝任何額外的擴充功能：

```shell
composer require predis/predis
```

<a name="configuration"></a>
## 設定

你可以透過 `config/database.php` 設定檔來設定應用程式的 Redis 設定。在這個檔案中，你會看到一個包含應用程式所使用之 Redis 伺服器的 `redis` 陣列：

```php
'redis' => [

    'client' => env('REDIS_CLIENT', 'phpredis'),

    'options' => [
        'cluster' => env('REDIS_CLUSTER', 'redis'),
        'prefix' => env('REDIS_PREFIX', Str::slug(env('APP_NAME', 'laravel'), '_').'_database_'),
    ],

    'default' => [
        'url' => env('REDIS_URL'),
        'host' => env('REDIS_HOST', '127.0.0.1'),
        'username' => env('REDIS_USERNAME'),
        'password' => env('REDIS_PASSWORD'),
        'port' => env('REDIS_PORT', '6379'),
        'database' => env('REDIS_DB', '0'),
    ],

    'cache' => [
        'url' => env('REDIS_URL'),
        'host' => env('REDIS_HOST', '127.0.0.1'),
        'username' => env('REDIS_USERNAME'),
        'password' => env('REDIS_PASSWORD'),
        'port' => env('REDIS_PORT', '6379'),
        'database' => env('REDIS_CACHE_DB', '1'),
    ],

],
```

在設定檔中定義的每個 Redis 伺服器都必須包含名稱、主機 (host) 與連接埠 (port)，除非你定義了單一 URL 來代表該 Redis 連線：

```php
'redis' => [

    'client' => env('REDIS_CLIENT', 'phpredis'),

    'options' => [
        'cluster' => env('REDIS_CLUSTER', 'redis'),
        'prefix' => env('REDIS_PREFIX', Str::slug(env('APP_NAME', 'laravel'), '_').'_database_'),
    ],

    'default' => [
        'url' => 'tcp://127.0.0.1:6379?database=0',
    ],

    'cache' => [
        'url' => 'tls://user:password@127.0.0.1:6380?database=1',
    ],

],
```

<a name="configuring-the-connection-scheme"></a>
#### 設定連線 Scheme

預設情況下，Redis 用戶端在連接到 Redis 伺服器時會使用 `tcp` scheme；不過，你可以在 Redis 伺服器的設定陣列中指定 `scheme` 設定選項來使用 TLS / SSL 加密：

```php
'default' => [
    'scheme' => 'tls',
    'url' => env('REDIS_URL'),
    'host' => env('REDIS_HOST', '127.0.0.1'),
    'username' => env('REDIS_USERNAME'),
    'password' => env('REDIS_PASSWORD'),
    'port' => env('REDIS_PORT', '6379'),
    'database' => env('REDIS_DB', '0'),
],
```

<a name="clusters"></a>
### 叢集

如果你的應用程式正在使用 Redis 伺服器叢集，你應該在 Redis 設定的 `clusters` 鍵中定義這些叢集。這個設定鍵預設並不存在，因此你需要自行在應用程式的 `config/database.php` 設定檔中建立它：

```php
'redis' => [

    'client' => env('REDIS_CLIENT', 'phpredis'),

    'options' => [
        'cluster' => env('REDIS_CLUSTER', 'redis'),
        'prefix' => env('REDIS_PREFIX', Str::slug(env('APP_NAME', 'laravel'), '_').'_database_'),
    ],

    'clusters' => [
        'default' => [
            [
                'url' => env('REDIS_URL'),
                'host' => env('REDIS_HOST', '127.0.0.1'),
                'username' => env('REDIS_USERNAME'),
                'password' => env('REDIS_PASSWORD'),
                'port' => env('REDIS_PORT', '6379'),
                'database' => env('REDIS_DB', '0'),
            ],
        ],
    ],

    // ...
],
```

預設情況下，Laravel 會使用原生 Redis 叢集功能，因為 `options.cluster` 設定值被設為 `redis`。Redis 叢集是一個良好的預設選項，因為它可以平滑地處理故障移轉 (Failover)。

Laravel 在使用 Predis 時也支援用戶端分片 (Client-side sharding)。然而，用戶端分片無法處理故障移轉；因此，它主要適用於可從另一個主要資料儲存庫取得的暫時性快取資料。

如果你想使用用戶端分片來替代原生 Redis 叢集，可以在應用程式的 `config/database.php` 設定檔中移除 `options.cluster` 設定值：

```php
'redis' => [

    'client' => env('REDIS_CLIENT', 'phpredis'),

    'clusters' => [
        // ...
    ],

    // ...
],
```

<a name="predis"></a>
### Predis

如果你希望應用程式透過 Predis 套件與 Redis 進行互動，應確保 `REDIS_CLIENT` 環境變數的值為 `predis`：

```php
'redis' => [

    'client' => env('REDIS_CLIENT', 'predis'),

    // ...
],
```

除了預設的設定選項外，Predis 還支援可為每個 Redis 伺服器定義的額外[連線參數](https://github.com/nrk/predis/wiki/Connection-Parameters)。若要使用這些額外的設定選項，請將它們加入到應用程式 `config/database.php` 設定檔中的 Redis 伺服器設定中：

```php
'default' => [
    'url' => env('REDIS_URL'),
    'host' => env('REDIS_HOST', '127.0.0.1'),
    'username' => env('REDIS_USERNAME'),
    'password' => env('REDIS_PASSWORD'),
    'port' => env('REDIS_PORT', '6379'),
    'database' => env('REDIS_DB', '0'),
    'read_write_timeout' => 60,
],
```

<a name="phpredis"></a>
### PhpRedis

預設情況下，Laravel 會使用 PhpRedis 擴充功能與 Redis 進行通訊。Laravel 用來與 Redis 通訊的用戶端是由 `redis.client` 設定選項的值所決定，該選項通常反映了 `REDIS_CLIENT` 環境變數的值：

```php
'redis' => [

    'client' => env('REDIS_CLIENT', 'phpredis'),

    // ...
],
```

除了預設的設定選項外，PhpRedis 還支援以下額外的連線參數：`name`、`persistent`、`persistent_id`、`prefix`、`read_timeout`、`retry_interval`、`max_retries`、`backoff_algorithm`、`backoff_base`、`backoff_cap`、`timeout` 以及 `context`。您可以將這些選項中的任何一個新增至 `config/database.php` 設定檔中的 Redis 伺服器設定：

```php
'default' => [
    'url' => env('REDIS_URL'),
    'host' => env('REDIS_HOST', '127.0.0.1'),
    'username' => env('REDIS_USERNAME'),
    'password' => env('REDIS_PASSWORD'),
    'port' => env('REDIS_PORT', '6379'),
    'database' => env('REDIS_DB', '0'),
    'read_timeout' => 60,
    'context' => [
        // 'auth' => ['username', 'secret'],
        // 'stream' => ['verify_peer' => false],
    ],
],
```

<a name="retry-and-backoff-configuration"></a>
#### 重試與退避設定

`retry_interval`、`max_retries`、`backoff_algorithm`、`backoff_base` 以及 `backoff_cap` 選項可用於設定 PhpRedis 用戶端應如何嘗試重新連線至 Redis 伺服器。目前支援以下退避演算法：`default`、`decorrelated_jitter`、`equal_jitter`、`exponential`、`uniform` 和 `constant`：

```php
'default' => [
    'url' => env('REDIS_URL'),
    'host' => env('REDIS_HOST', '127.0.0.1'),
    'username' => env('REDIS_USERNAME'),
    'password' => env('REDIS_PASSWORD'),
    'port' => env('REDIS_PORT', '6379'),
    'database' => env('REDIS_DB', '0'),
    'max_retries' => env('REDIS_MAX_RETRIES', 3),
    'backoff_algorithm' => env('REDIS_BACKOFF_ALGORITHM', 'decorrelated_jitter'),
    'backoff_base' => env('REDIS_BACKOFF_BASE', 100),
    'backoff_cap' => env('REDIS_BACKOFF_CAP', 1000),
],
```

Predis 3.4.0 及更新版本支援透過 `Retry` 類別進行內建的重試與退避設定。您可以使用 `max_retries` 選項來設定重試，並使用 `retry` 選項設定退避策略。`retry` 選項應該是一個以以下策略類別之一為鍵名的陣列：`NoBackoff`、`EqualBackoff` 或 `ExponentialBackoff`：

```php
use Predis\Retry\Strategy\ExponentialBackoff;

'default' => [
    'url' => env('REDIS_URL'),
    // ...
    'retry' => [
        ExponentialBackoff::class => [
            env('REDIS_BACKOFF_BASE', 100),
            env('REDIS_BACKOFF_CAP', 1000),
            true, // Enable jitter...
        ],
    ],
    'max_retries' => env('REDIS_MAX_RETRIES', 3),
],
```

當在 Redis 叢集中使用 Predis 時，您可以在叢集設定的 `parameters` 選項中定義重試設定：

```php
use Predis\Retry\Strategy\NoBackoff;

'clusters' => [
    'default' => [
        // ...
    ],
],

'options' => [
    'cluster' => env('REDIS_CLUSTER', 'redis'),
    'parameters' => [
        'retry' => [
            NoBackoff::class => [],
        ],
        'max_retries' => env('REDIS_MAX_RETRIES', 3),
    ],
],
```

<a name="unix-socket-connections"></a>
#### Unix Socket 連線

Redis 連線也可以設定為使用 Unix socket 代替 TCP。對於與您的應用程式位於同一台伺服器上的 Redis 執行個體，這可以消除 TCP 開銷，從而提高效能。若要將 Redis 設定為使用 Unix socket，請將 `REDIS_HOST` 環境變數設定為 Redis socket 的路徑，並將 `REDIS_PORT` 環境變數設定為 `0`：

```env
REDIS_HOST=/run/redis/redis.sock
REDIS_PORT=0
```

<a name="phpredis-serialization"></a>
#### PhpRedis 序列化與壓縮

PhpRedis 擴充功能也可以設定為使用各種序列化程式與壓縮演算法。這些演算法可以透過 Redis 設定中的 `options` 陣列進行設定：

```php
'redis' => [

    'client' => env('REDIS_CLIENT', 'phpredis'),

    'options' => [
        'cluster' => env('REDIS_CLUSTER', 'redis'),
        'prefix' => env('REDIS_PREFIX', Str::slug(env('APP_NAME', 'laravel'), '_').'_database_'),
        'serializer' => Redis::SERIALIZER_MSGPACK,
        'compression' => Redis::COMPRESSION_LZ4,
    ],

    // ...
],
```

目前支援的序列化程式包括：`Redis::SERIALIZER_NONE`（預設）、`Redis::SERIALIZER_PHP`、`Redis::SERIALIZER_JSON`、`Redis::SERIALIZER_IGBINARY` 和 `Redis::SERIALIZER_MSGPACK`。

支援的壓縮演算法包括：`Redis::COMPRESSION_NONE`（預設）、`Redis::COMPRESSION_LZF`、`Redis::COMPRESSION_ZSTD` 和 `Redis::COMPRESSION_LZ4`。

<a name="interacting-with-redis"></a>
## 與 Redis 互動

您可以透過呼叫 `Redis` [Facade](/docs/{{version}}/facades) 上的各種方法來與 Redis 進行互動。`Redis` Facade 支援動態方法，這意味著您可以在該 Facade 上呼叫任何 [Redis 指令](https://redis.io/commands)，且該指令將會直接傳遞給 Redis。在這個範例中，我們將透過呼叫 `Redis` Facade 上的 `get` 方法來呼叫 Redis 的 `GET` 指令：

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Support\Facades\Redis;
use Illuminate\View\View;

class UserController extends Controller
{
    /**
     * Show the profile for the given user.
     */
    public function show(string $id): View
    {
        return view('user.profile', [
            'user' => Redis::get('user:profile:'.$id)
        ]);
    }
}
```

如同前述，您可以在 `Redis` Facade 上呼叫任何 Redis 的指令。Laravel 使用魔術方法將這些指令傳遞給 Redis 伺服器。若某個 Redis 指令需要引數，您應該將它們傳入 Facade 所對應的方法中：

```php
use Illuminate\Support\Facades\Redis;

Redis::set('name', 'Taylor');

$values = Redis::lrange('names', 5, 10);
```

或者，您也可以使用 `Redis` Facade 的 `command` 方法來將指令傳送至伺服器，該方法接受指令名稱作為第一個引數，並接受數值陣列作為第二個引數：

```php
$values = Redis::command('lrange', ['name', 5, 10]);
```


<a name="using-multiple-redis-connections"></a>
#### 使用多個 Redis 連線

您的應用程式的 `config/database.php` 設定檔允許您定義多個 Redis 連線 / 伺服器。您可以透過使用 `Redis` Facade 的 `connection` 方法來取得特定的 Redis 連線：

```php
$redis = Redis::connection('connection-name');
```

若要取得預設 Redis 連線的執行個體，您可以在不傳入任何額外引數的情況下呼叫 `connection` 方法：

```php
$redis = Redis::connection();
```


<a name="transactions"></a>
### 交易

`Redis` Facade 的 `transaction` 方法針對 Redis 原生的 `MULTI` 與 `EXEC` 指令提供了方便的封裝。`transaction` 方法僅接受一個閉包作為引數。該閉包將接收一個 Redis 連線執行個體，並可對此執行個體發出任何它想執行的指令。閉包內發出的所有 Redis 指令都將在單一且具原子性的交易中執行：

```php
use Redis;
use Illuminate\Support\Facades;

Facades\Redis::transaction(function (Redis $redis) {
    $redis->incr('user_visits', 1);
    $redis->incr('total_visits', 1);
});
```

> [!WARNING]
> 在定義 Redis 交易時，您無法從 Redis 連線中檢索任何值。請記住，您的交易是作為單一且具原子性的操作來執行的，並且在您的整個閉包完成執行其指令之前，該操作都不會被執行。


#### Lua 腳本

`eval` 方法提供了另一種在單一原子性操作中執行多個 Redis 指令的方法。然而，`eval` 方法的好處在於能夠在該操作期間與 Redis 的鍵值進行互動與檢視。Redis 腳本是用 [Lua 程式語言](https://www.lua.org)編寫的。

`eval` 方法乍看之下可能有點嚇人，但我們將透過一個簡單的範例來打破僵局。`eval` 方法需要幾個引數。首先，您應該將 Lua 腳本（作為字串）傳給該方法。其次，您應該傳入該腳本所互動的鍵數量（作為整數）。第三，您應該傳入這些鍵的名稱。最後，您可以傳入在腳本內部需要存取的任何其他額外引數。

在這個範例中，我們將遞增一個計數器，檢視其新值，如果第一個計數器的值大於 5，則遞增第二個計數器。最後，我們將回傳第一個計數器的值：

```php
$value = Redis::eval(<<<'LUA'
    local counter = redis.call("incr", KEYS[1])

    if counter > 5 then
        redis.call("incr", KEYS[2])
    end

    return counter
LUA, 2, 'first-counter', 'second-counter');
```

> [!WARNING]
> 有關 Redis 腳本編寫的更多資訊，請參閱 [Redis 文件](https://redis.io/commands/eval)。


<a name="pipelining-commands"></a>
### 管線化指令

有時您可能需要執行數十個 Redis 指令。您可以改用 `pipeline` 方法，而不是為每個指令單獨對 Redis 伺服器進行網路傳輸。`pipeline` 方法接受一個引數：接收一個 Redis 執行個體的閉包。您可以對此 Redis 執行個體發出所有指令，這些指令將同時發送到 Redis 伺服器，以減少連線至伺服器的網路往返次數。這些指令仍會按照發出的順序執行：

```php
use Redis;
use Illuminate\Support\Facades;

Facades\Redis::pipeline(function (Redis $pipe) {
    for ($i = 0; $i < 1000; $i++) {
        $pipe->set("key:$i", $i);
    }
});
```


<a name="pubsub"></a>
## 發佈 / 訂閱

Laravel 針對 Redis 的 `publish` 與 `subscribe` 指令提供了便捷的介面。這些 Redis 指令允許您監聽指定「頻道」上的訊息。您可以從另一個應用程式，甚至是使用另一種程式語言發佈訊息到該頻道，從而實現應用程式與行程 (Processes) 之間輕鬆的通訊。

首先，讓我們使用 `subscribe` 方法設定頻道監聽器。我們將這個方法呼叫放置在 [Artisan 指令](/docs/{{version}}/artisan) 中，因為呼叫 `subscribe` 方法會啟動一個長時間執行的行程：

```php
<?php

namespace App\Console\Commands;

use Illuminate\Console\Command;
use Illuminate\Support\Facades\Redis;

class RedisSubscribe extends Command
{
    /**
     * The name and signature of the console command.
     *
     * @var string
     */
    protected $signature = 'redis:subscribe';

    /**
     * The console command description.
     *
     * @var string
     */
    protected $description = 'Subscribe to a Redis channel';

    /**
     * Execute the console command.
     */
    public function handle(): void
    {
        Redis::subscribe(['test-channel'], function (string $message) {
            echo $message;
        });
    }
}
```

現在我們可以透過 `publish` 方法向頻道發佈訊息：

```php
use Illuminate\Support\Facades\Redis;

Route::get('/publish', function () {
    // ...

    Redis::publish('test-channel', json_encode([
        'name' => 'Adam Wathan'
    ]));
});
```


<a name="wildcard-subscriptions"></a>
#### 通配符訂閱

使用 `psubscribe` 方法，您可以訂閱通配符頻道，這對於擷取所有頻道上的所有訊息非常有用。頻道名稱將作為第二個引數傳遞給指定的閉包：

```php
Redis::psubscribe(['*'], function (string $message, string $channel) {
    echo $message;
});

Redis::psubscribe(['users.*'], function (string $message, string $channel) {
    echo $message;
});
```