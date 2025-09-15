# Laravel Telescope

- [簡介](#introduction)
- [安裝](#installation)
    - [僅限本地安裝](#local-only-installation)
    - [設定](#configuration)
    - [資料清理](#data-pruning)
    - [儀表板授權](#dashboard-authorization)
- [升級 Telescope](#upgrading-telescope)
- [篩選](#filtering)
    - [項目](#filtering-entries)
    - [批次](#filtering-batches)
- [標籤](#tagging)
- [可用的監測器](#available-watchers)
    - [批次監測器](#batch-watcher)
    - [快取監測器](#cache-watcher)
    - [命令監測器](#command-watcher)
    - [傾印監測器](#dump-watcher)
    - [事件監測器](#event-watcher)
    - [例外監測器](#exception-watcher)
    - [Gate 監測器](#gate-watcher)
    - [HTTP Client 監測器](#http-client-watcher)
    - [任務監測器](#job-watcher)
    - [日誌監測器](#log-watcher)
    - [郵件監測器](#mail-watcher)
    - [模型監測器](#model-watcher)
    - [通知監測器](#notification-watcher)
    - [查詢監測器](#query-watcher)
    - [Redis 監測器](#redis-watcher)
    - [請求監測器](#request-watcher)
    - [排程監測器](#schedule-watcher)
    - [視圖監測器](#view-watcher)
- [顯示使用者頭像](#displaying-user-avatars)

<a name="introduction"></a>
## 簡介

[Laravel Telescope](https://github.com/laravel/telescope) 是您本地 Laravel 開發環境的絕佳夥伴。Telescope 提供了對應用程式接收到的請求、例外、日誌項目、資料庫查詢、佇列任務、郵件、通知、快取操作、排程任務、變數傾印等資訊的深入洞察。

<img src="https://laravel.com/img/docs/telescope-example.png">

<a name="installation"></a>
## 安裝

您可以使用 Composer 套件管理器將 Telescope 安裝到您的 Laravel 專案中：

```shell
composer require laravel/telescope
```

安裝 Telescope 後，請使用 `telescope:install` Artisan 命令發佈其資源與遷移檔。安裝 Telescope 後，您還應該執行 `migrate` 命令，以建立儲存 Telescope 資料所需的資料表：

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

執行 `telescope:install` 後，您應該從應用程式的 `bootstrap/providers.php` 設定檔中移除 `TelescopeServiceProvider` 服務提供者註冊。相反地，請在 `App\Providers\AppServiceProvider` 類別的 `register` 方法中手動註冊 Telescope 的服務提供者。我們將確保在註冊提供者之前，目前的環境是 `local`：

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

最後，您還應該透過在 `composer.json` 檔案中新增以下內容，防止 Telescope 套件被[自動探索](/docs/{{version}}/packages#package-discovery)：

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

發佈 Telescope 的資源後，其主要設定檔將位於 `config/telescope.php`。此設定檔允許您設定[監測器選項](#available-watchers)。每個設定選項都包含其用途的描述，因此請務必仔細探索此檔案。

如果需要，您可以使用 `enabled` 設定選項完全停用 Telescope 的資料收集：

    'enabled' => env('TELESCOPE_ENABLED', true),

<a name="data-pruning"></a>
### 資料清理

如果沒有清理，`telescope_entries` 資料表會非常快速地累積記錄。為了緩解這個問題，您應該[排程](/docs/{{version}}/scheduling) `telescope:prune` Artisan 命令每天執行：

    use Illuminate\Support\Facades\Schedule;

    Schedule::command('telescope:prune')->daily();

預設情況下，所有超過 24 小時的項目都將被清理。您可以在呼叫命令時使用 `hours` 選項來決定保留 Telescope 資料的時間長度。例如，以下命令將刪除所有在 48 小時前建立的記錄：

    use Illuminate\Support\Facades\Schedule;

    Schedule::command('telescope:prune --hours=48')->daily();

<a name="dashboard-authorization"></a>
### 儀表板授權

Telescope 儀表板可以透過 `/telescope` 路由存取。預設情況下，您只能在 `local` 環境中存取此儀表板。在您的 `app/Providers/TelescopeServiceProvider.php` 檔案中，有一個[授權 Gate](/docs/{{version}}/authorization#gates) 定義。此授權 Gate 控制在**非本地**環境中對 Telescope 的存取。您可以根據需要修改此 Gate，以限制對您的 Telescope 安裝的存取：

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
                'taylor @laravel.com',
            ]);
        });
    }

> [!WARNING]  
> 您應該確保在您的正式環境中將 `APP_ENV` 環境變數更改為 `production`。否則，您的 Telescope 安裝將會公開可用。

<a name="upgrading-telescope"></a>
## 升級 Telescope

升級到 Telescope 的新主要版本時，務必仔細查閱[升級指南](https://github.com/laravel/telescope/blob/master/UPGRADE.md)。

此外，升級到任何新的 Telescope 版本時，您應該重新發佈 Telescope 的資源：

```shell
php artisan telescope:publish
```

為了保持資源最新並避免未來更新中的問題，您可以將 `vendor:publish --tag=laravel-assets` 命令新增到應用程式 `composer.json` 檔案中的 `post-update-cmd` 腳本：

```json
{
    "scripts": {
        "post-update-cmd": [
            " @php artisan vendor:publish --tag=laravel-assets --ansi --force"
        ]
    }
}
```

<a name="filtering"></a>
## 篩選

<a name="filtering-entries"></a>
### 項目

您可以透過 `App\Providers\TelescopeServiceProvider` 類別中定義的 `filter` 閉包來篩選 Telescope 記錄的資料。預設情況下，此閉包在 `local` 環境中記錄所有資料，並在所有其他環境中記錄例外、失敗的任務、排程任務以及帶有受監控標籤的資料：

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

<a name="filtering-batches"></a>
### 批次

雖然 `filter` 閉包篩選單個項目的資料，但您可以使用 `filterBatch` 方法註冊一個閉包，該閉包篩選給定請求或主控台命令的所有資料。如果閉包返回 `true`，則所有項目都將由 Telescope 記錄：

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

<a name="tagging"></a>
## 標籤

Telescope 允許您透過「標籤」搜尋項目。通常，標籤是 Eloquent 模型類別名稱或已驗證的使用者 ID，Telescope 會自動將其新增到項目中。有時，您可能希望將自己的自訂標籤附加到項目中。為此，您可以使用 `Telescope::tag` 方法。`tag` 方法接受一個閉包，該閉包應返回一個標籤陣列。閉包返回的標籤將與 Telescope 自動附加到項目的任何標籤合併。通常，您應該在 `App\Providers\TelescopeServiceProvider` 類別的 `register` 方法中呼叫 `tag` 方法：

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

<a name="available-watchers"></a>
## 可用的監測器

Telescope「監測器」在執行請求或主控台命令時收集應用程式資料。您可以在 `config/telescope.php` 設定檔中自訂要啟用的監測器列表：

    'watchers' => [
        Watchers\CacheWatcher::class => true,
        Watchers\CommandWatcher::class => true,
        ...
    ],

一些監測器還允許您提供額外的自訂選項：

    'watchers' => [
        Watchers\QueryWatcher::class => [
            'enabled' => env('TELESCOPE_QUERY_WATCHER', true),
            'slow' => 100,
        ],
        ...
    ],

<a name="batch-watcher"></a>
### 批次監測器

批次監測器記錄有關佇列[批次](/docs/{{version}}/queues#job-batching)的資訊，包括任務和連線資訊。

<a name="cache-watcher"></a>
### 快取監測器

快取監測器在快取鍵被命中、未命中、更新和遺忘時記錄資料。

<a name="command-watcher"></a>
### 命令監測器

命令監測器在執行 Artisan 命令時記錄參數、選項、結束代碼和輸出。如果您想從監測器中排除某些命令，您可以在 `config/telescope.php` 檔案中的 `ignore` 選項中指定該命令：

    'watchers' => [
        Watchers\CommandWatcher::class => [
            'enabled' => env('TELESCOPE_COMMAND_WATCHER', true),
            'ignore' => ['key:generate'],
        ],
        ...
    ],

<a name="dump-watcher"></a>
### 傾印監測器

傾印監測器記錄並在 Telescope 中顯示您的變數傾印。使用 Laravel 時，可以使用全域 `dump` 函數傾印變數。傾印監測器分頁必須在瀏覽器中開啟才能記錄傾印，否則傾印將被監測器忽略。

<a name="event-watcher"></a>
### 事件監測器

事件監測器記錄應用程式分派的任何[事件](/docs/{{version}}/events)的負載、監聽器和廣播資料。Laravel 框架的內部事件會被事件監測器忽略。

<a name="exception-watcher"></a>
### 例外監測器

例外監測器記錄應用程式拋出的任何可報告例外的資料和堆疊追蹤。

<a name="gate-watcher"></a>
### Gate 監測器

Gate 監測器記錄應用程式[Gate 和 Policy](/docs/{{version}}/authorization) 檢查的資料和結果。如果您想從監測器中排除某些能力，您可以在 `config/telescope.php` 檔案中的 `ignore_abilities` 選項中指定這些能力：

    'watchers' => [
        Watchers\GateWatcher::class => [
            'enabled' => env('TELESCOPE_GATE_WATCHER', true),
            'ignore_abilities' => ['viewNova'],
        ],
        ...
    ],

<a name="http-client-watcher"></a>
### HTTP Client 監測器

HTTP Client 監測器記錄應用程式發出的[HTTP Client 請求](/docs/{{version}}/http-client)。

<a name="job-watcher"></a>
### 任務監測器

任務監測器記錄應用程式分派的任何[任務](/docs/{{version}}/queues)的資料和狀態。

<a name="log-watcher"></a>
### 日誌監測器

日誌監測器記錄應用程式寫入的任何[日誌資料](/docs/{{version}}/logging)。

預設情況下，Telescope 只會記錄 `error` 等級及以上的日誌。但是，您可以修改應用程式 `config/telescope.php` 設定檔中的 `level` 選項來修改此行為：

    'watchers' => [
        Watchers\LogWatcher::class => [
            'enabled' => env('TELESCOPE_LOG_WATCHER', true),
            'level' => 'debug',
        ],

        // ...
    ],

<a name="mail-watcher"></a>
### 郵件監測器

郵件監測器允許您在瀏覽器中預覽應用程式發送的[電子郵件](/docs/{{version}}/mail)及其相關資料。您也可以將電子郵件下載為 `.eml` 檔案。

<a name="model-watcher"></a>
### 模型監測器

模型監測器在分派 Eloquent [模型事件](/docs/{{version}}/eloquent#events)時記錄模型變更。您可以透過監測器的 `events` 選項指定應記錄哪些模型事件：

    'watchers' => [
        Watchers\ModelWatcher::class => [
            'enabled' => env('TELESCOPE_MODEL_WATCHER', true),
            'events' => ['eloquent.created*', 'eloquent.updated*'],
        ],
        ...
    ],

如果您想記錄在給定請求期間水合的模型數量，請啟用 `hydrations` 選項：

    'watchers' => [
        Watchers\ModelWatcher::class => [
            'enabled' => env('TELESCOPE_MODEL_WATCHER', true),
            'events' => ['eloquent.created*', 'eloquent.updated*'],
            'hydrations' => true,
        ],
        ...
    ],

<a name="notification-watcher"></a>
### 通知監測器

通知監測器記錄應用程式發送的所有[通知](/docs/{{version}}/notifications)。如果通知觸發電子郵件並且您已啟用郵件監測器，則該電子郵件也將在郵件監測器螢幕上預覽。

<a name="query-watcher"></a>
### 查詢監測器

查詢監測器記錄應用程式執行的所有查詢的原始 SQL、綁定和執行時間。監測器還將任何慢於 100 毫秒的查詢標記為 `slow`。您可以使用監測器的 `slow` 選項自訂慢查詢閾值：

    'watchers' => [
        Watchers\QueryWatcher::class => [
            'enabled' => env('TELESCOPE_QUERY_WATCHER', true),
            'slow' => 50,
        ],
        ...
    ],

<a name="redis-watcher"></a>
### Redis 監測器

Redis 監測器記錄應用程式執行的所有 [Redis](/docs/{{version}}/redis) 命令。如果您使用 Redis 進行快取，快取命令也將由 Redis 監測器記錄。

<a name="request-watcher"></a>
### 請求監測器

請求監測器記錄與應用程式處理的任何請求相關的請求、標頭、Session 和回應資料。您可以透過 `size_limit` (以 KB 為單位) 選項限制記錄的回應資料：

    'watchers' => [
        Watchers\RequestWatcher::class => [
            'enabled' => env('TELESCOPE_REQUEST_WATCHER', true),
            'size_limit' => env('TELESCOPE_RESPONSE_SIZE_LIMIT', 64),
        ],
        ...
    ],

<a name="schedule-watcher"></a>
### 排程監測器

排程監測器記錄應用程式執行的任何[排程任務](/docs/{{version}}/scheduling)的命令和輸出。

<a name="view-watcher"></a>
### 視圖監測器

視圖監測器記錄渲染視圖時使用的[視圖](/docs/{{version}}/views)名稱、路徑、資料和「composers」。

<a name="displaying-user-avatars"></a>
## 顯示使用者頭像

Telescope 儀表板顯示在儲存給定項目時已驗證的使用者頭像。預設情況下，Telescope 將使用 Gravatar 網路服務檢索頭像。但是，您可以透過在 `App\Providers\TelescopeServiceProvider` 類別中註冊回呼來自訂頭像 URL。回呼將接收使用者的 ID 和電子郵件地址，並應返回使用者頭像圖片 URL：

    use App\Models\User;
    use Laravel\Telescope\Telescope;

    /**
     * Register any application services.
     */
    public function register(): void
    {
        // ...

        Telescope::avatar(function (string $id, string $email) {
            return '/avatars/'.User::find($id)->avatar_path;
        });
    }
