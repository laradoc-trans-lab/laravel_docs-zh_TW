# Redis

- [介紹](#introduction)
- [設定](#configuration)
    - [叢集](#clusters)
    - [Predis](#predis)
    - [PhpRedis](#phpredis)
- [與 Redis 互動](#interacting-with-redis)
    - [交易](#transactions)
    - [管道化指令](#pipelining-commands)
- [發佈 / 訂閱](#pubsub)

<a name="introduction"></a>
## 介紹

[Redis](https://redis.io) 是一個開源、先進的鍵值儲存資料庫。它通常被稱為資料結構伺服器，因為鍵可以包含[字串](https://redis.io/docs/latest/develop/data-types/strings/)、[雜湊](https://redis.io/docs/latest/develop/data-types/hashes/)、[列表](https://redis.io/docs/latest/develop/data-types/lists/)、[集合](https://redis.io/docs/latest/develop/data-types/sets/)以及[有序集合](https://redis.io/docs/latest/develop/data-types/sorted-sets/)。

在 Laravel 中使用 Redis 之前，我們建議您透過 PECL 安裝並使用 [PhpRedis](https://github.com/phpredis/phpredis) PHP 擴充套件。相較於「使用者空間」的 PHP 套件，此擴充套件安裝起來更為複雜，但對於大量使用 Redis 的應用程式，它可能會帶來更好的效能。如果您正在使用 [Laravel Sail](/docs/{{version}}/sail)，則此擴充套件已安裝於您的應用程式 Docker 容器中。

如果您無法安裝 PhpRedis 擴充套件，您可以透過 Composer 安裝 `predis/predis` 套件。Predis 是一個完全以 PHP 編寫的 Redis 用戶端，且不需要任何額外的擴充套件：

```shell
composer require predis/predis
```

<a name="configuration"></a>
## 設定

你可以透過 `config/database.php` 設定檔來設定應用程式的 Redis 設定。在此檔案中，你會看到一個 `redis` 陣列，其中包含你的應用程式所使用的 Redis 伺服器：

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

在你的設定檔中定義的每個 Redis 伺服器都必須有名稱、主機與連接埠，除非你定義一個單一的 URL 來代表 Redis 連線：

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
#### 設定連線機制

預設情況下，Redis 用戶端在連線到 Redis 伺服器時會使用 `tcp` 機制；但是，你可以透過在 Redis 伺服器的設定陣列中指定 `scheme` 設定選項來使用 TLS / SSL 加密：

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

如果你的應用程式正在使用 Redis 伺服器叢集，你應該在 Redis 設定的 `clusters` 鍵中定義這些叢集。此設定鍵預設不存在，因此你需要它在應用程式的 `config/database.php` 設定檔中建立它：

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

預設情況下，Laravel 會使用原生的 Redis 叢集功能，因為 `options.cluster` 設定值設定為 `redis`。Redis 叢集功能是一個很棒的預設選項，因為它能妥善處理容錯移轉。

當使用 Predis 時，Laravel 也支援用戶端分片。但是，用戶端分片不處理容錯移轉；因此，它主要適用於可以從其他主要資料儲存庫取得的暫存快取資料。

如果你想使用用戶端分片而不是原生的 Redis 叢集功能，你可以移除應用程式 `config/database.php` 設定檔中的 `options.cluster` 設定值：

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

如果你希望應用程式透過 Predis 套件與 Redis 互動，你應該確保 `REDIS_CLIENT` 環境變數的值為 `predis`：

```php
'redis' => [

    'client' => env('REDIS_CLIENT', 'predis'),

    // ...
],
```

除了預設的設定選項外，Predis 還支援可以在每個 Redis 伺服器上定義的額外 [連線參數](https://github.com/nrk/predis/wiki/Connection-Parameters)。要利用這些額外的設定選項，請將它們新增到應用程式 `config/database.php` 設定檔中的 Redis 伺服器設定裡：

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

預設情況下，Laravel 將會使用 PhpRedis 擴充來與 Redis 溝通。Laravel 將會使用的 Redis 用戶端是由 `redis.client` 設定選項的值來決定，這個值通常反映了 `REDIS_CLIENT` 環境變數的值：

```php
'redis' => [

    'client' => env('REDIS_CLIENT', 'phpredis'),

    // ...
],
```

除了預設的設定選項外，PhpRedis 還支援以下額外的連線參數：`name`、`persistent`、`persistent_id`、`prefix`、`read_timeout`、`retry_interval`、`max_retries`、`backoff_algorithm`、`backoff_base`、`backoff_cap`、`timeout` 以及 `context`。您可以將這些選項中的任何一個新增至應用程式 `config/database.php` 設定檔中的 Redis 伺服器設定。

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

`retry_interval`、`max_retries`、`backoff_algorithm`、`backoff_base` 和 `backoff_cap` 選項可用於設定 PhpRedis 用戶端應如何嘗試重新連線到 Redis 伺服器。支援以下退避演算法：`default`、`decorrelated_jitter`、`equal_jitter`、`exponential`、`uniform` 以及 `constant`：

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


<a name="unix-socket-connections"></a>
#### Unix Socket 連線

Redis 連線也可以設定為使用 Unix sockets 而非 TCP。這可以透過消除在應用程式所在同一伺服器上的 Redis 實例連線的 TCP 開銷，來提高效能。要將 Redis 設定為使用 Unix socket，請將 `REDIS_HOST` 環境變數設定為 Redis socket 的路徑，並將 `REDIS_PORT` 環境變數設定為 `0`：

```env
REDIS_HOST=/run/redis/redis.sock
REDIS_PORT=0
```


<a name="phpredis-serialization"></a>
#### PhpRedis 序列化與壓縮

PhpRedis 擴充也可以設定為使用各種序列化器和壓縮演算法。這些演算法可以透過 Redis 設定中的 `options` 陣列來進行設定：

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

目前支援的序列化器包括：`Redis::SERIALIZER_NONE` (預設)、`Redis::SERIALIZER_PHP`、`Redis::SERIALIZER_JSON`、`Redis::SERIALIZER_IGBINARY` 和 `Redis::SERIALIZER_MSGPACK`。

支援的壓縮演算法包括：`Redis::COMPRESSION_NONE` (預設)、`Redis::COMPRESSION_LZF`、`Redis::COMPRESSION_ZSTD` 和 `Redis::COMPRESSION_LZ4`。

<a name="interacting-with-redis"></a>
## 與 Redis 互動

您可以透過呼叫 `Redis` [Facade](/docs/{{version}}/facades) 上的各種方法來與 Redis 互動。`Redis` Facade 支援動態方法，這表示您可以在 Facade 上呼叫任何 [Redis 指令](https://redis.io/commands)，該指令將直接傳遞給 Redis。在此範例中，我們將透過呼叫 `Redis` Facade 上的 `get` 方法來呼叫 Redis 的 `GET` 指令：

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

如上所述，您可以在 `Redis` Facade 上呼叫任何 Redis 指令。Laravel 使用魔術方法將指令傳遞給 Redis 伺服器。如果 Redis 指令需要引數，您應該將這些引數傳遞給 Facade 對應的方法：

```php
use Illuminate\Support\Facades\Redis;

Redis::set('name', 'Taylor');

$values = Redis::lrange('names', 5, 10);
```

或者，您可以使用 `Redis` Facade 的 `command` 方法將指令傳遞給伺服器，該方法接受指令名稱作為其第一個引數，並接受一個值陣列作為其第二個引數：

```php
$values = Redis::command('lrange', ['name', 5, 10]);
```


<a name="using-multiple-redis-connections"></a>
#### 使用多個 Redis 連線

您應用程式的 `config/database.php` 設定檔允許您定義多個 Redis 連線 / 伺服器。您可以使用 `Redis` Facade 的 `connection` 方法取得特定 Redis 連線的實例：

```php
$redis = Redis::connection('connection-name');
```

若要取得預設 Redis 連線的實例，您可以呼叫 `connection` 方法而無需任何額外引數：

```php
$redis = Redis::connection();
```


<a name="transactions"></a>
### 交易

`Redis` Facade 的 `transaction` 方法為 Redis 原生的 `MULTI` 和 `EXEC` 指令提供了方便的封裝。`transaction` 方法接受一個閉包作為其唯一的引數。此閉包將接收一個 Redis 連線實例，並可以向此實例發出任何指令。閉包內發出的所有 Redis 指令都將在單一、原子化的交易中執行：

```php
use Redis;
use Illuminate\Support\Facades;

Facades\Redis::transaction(function (Redis $redis) {
    $redis->incr('user_visits', 1);
    $redis->incr('total_visits', 1);
});
```

> [!WARNING]
> 定義 Redis 交易時，您可能無法從 Redis 連線中檢索任何值。請記住，您的交易是作為一個單一的原子操作執行的，並且該操作直到您的整個閉包完成執行其指令後才會執行。


#### Lua 腳本

`eval` 方法提供了另一種以單一、原子化操作執行多個 Redis 指令的方法。然而，`eval` 方法的好處是能夠在該操作期間與 Redis 鍵值進行互動和檢查。Redis 腳本是使用 [Lua 程式語言](https://www.lua.org) 編寫的。

`eval` 方法一開始可能有些嚇人，但我們會透過一個基本範例來打破僵局。`eval` 方法需要多個引數。首先，您應該將 Lua 腳本（作為字串）傳遞給該方法。其次，您應該傳遞腳本互動的鍵的數量（作為整數）。第三，您應該傳遞這些鍵的名稱。最後，您可以傳遞在腳本中需要存取的任何其他額外引數。

在此範例中，我們將遞增一個計數器，檢查其新值，並在第一個計數器值大於五時遞增第二個計數器。最後，我們將返回第一個計數器的值：

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
> 請參閱 [Redis 文件](https://redis.io/commands/eval) 以獲取有關 Redis 腳本的更多資訊。


<a name="pipelining-commands"></a>
### 管道化指令

有時您可能需要執行數十個 Redis 指令。您可以改用 `pipeline` 方法，而不是為每個指令向 Redis 伺服器發出一次網路請求。`pipeline` 方法接受一個引數：一個接收 Redis 實例的閉包。您可以向此 Redis 實例發出所有指令，它們將同時發送到 Redis 伺服器，以減少到伺服器的網路請求次數。這些指令仍將按照發出順序執行：

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

Laravel 為 Redis 的 `publish` 和 `subscribe` 指令提供了方便的介面。這些 Redis 指令允許您在給定的「頻道」上監聽訊息。您可以從另一個應用程式發佈訊息到頻道，甚至可以使用另一種程式語言，從而實現應用程式和程序之間的輕鬆通訊。

首先，讓我們使用 `subscribe` 方法設定一個頻道監聽器。我們將把這個方法呼叫放在一個 [Artisan command](/docs/{{version}}/artisan) 中，因為呼叫 `subscribe` 方法會啟動一個長時間執行的程序：

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

現在我們可以使用 `publish` 方法向頻道發佈訊息：

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
#### 萬用字元訂閱

使用 `psubscribe` 方法，您可以訂閱一個萬用字元頻道，這對於捕捉所有頻道上的所有訊息可能很有用。頻道名稱將作為第二個引數傳遞給提供的閉包：

```php
Redis::psubscribe(['*'], function (string $message, string $channel) {
    echo $message;
});

Redis::psubscribe(['users.*'], function (string $message, string $channel) {
    echo $message;
});
```