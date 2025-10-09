# Laravel Telescope

- [簡介](#introduction)
- [安裝](#installation)
    - [僅限本地安裝](#local-only-installation)
    - [設定](#configuration)
    - [資料修剪](#data-pruning)
    - [儀表板授權](#dashboard-authorization)
- [升級 Telescope](#upgrading-telescope)
- [過濾](#filtering)
    - [條目](#filtering-entries)
    - [批次](#filtering-batches)
- [標記](#tagging)
- [可用 Watchers](#available-watchers)
    - [批次 Watcher](#batch-watcher)
    - [快取 Watcher](#cache-watcher)
    - [命令 Watcher](#command-watcher)
    - [Dump Watcher](#dump-watcher)
    - [事件 Watcher](#event-watcher)
    - [例外 Watcher](#exception-watcher)
    - [Gate Watcher](#gate-watcher)
    - [HTTP 客戶端 Watcher](#http-client-watcher)
    - [任務 Watcher](#job-watcher)
    - [日誌 Watcher](#log-watcher)
    - [郵件 Watcher](#mail-watcher)
    - [Model Watcher](#model-watcher)
    - [通知 Watcher](#notification-watcher)
    - [查詢 Watcher](#query-watcher)
    - [Redis Watcher](#redis-watcher)
    - [請求 Watcher](#request-watcher)
    - [排程 Watcher](#schedule-watcher)
    - [視圖 Watcher](#view-watcher)
- [顯示使用者頭像](#displaying-user-avatars)

<a name="introduction"></a>
## 簡介

[Laravel Telescope](https://github.com/laravel/telescope) 是您本地 Laravel 開發環境的絕佳夥伴。Telescope 提供了對應用程式接收的請求、例外、日誌條目、資料庫查詢、佇列任務 (queued jobs)、郵件、通知、快取操作、排程任務、變數傾印 (variable dumps) 等的深入洞察。

<img src="https://laravel.com/img/docs/telescope-example.png">

<a name="installation"></a>
## 安裝

您可以使用 Composer 套件管理工具將 Telescope 安裝到您的 Laravel 專案中：

```shell
composer require laravel/telescope
```

安裝 Telescope 後，使用 `telescope:install` Artisan 命令發佈其資源和遷移。安裝 Telescope 後，您還應該執行 `migrate` 命令以建立儲存 Telescope 資料所需的資料表：

```shell
php artisan telescope:install

php artisan migrate
```

最後，您可以透過 `/telescope` 路由存取 Telescope 儀表板。

<a name="local-only-installation"></a>
### 僅限本地安裝

如果您計畫僅將 Telescope 用於輔助本地開發，您可以使用 `--dev` 旗標安裝 Telescope：

```shell
composer require laravel/telescope --dev

php artisan telescope:install

php artisan migrate
```

執行 `telescope:install` 後，您應該從應用程式的 `bootstrap/providers.php` 設定檔中移除 `TelescopeServiceProvider` 服務提供者註冊。取而代之的是，手動在 `App\Providers\AppServiceProvider` 類別的 `register` 方法中註冊 Telescope 的服務提供者。我們將確保目前環境為 `local` 後再註冊這些提供者：

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

最後，您還應該透過將以下內容添加到您的 `composer.json` 檔案中，防止 Telescope 套件被 [自動探索](/docs/{{version}}/packages#package-discovery)：

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

發佈 Telescope 的資源後，其主要設定檔將位於 `config/telescope.php`。此設定檔允許您設定 [watcher 選項](#available-watchers)。每個設定選項都包含其用途的描述，因此請務必詳細探索此檔案。

如果需要，您可以使用 `enabled` 設定選項完全停用 Telescope 的資料收集：

```php
'enabled' => env('TELESCOPE_ENABLED', true),
```

<a name="data-pruning"></a>
### 資料修剪

如果不修剪，`telescope_entries` 資料表會非常快速地累積記錄。為緩解此問題，您應該 [排程](/docs/{{version}}/scheduling) `telescope:prune` Artisan 命令每天執行：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('telescope:prune')->daily();
```

預設情況下，所有超過 24 小時的條目都將被修剪。您可以在呼叫命令時使用 `hours` 選項來決定保留 Telescope 資料的時間長度。例如，以下命令將刪除所有超過 48 小時前建立的記錄：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('telescope:prune --hours=48')->daily();
```

<a name="dashboard-authorization"></a>
### 儀表板授權

Telescope 儀表板可透過 `/telescope` 路由存取。預設情況下，您只能在 `local` 環境中存取此儀表板。在您的 `app/Providers/TelescopeServiceProvider.php` 檔案中，有一個 [授權 Gate](/docs/{{version}}/authorization#gates) 定義。此授權 Gate 控制在 **非本地** 環境中對 Telescope 的存取。您可以根據需要修改此 Gate，以限制對您 Telescope 安裝的存取：

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
> 您應該確保在生產環境中將您的 `APP_ENV` 環境變數更改為 `production`。否則，您的 Telescope 安裝將會公開可用。

<a name="upgrading-telescope"></a>
## 升級 Telescope

當升級到新的 Telescope 主要版本時，仔細查閱 [升級指南](https://github.com/laravel/telescope/blob/master/UPGRADE.md) 非常重要。

此外，當升級到任何新的 Telescope 版本時，您應該重新發佈 Telescope 的資源：

```shell
php artisan telescope:publish
```

為了保持資源最新並避免未來更新中的問題，您可以將 `vendor:publish --tag=laravel-assets` 命令添加到應用程式 `composer.json` 檔案中的 `post-update-cmd` 腳本：

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
## 過濾

<a name="filtering-entries"></a>
### 條目

您可以透過在 `App\Providers\TelescopeServiceProvider` 類別中定義的 `filter` 閉包來過濾 Telescope 記錄的資料。預設情況下，此閉包會記錄 `local` 環境中的所有資料，以及在所有其他環境中的例外、失敗的任務 (failed jobs)、排程任務，以及帶有監控標籤的資料：

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

儘管 `filter` 閉包會過濾單個條目的資料，您可以使用 `filterBatch` 方法註冊一個閉包，該閉包會過濾給定請求或主控台命令的所有資料。如果閉包返回 `true`，則所有條目都將由 Telescope 記錄：

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

Telescope 允許您依據「標籤」來搜尋條目。通常，標籤是 Eloquent Model 類別名稱或經過驗證的使用者 ID，Telescope 會自動將這些資訊新增至條目。有時，您可能希望為條目附加自己的自訂標籤。為此，您可以使用 `Telescope::tag` 方法。`tag` 方法接受一個閉包，該閉包應返回一個標籤陣列。該閉包返回的標籤將與 Telescope 自動附加到條目的任何標籤合併。通常，您應該在 `App\Providers\TelescopeServiceProvider` 類別的 `register` 方法中呼叫 `tag` 方法：

```php
use Laravel\Telescope\EntryType;
use Laravel\Telescope\IncomingEntry;
use Laravel\Telescope\Telescope;

/**
 * Register any application services.
 */
public function register(): void
{
    $this->hideSensitiveRequestDetails();

    Telescope::tag(function (IncomingEntry $entry) {
        return $entry->type === EntryType::REQUEST
            ? ['status:'.$entry->content['response_status']]
            : [];
    });
}
```


<a name="available-watchers"></a>
## 可用 Watchers

Telescope 的「watchers」會在請求或主控台命令執行時收集應用程式資料。您可以在 `config/telescope.php` 設定檔中自訂要啟用的 watchers 列表：

```php
'watchers' => [
    Watchers\CacheWatcher::class => true,
    Watchers\CommandWatcher::class => true,
    // ...
],
```

有些 watchers 也允許您提供額外的自訂選項：

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
### 批次 Watcher

批次 watcher 記錄有關排隊 [批次](/docs/{{version}}/queues#job-batching) 的資訊，包括任務 (job) 和連線資訊。


<a name="cache-watcher"></a>
### 快取 Watcher

快取 watcher 在快取鍵被命中、未命中、更新和遺忘時記錄資料。


<a name="command-watcher"></a>
### 命令 Watcher

命令 watcher 會在執行 Artisan 命令時記錄引數、選項、結束代碼和輸出。如果您希望將某些命令排除在 watcher 的記錄範圍之外，可以在 `config/telescope.php` 檔案的 `ignore` 選項中指定該命令：

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
### Dump Watcher

dump watcher 在 Telescope 中記錄並顯示您的變數 dumps。當使用 Laravel 時，變數可以使用全域 `dump` 函數進行 dump。dump watcher 分頁必須在瀏覽器中打開，才能記錄 dump，否則 watcher 將會忽略這些 dumps。


<a name="event-watcher"></a>
### 事件 Watcher

事件 watcher 記錄應用程式所派遣的任何 [事件](/docs/{{version}}/events) 的 payload、監聽器和廣播資料。Laravel 框架的內部事件會被事件 watcher 忽略。


<a name="exception-watcher"></a>
### 例外 Watcher

例外 watcher 記錄應用程式拋出的任何可報告例外 (reportable exceptions) 的資料和堆疊追蹤。


<a name="gate-watcher"></a>
### Gate Watcher

Gate watcher 記錄應用程式執行的 [Gate 和 Policy](/docs/{{version}}/authorization) 檢查的資料與結果。如果您想將某些能力 (abilities) 排除在 watcher 的記錄範圍之外，可以在 `config/telescope.php` 檔案的 `ignore_abilities` 選項中指定這些能力：

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
### HTTP 客戶端 Watcher

HTTP 客戶端 watcher 記錄應用程式發出的對外 [HTTP 客戶端請求](/docs/{{version}}/http-client)。


<a name="job-watcher"></a>
### 任務 Watcher

任務 watcher 記錄應用程式派遣的任何 [任務 (jobs)](/docs/{{version}}/queues) 的資料和狀態。


<a name="log-watcher"></a>
### 日誌 Watcher

日誌 watcher 記錄應用程式寫入的任何 [日誌資料](/docs/{{version}}/logging)。

依預設，Telescope 只會記錄 `error` 級別及以上的日誌。但是，您可以在應用程式的 `config/telescope.php` 設定檔中修改 `level` 選項來更改此行為：

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
### 郵件 Watcher

郵件 watcher 允許您在瀏覽器中預覽應用程式發送的 [電子郵件](/docs/{{version}}/mail) 及其相關資料。您也可以將電子郵件下載為 `.eml` 檔案。


<a name="model-watcher"></a>
### Model Watcher

Model watcher 會在 Eloquent [Model 事件](/docs/{{version}}/eloquent#events) 派遣時記錄 Model 的變更。您可以透過 watcher 的 `events` 選項指定要記錄哪些 Model 事件：

```php
'watchers' => [
    Watchers\ModelWatcher::class => [
        'enabled' => env('TELESCOPE_MODEL_WATCHER', true),
        'events' => ['eloquent.created*', 'eloquent.updated*'],
    ],
    // ...
],
```

如果您想記錄在給定請求期間水合 (hydrated) 的 Model 數量，請啟用 `hydrations` 選項：

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
### 通知 Watcher

通知 watcher 會記錄應用程式發送的所有 [通知](/docs/{{version}}/notifications)。如果通知觸發了電子郵件且您已啟用郵件 watcher，則該電子郵件也將可在郵件 watcher 畫面中預覽。


<a name="query-watcher"></a>
### 查詢 Watcher

查詢 watcher 記錄應用程式執行所有查詢的原始 SQL、繫結 (bindings) 和執行時間。watcher 還會將任何慢於 100 毫秒的查詢標記為 `slow`。您可以使用 watcher 的 `slow` 選項自訂慢速查詢的閾值：

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
### Redis Watcher

Redis watcher 記錄應用程式執行的所有 [Redis](/docs/{{version}}/redis) 命令。如果您使用 Redis 進行快取，快取命令也將由 Redis watcher 記錄。


<a name="request-watcher"></a>
### 請求 Watcher

請求 watcher 記錄與應用程式處理的任何請求相關的請求、標頭、Session 和回應資料。您可以透過 `size_limit` (以 KB 為單位) 選項限制您記錄的回應資料：

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
### 排程 Watcher

排程 watcher 記錄應用程式執行的任何 [排程任務](/docs/{{version}}/scheduling) 的命令和輸出。


<a name="view-watcher"></a>
### 視圖 Watcher

視圖 watcher 記錄渲染視圖時使用的 [視圖 (view)](/docs/{{version}}/views) 名稱、路徑、資料和「composers」。

<a name="displaying-user-avatars"></a>
## 顯示使用者頭像

Telescope 儀表板會顯示在儲存條目時已驗證的使用者頭像。Telescope 預設會使用 Gravatar 網路服務來擷取頭像。不過，您可以透過在您的 `App\Providers\TelescopeServiceProvider` 類別中註冊一個回呼來客製化頭像 URL。該回呼會接收使用者的 ID 和電子郵件地址，並應回傳使用者的頭像圖片 URL：

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