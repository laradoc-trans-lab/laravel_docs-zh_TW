# Redis

- [介紹](#introduction)
- [配置](#configuration)
    - [叢集](#clusters)
    - [Predis](#predis)
    - [PhpRedis](#phpredis)
- [與 Redis 互動](#interacting-with-redis)
    - [交易](#transactions)
    - [管道指令](#pipelining-commands)
- [發布 / 訂閱](#pubsub)

<a name="introduction"></a>
## 介紹

[Redis](https://redis.io) 是一個開源的進階鍵值資料庫。它常被稱為資料結構伺服器，因為鍵可以包含 [字串](https://redis.io/docs/latest/develop/data-types/strings/)、[雜湊](https://redis.io/docs/latest/develop/data-types/hashes/)、[列表](https://redis.io/docs/latest/develop/data-types/lists/)、[集合](https://redis.io/docs/latest/develop/data-types/sets/) 和 [有序集合](https://redis.io/docs/latest/develop/data-types/sorted-sets/)。

在使用 Redis 與 Laravel 之前，我們鼓勵您透過 PECL 安裝並使用 [PhpRedis](https://github.com/phpredis/phpredis) PHP 擴充功能。與「使用者空間」PHP 套件相比，此擴充功能的安裝較為複雜，但對於大量使用 Redis 的應用程式來說，它可能會帶來更好的效能。如果您正在使用 [Laravel Sail](/docs/{{version}}/sail)，此擴充功能已預先安裝在您應用程式的 Docker 容器中。

如果您無法安裝 PhpRedis 擴充功能，您可以透過 Composer 安裝 `predis/predis` 套件。Predis 是一個完全以 PHP 編寫的 Redis 用戶端，且不需要任何額外的擴充功能。

```shell
composer require predis/predis:^2.0
```

<a name="configuration"></a>
## 配置

您可以透過 `config/database.php` 配置檔來配置應用程式的 Redis 設定。在此檔案中，您會看到一個 `redis` 陣列，其中包含了您的應用程式所使用的 Redis 伺服器：

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

您配置檔中定義的每個 Redis 伺服器都必須包含名稱、主機與連接埠，除非您定義單一 URL 來表示該 Redis 連線：

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

<a name="configuring-the-connection-scheme"></a>
#### 配置連線方案

預設情況下，Redis 客戶端在連線到您的 Redis 伺服器時會使用 `tcp` 方案；不過，您可以透過在 Redis 伺服器配置陣列中指定 `scheme` 配置選項來使用 TLS / SSL 加密：

    'default' => [
        'scheme' => 'tls',
        'url' => env('REDIS_URL'),
        'host' => env('REDIS_HOST', '127.0.0.1'),
        'username' => env('REDIS_USERNAME'),
        'password' => env('REDIS_PASSWORD'),
        'port' => env('REDIS_PORT', '6379'),
        'database' => env('REDIS_DB', '0'),
    ],

<a name="clusters"></a>
### 叢集

如果您的應用程式正在使用 Redis 伺服器叢集，您應該在 Redis 配置的 `clusters` 鍵中定義這些叢集。此配置鍵預設不存在，因此您需要在應用程式的 `config/database.php` 配置檔中建立它：

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

預設情況下，由於 `options.cluster` 配置值設定為 `redis`，Laravel 會使用原生的 Redis 叢集。Redis 叢集是一個很棒的預設選項，因為它能優雅地處理故障轉移。

當使用 Predis 時，Laravel 也支援客戶端分片。不過，客戶端分片不處理故障轉移；因此，它主要適用於可從其他主要資料儲存區取得的暫存快取資料。

如果您想使用客戶端分片而非原生的 Redis 叢集，您可以從應用程式的 `config/database.php` 配置檔中移除 `options.cluster` 配置值：

    'redis' => [

        'client' => env('REDIS_CLIENT', 'phpredis'),

        'clusters' => [
            // ...
        ],

        // ...
    ],

<a name="predis"></a>
### Predis

如果您希望您的應用程式透過 Predis 套件與 Redis 互動，您應該確保 `REDIS_CLIENT` 環境變數的值為 `predis`：

    'redis' => [

        'client' => env('REDIS_CLIENT', 'predis'),

        // ...
    ],

除了預設的配置選項外，Predis 還支援額外的 [連線參數](https://github.com/nrk/predis/wiki/Connection-Parameters)，這些參數可以為您的每個 Redis 伺服器定義。要使用這些額外的配置選項，請將它們添加到應用程式的 `config/database.php` 配置檔中的 Redis 伺服器配置：

    'default' => [
        'url' => env('REDIS_URL'),
        'host' => env('REDIS_HOST', '127.0.0.1'),
        'username' => env('REDIS_USERNAME'),
        'password' => env('REDIS_PASSWORD'),
        'port' => env('REDIS_PORT', '6379'),
        'database' => env('REDIS_DB', '0'),
        'read_write_timeout' => 60,
    ],

<a name="phpredis"></a>
### PhpRedis

預設情況下，Laravel 將使用 PhpRedis 擴充功能與 Redis 進行通訊。Laravel 將用於與 Redis 通訊的客戶端是由 `redis.client` 配置選項的值決定，此值通常反映 `REDIS_CLIENT` 環境變數的值：

    'redis' => [

        'client' => env('REDIS_CLIENT', 'phpredis'),

        // ...
    ],

除了預設的配置選項外，PhpRedis 還支援以下額外的連線參數：`name`、`persistent`、`persistent_id`、`prefix`、`read_timeout`、`retry_interval`、`max_retries`、`backoff_algorithm`、`backoff_base`、`backoff_cap`、`timeout` 和 `context`。您可以將其中任何一個選項新增至應用程式 `config/database.php` 配置檔中的 Redis 伺服器配置：

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

<a name="phpredis-serialization"></a>
#### PhpRedis 序列化與壓縮

PhpRedis 擴充功能也可以配置為使用各種序列化器和壓縮演算法。這些演算法可以透過 Redis 配置的 `options` 陣列進行配置：

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

目前支援的序列化器包括：`Redis::SERIALIZER_NONE` (預設)、`Redis::SERIALIZER_PHP`、`Redis::SERIALIZER_JSON`、`Redis::SERIALIZER_IGBINARY` 和 `Redis::SERIALIZER_MSGPACK`。

支援的壓縮演算法包括：`Redis::COMPRESSION_NONE` (預設)、`Redis::COMPRESSION_LZF`、`Redis::COMPRESSION_ZSTD` 和 `Redis::COMPRESSION_LZ4`。

<a name="interacting-with-redis"></a>
## 與 Redis 互動

您可以透過呼叫 `Redis` [Facade](/docs/{{version}}/facades) 上的各種方法來與 Redis 互動。`Redis` Facade 支援動態方法，這表示您可以在 Facade 上呼叫任何 [Redis 指令](https://redis.io/commands)，該指令將直接傳遞給 Redis。在此範例中，我們將透過呼叫 `Redis` Facade 上的 `get` 方法來執行 Redis `GET` 指令：

    <?php

    namespace App\Http\Controllers;

    use App\Http\Controllers\Controller;
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

如上所述，您可以在 `Redis` Facade 上呼叫任何 Redis 指令。Laravel 使用魔術方法 (magic methods) 將指令傳遞給 Redis 伺服器。如果 Redis 指令需要引數，您應該將這些引數傳遞給 Facade 對應的方法：

    use Illuminate\Support\Facades\Redis;

    Redis::set('name', 'Taylor');

    $values = Redis::lrange('names', 5, 10);

或者，您可以使用 `Redis` Facade 的 `command` 方法將指令傳遞給伺服器，該方法接受指令名稱作為其第一個引數，並接受一個值陣列作為其第二個引數：

    $values = Redis::command('lrange', ['name', 5, 10]);


<a name="using-multiple-redis-connections"></a>
#### 使用多個 Redis 連線

您應用程式的 `config/database.php` 設定檔允許您定義多個 Redis 連線/伺服器。您可以使用 `Redis` Facade 的 `connection` 方法來取得特定 Redis 連線：

    $redis = Redis::connection('connection-name');

要取得預設 Redis 連線的實例，您可以不帶任何額外引數呼叫 `connection` 方法：

    $redis = Redis::connection();


<a name="transactions"></a>
### 交易

`Redis` Facade 的 `transaction` 方法為 Redis 原生 `MULTI` 和 `EXEC` 指令提供了方便的包裝。`transaction` 方法接受一個閉包作為其唯一引數。此閉包將接收一個 Redis 連線實例，並可以向此實例發出任何指令。閉包內發出的所有 Redis 指令都將以單一的原子交易執行：

    use Redis;
    use Illuminate\Support\Facades;

    Facades\Redis::transaction(function (Redis $redis) {
        $redis->incr('user_visits', 1);
        $redis->incr('total_visits', 1);
    });

> [!WARNING]  
> 定義 Redis 交易時，您不能從 Redis 連線中檢索任何值。請記住，您的交易是作為單一原子操作執行的，並且該操作在您的整個閉包完成執行其指令之前不會執行。


#### Lua 腳本

`eval` 方法提供了另一種以單一原子操作執行多個 Redis 指令的方法。然而，`eval` 方法的優點是能夠在該操作期間與 Redis 鍵值進行互動和檢查。Redis 腳本是用 [Lua 程式語言](https://www.lua.org) 編寫的。

`eval` 方法一開始可能有點嚇人，但我們將探索一個基本範例來打破僵局。`eval` 方法需要多個引數。首先，您應該將 Lua 腳本（作為字串）傳遞給該方法。其次，您應該傳遞腳本互動的鍵的數量（作為整數）。第三，您應該傳遞這些鍵的名稱。最後，您可以傳遞在腳本中需要存取的任何其他額外引數。

在此範例中，我們將遞增一個計數器，檢查其新值，如果第一個計數器的值大於五，則遞增第二個計數器。最後，我們將返回第一個計數器的值：

    $value = Redis::eval(<<<'LUA'
        local counter = redis.call("incr", KEYS[1])

        if counter > 5 then
            redis.call("incr", KEYS[2])
        end

        return counter
    LUA, 2, 'first-counter', 'second-counter');

> [!WARNING]  
> 有關 Redis 腳本的更多資訊，請查閱 [Redis 文件](https://redis.io/commands/eval)。


<a name="pipelining-commands"></a>
### 管道指令

有時您可能需要執行數十個 Redis 指令。您可以改用 `pipeline` 方法，而不是為每個指令向 Redis 伺服器發出網路請求。`pipeline` 方法接受一個引數：一個接收 Redis 實例的閉包。您可以向此 Redis 實例發出所有指令，並且所有指令將同時傳送到 Redis 伺服器，以減少對伺服器的網路請求。這些指令仍將按照發出它們的順序執行：

    use Redis;
    use Illuminate\Support\Facades;

    Facades\Redis::pipeline(function (Redis $pipe) {
        for ($i = 0; $i < 1000; $i++) {
            $pipe->set("key:$i", $i);
        }
    });


<a name="pubsub"></a>
## 發布 / 訂閱

Laravel 為 Redis 的 `publish` 和 `subscribe` 指令提供了一個方便的介面。這些 Redis 指令允許您在給定的「channel (頻道)」上監聽訊息。您可以從另一個應用程式發布訊息到頻道，甚至可以使用另一種程式語言，從而實現應用程式和流程之間的輕鬆通訊。

首先，讓我們使用 `subscribe` 方法設定一個頻道監聽器。我們將把這個方法呼叫放在一個 [Artisan command](/docs/{{version}}/artisan) 中，因為呼叫 `subscribe` 方法會啟動一個長時間執行的流程：

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

現在我們可以使用 `publish` 方法向頻道發布訊息：

    use Illuminate\Support\Facades\Redis;

    Route::get('/publish', function () {
        // ...

        Redis::publish('test-channel', json_encode([
            'name' => 'Adam Wathan'
        ]));
    });


<a name="wildcard-subscriptions"></a>
#### 萬用字元訂閱

使用 `psubscribe` 方法，您可以訂閱一個萬用字元頻道，這對於捕捉所有頻道上的所有訊息可能很有用。頻道名稱將作為第二個引數傳遞給提供的閉包：

    Redis::psubscribe(['*'], function (string $message, string $channel) {
        echo $message;
    });

    Redis::psubscribe(['users.*'], function (string $message, string $channel) {
        echo $message;
    });