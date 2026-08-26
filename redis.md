# Redis

- [簡介](#introduction)
- [設定](#configuration)
    - [叢集](#clusters)
    - [Predis](#predis)
    - [PhpRedis](#phpredis)
- [與 Redis 互動](#interacting-with-redis)
    - [交易](#transactions)
    - [管線化指令](#pipelining-commands)
- [發布 / 訂閱](#pubsub)

<a name="introduction"></a>
## 簡介

[Redis](https://redis.io) 是一個開源且先進的鍵值（Key-Value）儲存系統。它通常被稱為資料結構伺服器，因為鍵可以包含[字串 (Strings)](https://redis.io/docs/latest/develop/data-types/strings/)、[雜湊 (Hashes)](https://redis.io/docs/latest/develop/data-types/hashes/)、[列表 (Lists)](https://redis.io/docs/latest/develop/data-types/lists/)、[集合 (Sets)](https://redis.io/docs/latest/develop/data-types/sets/) 以及[有序集合 (Sorted sets)](https://redis.io/docs/latest/develop/data-types/sorted-sets/)。

在將 Redis 與 Laravel 搭配使用之前，我們建議您透過 PECL 安裝並使用 [PhpRedis](https://github.com/phpredis/phpredis) PHP 擴充功能。與使用者層級（User-land）的 PHP 套件相比，該擴充功能的安裝雖然較為複雜，但對於大量使用 Redis 的應用程式來說，可以提供更好的效能。如果您使用的是 [Laravel Sail](/docs/{{version}}/sail)，該擴充功能已經預先安裝在您應用程式的 Docker 容器中了。

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

設定檔中定義的每個 Redis 伺服器都需要有名稱、主機 (host) 與連接埠 (port)，除非你定義了單一 URL 來代表該 Redis 連線：

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

預設情況下，Redis 客戶端在連線至 Redis 伺服器時會使用 `tcp` scheme；不過，你可以在 Redis 伺服器的設定陣列中指定 `scheme` 設定選項，來使用 TLS / SSL 加密：

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

預設情況下，Laravel 會使用原生 Redis 叢集功能，因為 `options.cluster` 設定值設為 `redis`。Redis 叢集是一個優良的預設選項，因為它能妥善處理故障轉移 (Failover)。

使用 Predis 時，Laravel 也支援客戶端分片 (Client-side sharding)。然而，客戶端分片無法處理故障轉移；因此，它主要適用於可從另一個主要資料儲存庫取得的暫存快取資料。

如果你想使用客戶端分片而非原生 Redis 叢集，可以移除應用程式 `config/database.php` 設定檔中的 `options.cluster` 設定值：

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

如果你希望應用程式透過 Predis 套件與 Redis 互動，應該確保 `REDIS_CLIENT` 環境變數的值為 `predis`：

```php
'redis' => [

    'client' => env('REDIS_CLIENT', 'predis'),

    // ...
],
```

除了預設的設定選項外，Predis 還支援可為每個 Redis 伺服器定義的額外[連線參數](https://github.com/nrk/predis/wiki/Connection-Parameters)。若要使用這些額外的設定選項，請將它們新增至應用程式 `config/database.php` 設定檔中的 Redis 伺服器設定：

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

預設情況下，Laravel 會使用 PhpRedis 擴充功能與 Redis 進行通訊。Laravel 用來與 Redis 通訊的用戶端取決於 `redis.client` 設定選項的值，該選項通常反映 `REDIS_CLIENT` 環境變數的值：

```php
'redis' => [

    'client' => env('REDIS_CLIENT', 'phpredis'),

    // ...
],
```

除了預設的設定選項外，PhpRedis 還支援以下額外的連線參數：`name`、`persistent`、`persistent_id`、`prefix`、`read_timeout`、`retry_interval`、`max_retries`、`backoff_algorithm`、`backoff_base`、`backoff_cap`、`timeout` 以及 `context`。你可以將這些選項中的任何一個新增至 `config/database.php` 設定檔中的 Redis 伺服器設定：

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

`retry_interval`、`max_retries`、`backoff_algorithm`、`backoff_base` 與 `backoff_cap` 選項可用於設定 PhpRedis 用戶端應如何嘗試重新連線至 Redis 伺服器。支援以下退避演算法：`default`、`decorrelated_jitter`、`equal_jitter`、`exponential`、`uniform` 和 `constant`：

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

在發生暫時性連線失敗後，Laravel 會自動重試安全的讀取指令一次。你可以使用 `command_retries` 選項來設定所有 Redis 指令的重試次數：

```php
'default' => [
    // ...
    'command_retries' => env('REDIS_COMMAND_RETRIES', 0),
],
```

Predis 3.4.0 及更新版本支援透過 `Retry` 類別進行內建的重試與退避設定。你可以使用 `max_retries` 選項設定重試次數，並使用 `retry` 選項設定退避策略。`retry` 選項應為一個陣列，其鍵值為以下策略類別之一：`NoBackoff`、`EqualBackoff` 或 `ExponentialBackoff`：

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

當在 Redis 叢集中使用 Predis 時，你可以在叢集設定的 `parameters` 選項中定義重試設定：

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

Redis 連線也可以設定為使用 Unix Socket 而非 TCP。這可以透過消除與應用程式同一台伺服器上的 Redis 執行個體連線時的 TCP 開銷，來提供更好的效能。若要將 Redis 設定為使用 Unix Socket，請將 `REDIS_HOST` 環境變數設定為 Redis Socket 的路徑，並將 `REDIS_PORT` 環境變數設定為 `0`：

```env
REDIS_HOST=/run/redis/redis.sock
REDIS_PORT=0
```

<a name="phpredis-serialization"></a>
#### PhpRedis 序列化與壓縮

PhpRedis 擴充功能也可以設定為使用各種序列化器與壓縮演算法。這些演算法可以透過 Redis 設定中的 `options` 陣列進行設定：

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

目前支援的序列化器包含：`Redis::SERIALIZER_NONE`（預設）、`Redis::SERIALIZER_PHP`、`Redis::SERIALIZER_JSON`、`Redis::SERIALIZER_IGBINARY` 以及 `Redis::SERIALIZER_MSGPACK`。

支援的壓縮演算法包含：`Redis::COMPRESSION_NONE`（預設）、`Redis::COMPRESSION_LZF`、`Redis::COMPRESSION_ZSTD` 以及 `Redis::COMPRESSION_LZ4`。

<a name="interacting-with-redis"></a>
## 與 Redis 互動

您可以透過在 `Redis` [Facade](/docs/{{version}}/facades) 上呼叫各種方法來與 Redis 進行互動。`Redis` Facade 支援動態方法，這意味著您可以在該 Facade 上呼叫任何 [Redis 指令](https://redis.io/commands)，且該指令將直接傳遞給 Redis。在此範例中，我們將透過呼叫 `Redis` Facade 上的 `get` 方法來呼叫 Redis 的 `GET` 指令：

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

如上所述，您可以在 `Redis` Facade 上呼叫任何 Redis 的指令。Laravel 會使用魔術方法 (Magic Methods) 將指令傳遞給 Redis 伺服器。如果某個 Redis 指令需要引數，您應該將這些引數傳遞給 Facade 的對應方法：

```php
use Illuminate\Support\Facades\Redis;

Redis::set('name', 'Taylor');

$values = Redis::lrange('names', 5, 10);
```

或者，您可以使用 `Redis` Facade 的 `command` 方法將指令傳遞給伺服器，該方法接受指令名稱作為其第一個引數，並接受一個數值陣列作為其第二個引數：

```php
$values = Redis::command('lrange', ['name', 5, 10]);
```


<a name="using-multiple-redis-connections"></a>
#### 使用多個 Redis 連線

您的應用程式設定檔 `config/database.php` 允許您定義多個 Redis 連線 / 伺服器。您可以使用 `Redis` Facade 的 `connection` 方法來取得特定 Redis 連線的執行個體：

```php
$redis = Redis::connection('connection-name');
```

若要取得預設 Redis 連線的執行個體，您可以在呼叫 `connection` 方法時不傳入任何額外引數：

```php
$redis = Redis::connection();
```


<a name="transactions"></a>
### 交易

`Redis` Facade 的 `transaction` 方法提供了 Redis 原生 `MULTI` 和 `EXEC` 指令的便利封裝。`transaction` 方法接受一個閉包 (Closure) 作為其唯一的引數。此閉包將接收一個 Redis 連線執行個體，並可以對該執行個體發出任何想要的指令。閉包內發出的所有 Redis 指令都將在單一、不可分割的交易 (Atomic Transaction) 中執行：

```php
use Redis;
use Illuminate\Support\Facades;

Facades\Redis::transaction(function (Redis $redis) {
    $redis->incr('user_visits', 1);
    $redis->incr('total_visits', 1);
});
```

> [!WARNING]
> 當定義 Redis 交易時，您無法從 Redis 連線中取得任何數值。請記住，您的交易是作為單一、原子性 (Atomic) 操作執行的，該操作在整個閉包完成執行其指令之前都不會被執行。


#### Lua 腳本

`eval` 方法提供了另一種在單一、原子性操作中執行多個 Redis 指令的方法。然而，`eval` 方法的好處是能夠在該操作期間與 Redis 的 Key 值進行互動並加以檢查。Redis 腳本是用 [Lua 程式語言](https://www.lua.org) 所撰寫。

`eval` 方法一開始可能看起來有點可怕，但我們將透過一個基本範例來破冰說明。`eval` 方法需要傳入多個引數。首先，您應該將 Lua 腳本（作為字串）傳給該方法。其次，您應該傳入該腳本會互動的 Key 數量（作為整數）。第三，您應該傳入這些 Key 的名稱。最後，您可以傳入需要要在腳本中存取的任何其他額外引數。

在這個範例中，我們將遞增一個計數器、檢查其新值，如果第一個計數器的值大於五，則遞增第二個計數器。最後，我們將回傳第一個計數器的值：

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
> 請參考 [Redis 官方文件](https://redis.io/commands/eval) 以取得更多關於 Redis 腳本編寫的詳細資訊。


<a name="pipelining-commands"></a>
### 管線化指令

有時您可能需要執行數十個 Redis 指令。您可以使用 `pipeline` 方法，而不是為每個指令都對 Redis 伺服器進行一次網路傳輸。`pipeline` 方法接受一個引數：一個接收 Redis 執行個體的閉包。您可以向此 Redis 執行個體發出所有指令，這些指令將同時發送到 Redis 伺服器，以減少對伺服器的網路請求次數。這些指令仍將按照發出的順序執行：

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
## 發布 / 訂閱

Laravel 為 Redis 的 `publish` 與 `subscribe` 指令提供了便利的介面。這些 Redis 指令允許您監聽特定「頻道 (Channel)」上的訊息。您可以從另一個應用程式、甚至是使用另一種程式語言向頻道發布訊息，從而輕鬆實現應用程式與行程 (Processes) 之間的溝通。

首先，讓我們使用 `subscribe` 方法設定一個頻道監聽器。我們會將此方法呼叫放置在 [Artisan 命令](/docs/{{version}}/artisan) 中，因為呼叫 `subscribe` 方法會啟動一個長時間執行的行程 (Process)：

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

現在我們可以透過 `publish` 方法向頻道發布訊息：

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

使用 `psubscribe` 方法，您可以訂閱通配符 (Wildcard) 頻道，這對於擷取所有頻道上的所有訊息非常有用。頻道名稱將作為第二個引數傳遞給傳入的閉包：

```php
Redis::psubscribe(['*'], function (string $message, string $channel) {
    echo $message;
});

Redis::psubscribe(['users.*'], function (string $message, string $channel) {
    echo $message;
});
```