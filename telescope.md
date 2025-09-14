# Laravel Telescope

- [簡介](#introduction)
- [安裝](#installation)
    - [僅限本地安裝](#local-only-installation)
    - [設定](#configuration)
    - [資料清理](#data-pruning)
    - [儀表板授權](#dashboard-authorization)
- [升級 Telescope](#upgrading-telescope)
- [過濾](#filtering)
    - [項目](#filtering-entries)
    - [批次](#filtering-batches)
- [標籤](#tagging)
- [可用的 Watcher](#available-watchers)
    - [Batch Watcher](#batch-watcher)
    - [Cache Watcher](#cache-watcher)
    - [Command Watcher](#command-watcher)
    - [Dump Watcher](#dump-watcher)
    - [Event Watcher](#event-watcher)
    - [Exception Watcher](#exception-watcher)
    - [Gate Watcher](#gate-watcher)
    - [HTTP Client Watcher](#http-client-watcher)
    - [Job Watcher](#job-watcher)
    - [Log Watcher](#log-watcher)
    - [Mail Watcher](#mail-watcher)
    - [Model Watcher](#model-watcher)
    - [Notification Watcher](#notification-watcher)
    - [Query Watcher](#query-watcher)
    - [Redis Watcher](#redis-watcher)
    - [Request Watcher](#request-watcher)
    - [Schedule Watcher](#schedule-watcher)
    - [View Watcher](#view-watcher)
- [顯示使用者頭像](#displaying-user-avatars)

<a name="introduction"></a>
## 簡介

[Laravel Telescope](https://github.com/laravel/telescope) 是您本地 Laravel 開發環境的絕佳夥伴。Telescope 提供了對應用程式接收到的請求、例外、日誌項目、資料庫查詢、佇列任務、郵件、通知、快取操作、排程任務、變數傾印等更深入的洞察。

<img src="https://laravel.com/img/docs/telescope-example.png">

<a name="installation"></a>
## 安裝

您可以使用 Composer 套件管理器將 Telescope 安裝到您的 Laravel 專案中：

```shell
composer require laravel/telescope
```

安裝 Telescope 後，使用 `telescope:install` Artisan 命令發布其資源與遷移檔。安裝 Telescope 後，您還應該執行 `migrate` 命令，以建立儲存 Telescope 資料所需的資料表：

```shell
php artisan telescope:install

php artisan migrate
```

最後，您可以透過 `/telescope` 路由存取 Telescope 儀表板。

<a name="local-only-installation"></a>
### 僅限本地安裝

如果您計畫僅使用 Telescope 協助您的本地開發，您可以使用 `--dev` 旗標安裝 Telescope：

```shell
composer require laravel/telescope --dev

php artisan telescope:install

php artisan migrate
```

執行 `telescope:install` 後，您應該從應用程式的 `bootstrap/providers.php` 設定檔中移除 `TelescopeServiceProvider` 服務提供者註冊。相反地，您應該在 `App\Providers\AppServiceProvider` 類別的 `register` 方法中手動註冊 Telescope 的服務提供者。我們將確保在註冊提供者之前，目前的環境是 `local`：

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

最後，您還應該透過在 `composer.json` 檔案中新增以下內容，防止 Telescope 套件被 [自動探索](/docs/{{version}}/packages#package-discovery)：

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

發布 Telescope 的資源後，其主要設定檔將位於 `config/telescope.php`。此設定檔允許您設定您的 [watcher 選項](#available-watchers)。每個設定選項都包含其用途的描述，因此請務必仔細探索此檔案。

如果需要，您可以透過 `enabled` 設定選項完全停用 Telescope 的資料收集：

```php
'enabled' => env('TELESCOPE_ENABLED', true),
```

<a name="data-pruning"></a>
### 資料清理

如果沒有清理，`telescope_entries` 資料表會非常快速地累積記錄。為了緩解這個問題，您應該[排程](/docs/{{version}}/scheduling) `telescope:prune` Artisan 命令每日執行：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('telescope:prune')->daily();
```

預設情況下，所有超過 24 小時的項目都將被清理。您可以在呼叫命令時使用 `hours` 選項來決定保留 Telescope 資料的時間長度。例如，以下命令將刪除所有 48 小時前建立的記錄：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('telescope:prune --hours=48')->daily();
```

<a name="dashboard-authorization"></a>
### 儀表板授權

Telescope 儀表板可以透過 `/telescope` 路由存取。預設情況下，您只能在 `local` 環境中存取此儀表板。在您的 `app/Providers/TelescopeServiceProvider.php` 檔案中，有一個 [授權 Gate](/docs/{{version}}/authorization#gates) 定義。此授權 Gate 控制在**非本地**環境中對 Telescope 的存取。您可以根據需要修改此 Gate，以限制對您的 Telescope 安裝的存取：

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
> 您應該確保在生產環境中將 `APP_ENV` 環境變數更改為 `production`。否則，您的 Telescope 安裝將會公開可用。

<a name="upgrading-telescope"></a>
## 升級 Telescope

升級到新的 Telescope 主要版本時，務必仔細查閱 [升級指南](https://github.com/laravel/telescope/blob/master/UPGRADE.md)。

此外，升級到任何新的 Telescope 版本時，您應該重新發布 Telescope 的資源：

```shell
php artisan telescope:publish
```

為了保持資源最新並避免未來更新中的問題，您可以將 `vendor:publish --tag=laravel-assets` 命令新增到應用程式 `composer.json` 檔案中的 `post-update-cmd` 腳本：

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
### 項目

您可以透過 `App\Providers\TelescopeServiceProvider` 類別中定義的 `filter` 閉包來過濾 Telescope 記錄的資料。預設情況下，此閉包在 `local` 環境中記錄所有資料，並在所有其他環境中記錄例外、失敗的任務、排程任務以及帶有受監控標籤的資料：

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

雖然 `filter` 閉包過濾單個項目的資料，但您可以使用 `filterBatch` 方法註冊一個閉包，該閉包過濾給定請求或控制台命令的所有資料。如果閉包返回 `true`，則所有項目都將由 Telescope 記錄：

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
## 標籤

Telescope 允許您透過「標籤」搜尋項目。通常，標籤是 Eloquent 模型類別名稱或已驗證的使用者 ID，Telescope 會自動將其新增到項目中。偶爾，您可能希望將自己的自訂標籤附加到項目中。為此，您可以使用 `Telescope::tag` 方法。`tag` 方法接受一個閉包，該閉包應返回一個標籤陣列。閉包返回的標籤將與 Telescope 自動附加到項目的任何標籤合併。通常，您應該在 `App\Providers\TelescopeServiceProvider` 類別的 `register` 方法中呼叫 `tag` 方法：

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
## 可用的 Watcher

Telescope 「watcher」在請求或控制台命令執行時收集應用程式資料。您可以在 `config/telescope.php` 設定檔中自訂要啟用的 watcher 列表：

```php
'watchers' => [
    Watchers\CacheWatcher::class => true,
    Watchers\CommandWatcher::class => true,
    // ...
],
```

一些 watcher 也允許您提供額外的自訂選項：

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
### Batch Watcher

Batch watcher 記錄有關佇列 [批次](/docs/{{version}}/queues#job-batching) 的資訊，包括任務和連線資訊。

<a name="cache-watcher"></a>
### Cache Watcher

Cache watcher 在快取鍵被命中、未命中、更新和遺忘時記錄資料。

<a name="command-watcher"></a>
### Command Watcher

Command watcher 在 Artisan 命令執行時記錄參數、選項、結束代碼和輸出。如果您想從 watcher 中排除某些命令，您可以在 `config/telescope.php` 檔案中的 `ignore` 選項中指定該命令：

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

Dump watcher 在 Telescope 中記錄並顯示您的變數傾印。使用 Laravel 時，可以使用全域 `dump` 函數傾印變數。Dump watcher 標籤必須在瀏覽器中打開才能記錄傾印，否則，傾印將被 watcher 忽略。

<a name="event-watcher"></a>
### Event Watcher

Event watcher 記錄應用程式分派的任何 [事件](/docs/{{version}}/events) 的負載、監聽器和廣播資料。Laravel 框架的內部事件會被 Event watcher 忽略。

<a name="exception-watcher"></a>
### Exception Watcher

Exception watcher 記錄應用程式拋出的任何可報告例外（reportable exceptions）的資料和堆疊追蹤。

<a name="gate-watcher"></a>
### Gate Watcher

Gate watcher 記錄應用程式的 [Gate 和 Policy](/docs/{{version}}/authorization) 檢查的資料和結果。如果您想從 watcher 中排除某些能力（abilities），您可以在 `config/telescope.php` 檔案中的 `ignore_abilities` 選項中指定這些能力：

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
### HTTP Client Watcher

HTTP Client watcher 記錄應用程式發出的對外 [HTTP Client 請求](/docs/{{version}}/http-client)。

<a name="job-watcher"></a>
### Job Watcher

Job watcher 記錄應用程式分派的任何 [任務](/docs/{{version}}/queues) 的資料和狀態。

<a name="log-watcher"></a>
### Log Watcher

Log watcher 記錄應用程式寫入的任何 [日誌資料](/docs/{{version}}/logging)。

預設情況下，Telescope 只會記錄 `error` 等級及以上的日誌。但是，您可以在應用程式的 `config/telescope.php` 設定檔中修改 `level` 選項來更改此行為：

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
### Mail Watcher

Mail watcher 允許您在瀏覽器中預覽應用程式發送的 [電子郵件](/docs/{{version}}/mail) 及其相關資料。您也可以將電子郵件下載為 `.eml` 檔案。

<a name="model-watcher"></a>
### Model Watcher

Model watcher 在 Eloquent [模型事件](/docs/{{version}}/eloquent#events) 分派時記錄模型變更。您可以透過 watcher 的 `events` 選項指定要記錄的模型事件：

```php
'watchers' => [
    Watchers\ModelWatcher::class => [
        'enabled' => env('TELESCOPE_MODEL_WATCHER', true),
        'events' => ['eloquent.created*', 'eloquent.updated*'],
    ],
    // ...
],
```

如果您想記錄在給定請求期間水合（hydrated）的模型數量，請啟用 `hydrations` 選項：

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
### Notification Watcher

Notification watcher 記錄應用程式發送的所有 [通知](/docs/{{version}}/notifications)。如果通知觸發電子郵件且您已啟用 Mail watcher，則該電子郵件也將在 Mail watcher 畫面中預覽。

<a name="query-watcher"></a>
### Query Watcher

Query watcher 記錄應用程式執行的所有查詢的原始 SQL、綁定和執行時間。Watcher 還將所有執行時間超過 100 毫秒的查詢標記為 `slow`。您可以使用 watcher 的 `slow` 選項自訂慢查詢閾值：

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
### Request Watcher

Request watcher 記錄與應用程式處理的任何請求相關的請求、標頭、Session 和回應資料。您可以透過 `size_limit`（以 KB 為單位）選項限制記錄的回應資料：

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
### Schedule Watcher

Schedule watcher 記錄應用程式執行的任何 [排程任務](/docs/{{version}}/scheduling) 的命令和輸出。

<a name="view-watcher"></a>
### View Watcher

View watcher 記錄渲染視圖時使用的 [視圖](/docs/{{version}}/views) 名稱、路徑、資料和「composers」。

<a name="displaying-user-avatars"></a>
## 顯示使用者頭像

Telescope 儀表板顯示在儲存給定項目時已驗證的使用者的頭像。預設情況下，Telescope 將使用 Gravatar 網路服務檢索頭像。但是，您可以透過在 `App\Providers\TelescopeServiceProvider` 類別中註冊回呼來自訂頭像 URL。回呼將接收使用者的 ID 和電子郵件地址，並應返回使用者的頭像圖片 URL：

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
