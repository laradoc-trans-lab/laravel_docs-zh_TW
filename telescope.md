# Laravel Telescope

- [簡介](#introduction)
- [安裝](#installation)
    - [僅限本地端安裝](#local-only-installation)
    - [設定](#configuration)
    - [資料修剪](#data-pruning)
    - [控制台授權](#dashboard-authorization)
- [升級 Telescope](#upgrading-telescope)
- [篩選](#filtering)
    - [條目](#filtering-entries)
    - [批次](#filtering-batches)
- [標記](#tagging)
- [可用的監聽器](#available-watchers)
    - [批次監聽器](#batch-watcher)
    - [快取監聽器](#cache-watcher)
    - [命令監聽器](#command-watcher)
    - [傾印監聽器](#dump-watcher)
    - [事件監聽器](#event-watcher)
    - [例外監聽器](#exception-watcher)
    - [Gate 監聽器](#gate-watcher)
    - [HTTP 用戶端監聽器](#http-client-watcher)
    - [任務監聽器](#job-watcher)
    - [日誌監聽器](#log-watcher)
    - [郵件監聽器](#mail-watcher)
    - [Model 監聽器](#model-watcher)
    - [通知監聽器](#notification-watcher)
    - [查詢監聽器](#query-watcher)
    - [Redis 監聽器](#redis-watcher)
    - [請求監聽器](#request-watcher)
    - [排程監聽器](#schedule-watcher)
    - [視圖監聽器](#view-watcher)
- [顯示使用者頭像](#displaying-user-avatars)

<a name="introduction"></a>
## 簡介

[Laravel Telescope](https://github.com/laravel/telescope) 是您本地 Laravel 開發環境的絕佳夥伴。Telescope 提供對進入應用程式的請求、例外、日誌條目、資料庫查詢、佇列任務、郵件、通知、快取操作、排程任務、變數傾印等資訊的洞察。

<img src="https://laravel.com/img/docs/telescope-example.png">

<a name="installation"></a>
## 安裝

您可以使用 Composer 套件管理工具將 Telescope 安裝到您的 Laravel 專案中：

```shell
composer require laravel/telescope
```

安裝 Telescope 後，請使用 `telescope:install` Artisan 命令發布其資源和遷移。安裝 Telescope 後，您還應該執行 `migrate` 命令，以建立儲存 Telescope 資料所需的資料表：

```shell
php artisan telescope:install

php artisan migrate
```

最後，您可以透過 `/telescope` 路由存取 Telescope 控制台。

<a name="local-only-installation"></a>
### 僅限本地端安裝

如果您計畫僅在本地開發時使用 Telescope 輔助開發，您可以使用 `--dev` 旗標安裝 Telescope：

```shell
composer require laravel/telescope --dev

php artisan telescope:install

php artisan migrate
```

執行 `telescope:install` 後，您應該從應用程式的 `bootstrap/providers.php` 設定檔中移除 `TelescopeServiceProvider` 服務提供者註冊。相反地，請在 `App\Providers\AppServiceProvider` 類別的 `register` 方法中手動註冊 Telescope 的服務提供者。我們會確保在註冊提供者之前，當前環境是 `local`：

```php
/**
 * Register any application services.
 */
public function register(): void
{
    if ($this->app->environment('local') && class_exists(\Laravel\Telescope\TelescopeServiceProvider::class)) {
        $this->app->register(\Laravel\Telescope\TelescopeServiceProvider::class);
        $this->app->register(TelescopeServiceProvider::class);
    }
}
```

最後，您還應該透過將以下內容新增到 `composer.json` 檔案中，阻止 Telescope 套件被 [自動探索](/docs/{{version}}/packages#package-discovery)：

```json
"extra": {
    "laravel": {
        "dont-discover": [
            "laravel/telescope"
        ]
    }
},
```

<a name="configuration"></a>
### 設定

發布 Telescope 的資源後，其主要設定檔將位於 `config/telescope.php`。此設定檔讓您可以設定您的 [監聽器選項](#available-watchers)。每個設定選項都包含其用途說明，務必仔細探索此檔案。

如有需要，您可以透過 `enabled` 設定選項完全停用 Telescope 的資料收集：

```php
'enabled' => env('TELESCOPE_ENABLED', true),
```

<a name="data-pruning"></a>
### 資料修剪

若不修剪，`telescope_entries` 資料表會非常快速地累積記錄。為緩解此問題，您應該 [排程](/docs/{{version}}/scheduling) `telescope:prune` Artisan 命令每日執行：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('telescope:prune')->daily();
```

依預設，所有超過 24 小時的條目都將被修剪。您可以在呼叫命令時使用 `hours` 選項來決定保留 Telescope 資料多久。例如，下列命令將刪除所有超過 48 小時前建立的記錄：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('telescope:prune --hours=48')->daily();
```

<a name="dashboard-authorization"></a>
### 控制台授權

Telescope 控制台可透過 `/telescope` 路由存取。依預設，您將只能在 `local` 環境中存取此控制台。在您的 `app/Providers/TelescopeServiceProvider.php` 檔案中，有一個 [授權 Gate](/docs/{{version}}/authorization#gates) 定義。此授權 Gate 控制在 **非本地** 環境中對 Telescope 的存取。您可以視需要修改此 Gate，以限制對您 Telescope 安裝的存取：

```php
use App\Models\User;

/**
 * Register the Telescope gate.
 *
 * This gate determines who can access Telescope in non-local environments.
 */
protected function gate(): void
{
    Gate::define('viewTelescope', function (User $user) {
        return in_array($user->email, [
            'taylor@laravel.com',
        ]);
    });
}
```

> [!WARNING]
> 您應確保在您的正式環境中，將 `APP_ENV` 環境變數更改為 `production`。否則，您的 Telescope 安裝將會公開可存取。

<a name="upgrading-telescope"></a>
## 升級 Telescope

升級到新主要版本 Telescope 時，務必仔細審閱 [升級指南](https://github.com/laravel/telescope/blob/master/UPGRADE.md)。

此外，升級到任何新版 Telescope 時，您應該重新發布 Telescope 的資源：

```shell
php artisan telescope:publish
```

為了保持資源最新並避免未來更新時發生問題，您可以將 `vendor:publish --tag=laravel-assets` 命令加入到應用程式 `composer.json` 檔案的 `post-update-cmd` 腳本中：

```json
{
    "scripts": {
        "post-update-cmd": [
            "@php artisan vendor:publish --tag=laravel-assets --ansi --force"
        ]
    }
}
```

<a name="filtering"></a>
## 篩選

<a name="filtering-entries"></a>
### 條目

您可以透過在 `App\Providers\TelescopeServiceProvider` 類別中定義的 `filter` 閉包來篩選由 Telescope 記錄的資料。依預設，此閉包會記錄 `local` 環境中的所有資料，以及所有其他環境中的例外、失敗任務、排程任務，以及帶有監控標籤的資料：

```php
use Laravel\Telescope\IncomingEntry;
use Laravel\Telescope\Telescope;

/**
 * Register any application services.
 */
public function register(): void
{
    $this->hideSensitiveRequestDetails();

    Telescope::filter(function (IncomingEntry $entry) {
        if ($this->app->environment('local')) {
            return true;
        }

        return $entry->isReportableException() ||
            $entry->isFailedJob() ||
            $entry->isScheduledTask() ||
            $entry->isSlowQuery() ||
            $entry->hasMonitoredTag();
    });
}
```

<a name="filtering-batches"></a>
### 批次

雖然 `filter` 閉包會篩選個別條目的資料，但您可以使用 `filterBatch` 方法註冊一個閉包，用於篩選指定請求或控制台命令的所有資料。如果閉包回傳 `true`，所有條目都將由 Telescope 記錄：

```php
use Illuminate\Support\Collection;
use Laravel\Telescope\IncomingEntry;
use Laravel\Telescope\Telescope;

/**
 * Register any application services.
 */
public function register(): void
{
    $this->hideSensitiveRequestDetails();

    Telescope::filterBatch(function (Collection $entries) {
        if ($this->app->environment('local')) {
            return true;
        }

        return $entries->contains(function (IncomingEntry $entry) {
            return $entry->isReportableException() ||
                $entry->isFailedJob() ||
                $entry->isScheduledTask() ||
                $entry->isSlowQuery() ||
                $entry->hasMonitoredTag();
            });
    });
}
```

<a name="tagging"></a>
## 標記

Telescope 允許你依據「標籤」來搜尋條目。通常，標籤會是 Eloquent Model 類別名稱或已驗證的使用者 ID，Telescope 會自動將其新增到條目中。偶爾，你可能想要為條目附加自訂標籤。為此，你可以使用 `Telescope::tag` 方法。此 `tag` 方法接受一個閉包，該閉包應回傳一個標籤陣列。該閉包回傳的標籤將會與 Telescope 自動附加到條目的任何標籤合併。通常，你應該在 `App\Providers\TelescopeServiceProvider` 類別的 `register` 方法中呼叫 `tag` 方法：

```php
use Laravel\Telescope\IncomingEntry;
use Laravel\Telescope\Telescope;

/**
 * Register any application services.
 */
public function register(): void
{
    $this->hideSensitiveRequestDetails();

    Telescope::tag(function (IncomingEntry $entry) {
        return $entry->type === 'request'
            ? ['status:'.$entry->content['response_status']]
            : [];
    });
}
```

<a name="available-watchers"></a>
## 可用的監聽器

當請求或主控台命令執行時，Telescope 的「監聽器 (watchers)」會收集應用程式資料。你可以在 `config/telescope.php` 設定檔中自訂要啟用哪些監聽器：

```php
'watchers' => [
    Watchers\CacheWatcher::class => true,
    Watchers\CommandWatcher::class => true,
    // ...
],
```

有些監聽器也允許你提供額外的自訂選項：

```php
'watchers' => [
    Watchers\QueryWatcher::class => [
        'enabled' => env('TELESCOPE_QUERY_WATCHER', true),
        'slow' => 100,
    ],
    // ...
],
```

<a name="batch-watcher"></a>
### 批次監聽器

批次監聽器會記錄有關佇列中 [批次](/docs/{{version}}/queues#job-batching) 的資訊，包括任務與連線資訊。

<a name="cache-watcher"></a>
### 快取監聽器

快取監聽器會在快取鍵被命中 (hit)、錯失 (missed)、更新 (updated) 和清除 (forgotten) 時記錄資料。

<a name="command-watcher"></a>
### 命令監聽器

命令監聽器會記錄 Artisan 命令執行時的參數、選項、結束碼與輸出。如果你想要排除某些命令不被監聽器記錄，你可以在 `config/telescope.php` 檔案中的 `ignore` 選項裡指定該命令：

```php
'watchers' => [
    Watchers\CommandWatcher::class => [
        'enabled' => env('TELESCOPE_COMMAND_WATCHER', true),
        'ignore' => ['key:generate'],
    ],
    // ...
],
```

<a name="dump-watcher"></a>
### 傾印監聽器

傾印監聽器會記錄並在 Telescope 中顯示你的變數傾印 (variable dumps)。在使用 Laravel 時，可以使用全域 `dump` 函式來傾印變數。傾印監聽器分頁必須在瀏覽器中開啟才能記錄傾印，否則監聽器將會忽略這些傾印。

<a name="event-watcher"></a>
### 事件監聽器

事件監聽器會記錄應用程式分派 (dispatched) 的任何 [事件](/docs/{{version}}/events) 的負載 (payload)、監聽器與廣播資料。Laravel 框架的內部事件會被 Event 監聽器忽略。

<a name="exception-watcher"></a>
### 例外監聽器

例外監聽器會記錄應用程式拋出 (thrown) 的任何可回報 (reportable) 例外 (exceptions) 的資料與堆疊追蹤 (stack trace)。

<a name="gate-watcher"></a>
### Gate 監聽器

Gate 監聽器會記錄應用程式 [Gate 和 Policy](/docs/{{version}}/authorization) 檢查的資料與結果。如果你想要排除某些權限 (abilities) 不被監聽器記錄，你可以在 `config/telescope.php` 檔案中的 `ignore_abilities` 選項裡指定這些權限：

```php
'watchers' => [
    Watchers\GateWatcher::class => [
        'enabled' => env('TELESCOPE_GATE_WATCHER', true),
        'ignore_abilities' => ['viewNova'],
    ],
    // ...
],
```

<a name="http-client-watcher"></a>
### HTTP 用戶端監聽器

HTTP 用戶端監聽器會記錄應用程式發出的 [HTTP 用戶端請求](/docs/{{version}}/http-client)。

<a name="job-watcher"></a>
### 任務監聽器

任務監聽器會記錄應用程式分派 (dispatched) 的任何 [任務](/docs/{{version}}/queues) 的資料與狀態。

<a name="log-watcher"></a>
### 日誌監聽器

日誌監聽器會記錄應用程式寫入的任何 [日誌資料](/docs/{{version}}/logging)。

依預設，Telescope 只會記錄 `error` 等級及以上的日誌。然而，你可以修改應用程式 `config/telescope.php` 設定檔中的 `level` 選項來修改此行為：

```php
'watchers' => [
    Watchers\LogWatcher::class => [
        'enabled' => env('TELESCOPE_LOG_WATCHER', true),
        'level' => 'debug',
    ],

    // ...
],
```

<a name="mail-watcher"></a>
### 郵件監聽器

郵件監聽器允許你在瀏覽器中預覽應用程式傳送的 [電子郵件](/docs/{{version}}/mail) 及其相關資料。你也可以將電子郵件下載為 `.eml` 檔案。

<a name="model-watcher"></a>
### Model 監聽器

Model 監聽器會記錄任何 Eloquent [Model 事件](/docs/{{version}}/eloquent#events) 分派 (dispatched) 時的 Model 變更。你可以透過監聽器的 `events` 選項來指定應記錄哪些 Model 事件：

```php
'watchers' => [
    Watchers\ModelWatcher::class => [
        'enabled' => env('TELESCOPE_MODEL_WATCHER', true),
        'events' => ['eloquent.created*', 'eloquent.updated*'],
    ],
    // ...
],
```

如果你想要記錄在指定請求期間「水合 (hydrated)」的 Model 數量，請啟用 `hydrations` 選項：

```php
'watchers' => [
    Watchers\ModelWatcher::class => [
        'enabled' => env('TELESCOPE_MODEL_WATCHER', true),
        'events' => ['eloquent.created*', 'eloquent.updated*'],
        'hydrations' => true,
    ],
    // ...
],
```

<a name="notification-watcher"></a>
### 通知監聽器

通知監聽器會記錄應用程式傳送的所有 [通知](/docs/{{version}}/notifications)。如果通知觸發了電子郵件，並且你啟用了郵件監聽器，那麼該電子郵件也將可以在郵件監聽器螢幕上預覽。

<a name="query-watcher"></a>
### 查詢監聽器

查詢監聽器會記錄應用程式執行之所有查詢的原始 SQL、綁定 (bindings) 與執行時間。監聽器也會將任何執行時間慢於 100 毫秒的查詢標記為 `slow`。你可以使用監聽器的 `slow` 選項來自訂慢速查詢的閾值 (threshold)：

```php
'watchers' => [
    Watchers\QueryWatcher::class => [
        'enabled' => env('TELESCOPE_QUERY_WATCHER', true),
        'slow' => 50,
    ],
    // ...
],
```

<a name="redis-watcher"></a>
### Redis 監聽器

Redis 監聽器會記錄應用程式執行之所有 [Redis](/docs/{{version}}/redis) 命令。如果你將 Redis 用於快取，那麼快取命令也會被 Redis 監聽器記錄。

<a name="request-watcher"></a>
### 請求監聽器

請求監聽器會記錄應用程式處理的任何請求的請求、標頭、Session 與回應資料。你可以透過 `size_limit` (以 KB 為單位) 選項來限制記錄的回應資料大小：

```php
'watchers' => [
    Watchers\RequestWatcher::class => [
        'enabled' => env('TELESCOPE_REQUEST_WATCHER', true),
        'size_limit' => env('TELESCOPE_RESPONSE_SIZE_LIMIT', 64),
    ],
    // ...
],
```

<a name="schedule-watcher"></a>
### 排程監聽器

排程監聽器會記錄應用程式執行之任何 [排程任務](/docs/{{version}}/scheduling) 的命令與輸出。

<a name="view-watcher"></a>
### 視圖監聽器

視圖監聽器會記錄渲染 (rendering) 視圖時所使用的 [視圖](/docs/{{version}}/views) 名稱、路徑、資料與「Composer」。

<a name="displaying-user-avatars"></a>
## 顯示使用者頭像

Telescope 控制台會顯示在儲存了特定條目時，已認證使用者的頭像。預設情況下，Telescope 會使用 Gravatar 網路服務來取得頭像。不過，您可以在 `App\Providers\TelescopeServiceProvider` 類別中註冊一個回呼函式來自訂頭像的 URL。此回呼函式將會接收到使用者的 ID 與電子郵件地址，並應回傳使用者頭像的圖片 URL：

```php
use App\Models\User;
use Laravel\Telescope\Telescope;

/**
 * Register any application services.
 */
public function register(): void
{
    // ...

    Telescope::avatar(function (?string $id, ?string $email) {
        return ! is_null($id)
            ? '/avatars/'.User::find($id)->avatar_path
            : '/generic-avatar.jpg';
    });
}
```