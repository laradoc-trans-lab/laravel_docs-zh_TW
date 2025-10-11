# Laravel Telescope

- [簡介](#introduction)
- [安裝](#installation)
    - [僅限本地安裝](#local-only-installation)
    - [設定](#configuration)
    - [資料清除](#data-pruning)
    - [儀表板授權](#dashboard-authorization)
- [升級 Telescope](#upgrading-telescope)
- [過濾](#filtering)
    - [條目](#filtering-entries)
    - [批次](#filtering-batches)
- [標記](#tagging)
- [可用的監測器](#available-watchers)
    - [批次監測器](#batch-watcher)
    - [快取監測器](#cache-watcher)
    - [指令監測器](#command-watcher)
    - [傾印監測器](#dump-watcher)
    - [事件監測器](#event-watcher)
    - [例外監測器](#exception-watcher)
    - [Gate 監測器](#gate-watcher)
    - [HTTP 用戶端監測器](#http-client-watcher)
    - [任務監測器](#job-watcher)
    - [日誌監測器](#log-watcher)
    - [郵件監測器](#mail-watcher)
    - [Model 監測器](#model-watcher)
    - [通知監測器](#notification-watcher)
    - [查詢監測器](#query-watcher)
    - [Redis 監測器](#redis-watcher)
    - [請求監測器](#request-watcher)
    - [排程監測器](#schedule-watcher)
    - [視圖監測器](#view-watcher)
- [顯示使用者頭像](#displaying-user-avatars)

<a name="introduction"></a>
## 簡介

[Laravel Telescope](https://github.com/laravel/telescope) 是您本地 Laravel 開發環境的絕佳夥伴。Telescope 提供了對應用程式的請求、例外、日誌條目、資料庫查詢、佇列任務、郵件、通知、快取操作、排程任務、變數傾印等資訊的深入洞察。

<img src="https://laravel.com/img/docs/telescope-example.png">


<a name="installation"></a>
## 安裝

您可以使用 Composer 套件管理器將 Telescope 安裝到您的 Laravel 專案中：

```shell
composer require laravel/telescope
```

安裝 Telescope 後，使用 `telescope:install` Artisan 指令發佈其資產和遷移檔案。安裝 Telescope 後，您還應該執行 `migrate` 指令，以建立儲存 Telescope 資料所需的資料表：

```shell
php artisan telescope:install

php artisan migrate
```

最後，您可以透過 `/telescope` 路由存取 Telescope 儀表板。


<a name="local-only-installation"></a>
### 僅限本地安裝

如果您計畫僅使用 Telescope 來輔助您的本地開發，您可以使用 `--dev` 標誌來安裝 Telescope：

```shell
composer require laravel/telescope --dev

php artisan telescope:install

php artisan migrate
```

執行 `telescope:install` 後，您應該從應用程式的 `bootstrap/providers.php` 設定檔中移除 `TelescopeServiceProvider` 服務提供者註冊。取而代之的是，在您的 `App\Providers\AppServiceProvider` 類別的 `register` 方法中手動註冊 Telescope 的服務提供者。我們將確保在註冊提供者之前，目前的環境是 `local`：

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

最後，您還應該透過將以下內容新增到 `composer.json` 檔案中，來防止 Telescope 套件被 [自動探索](/docs/{{version}}/packages#package-discovery)：

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

發佈 Telescope 的資產後，其主要設定檔將位於 `config/telescope.php`。此設定檔允許您設定您的 [監測器選項](#available-watchers)。每個設定選項都包含其用途的說明，因此請務必仔細探究此檔案。

如果需要，您可以使用 `enabled` 設定選項完全停用 Telescope 的資料收集功能：

    'enabled' => env('TELESCOPE_ENABLED', true),


<a name="data-pruning"></a>
### 資料清除

若不安裝清除功能，`telescope_entries` 資料表會很快累積大量記錄。為了減輕此問題，您應該 [排程](/docs/{{version}}/scheduling) `telescope:prune` Artisan 指令每日執行：

    use Illuminate\Support\Facades\Schedule;

    Schedule::command('telescope:prune')->daily();

預設情況下，所有超過 24 小時的條目都將被清除。您可以在呼叫指令時使用 `hours` 選項來決定保留 Telescope 資料的時間長度。例如，以下指令將刪除所有超過 48 小時前建立的記錄：

    use Illuminate\Support\Facades\Schedule;

    Schedule::command('telescope:prune --hours=48')->daily();


<a name="dashboard-authorization"></a>
### 儀表板授權

Telescope 儀表板可透過 `/telescope` 路由存取。預設情況下，您只能在 `local` 環境中存取此儀表板。在您的 `app/Providers/TelescopeServiceProvider.php` 檔案中，有一個 [授權 Gate](/docs/{{version}}/authorization#gates) 定義。此授權 Gate 控制在 **非本地** 環境中對 Telescope 的存取。您可以根據需要修改此 Gate，以限制對您的 Telescope 安裝的存取：

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

> [!WARNING]  
> 您應該確保在您的正式環境中將 `APP_ENV` 環境變數更改為 `production`。否則，您的 Telescope 安裝將會公開可用。


<a name="upgrading-telescope"></a>
## 升級 Telescope

當升級到 Telescope 的新主要版本時，請務必仔細審閱 [升級指南](https://github.com/laravel/telescope/blob/master/UPGRADE.md)。

此外，當升級到任何新的 Telescope 版本時，您應該重新發佈 Telescope 的資產：

```shell
php artisan telescope:publish
```

為了保持資產最新並避免未來更新中的問題，您可以將 `vendor:publish --tag=laravel-assets` 指令新增到應用程式 `composer.json` 檔案中的 `post-update-cmd` 腳本中：

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

您可以透過定義在 `App\Providers\TelescopeServiceProvider` 類別中的 `filter` 閉包來過濾 Telescope 記錄的資料。預設情況下，此閉包會記錄 `local` 環境中的所有資料，以及在所有其他環境中的例外、失敗的任務、排程任務和帶有受監測標籤的資料：

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

雖然 `filter` 閉包會過濾個別條目的資料，但您可以使用 `filterBatch` 方法來註冊一個閉包，該閉包會過濾特定請求或主控台指令的所有資料。如果閉包回傳 `true`，所有條目都將被 Telescope 記錄：

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
## 標記

Telescope 允許您依據「標記」搜尋條目。通常，標記是 Eloquent Model 類別名稱或已驗證的使用者 ID，Telescope 會自動將其新增至條目中。有時，您可能希望為條目附加自己的自訂標記。為此，您可以使用 `Telescope::tag` 方法。`tag` 方法接受一個閉包，該閉包應回傳一個標記陣列。閉包回傳的標記將與 Telescope 自動附加到條目的任何標記合併。通常，您應該在 `App\Providers\TelescopeServiceProvider` 類別的 `register` 方法中呼叫 `tag` 方法：

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

當請求或終端機指令執行時，Telescope 的「監測器」會收集應用程式資料。您可以在 `config/telescope.php` 設定檔中自訂要啟用的監測器列表：

    'watchers' => [
        Watchers\CacheWatcher::class => true,
        Watchers\CommandWatcher::class => true,
        ...
    ],

有些監測器也允許您提供額外的自訂選項：

    'watchers' => [
        Watchers\QueryWatcher::class => [
            'enabled' => env('TELESCOPE_QUERY_WATCHER', true),
            'slow' => 100,
        ],
        ...
    ],


<a name="batch-watcher"></a>
### 批次監測器

批次監測器會記錄排入佇列的 [批次](/docs/{{version}}/queues#job-batching) 相關資訊，包括任務和連線資訊。


<a name="cache-watcher"></a>
### 快取監測器

快取監測器會在快取鍵被命中、未命中、更新和遺忘時記錄資料。


<a name="command-watcher"></a>
### 指令監測器

指令監測器會記錄 Artisan 指令執行時的引數、選項、結束代碼和輸出。如果您想將某些指令從監測器記錄中排除，可以在 `config/telescope.php` 檔案的 `ignore` 選項中指定該指令：

    'watchers' => [
        Watchers\CommandWatcher::class => [
            'enabled' => env('TELESCOPE_COMMAND_WATCHER', true),
            'ignore' => ['key:generate'],
        ],
        ...
    ],


<a name="dump-watcher"></a>
### 傾印監測器

傾印監測器會記錄並在 Telescope 中顯示您的變數傾印。使用 Laravel 時，可以使用全域 `dump` 函式傾印變數。傾印監測器的分頁必須在瀏覽器中開啟才能記錄傾印，否則監測器將會忽略這些傾印。


<a name="event-watcher"></a>
### 事件監測器

事件監測器會記錄您的應用程式分派的任何 [事件](/docs/{{version}}/events) 的負載、監聽器和廣播資料。Laravel 框架的內部事件將被事件監測器忽略。


<a name="exception-watcher"></a>
### 例外監測器

例外監測器會記錄您的應用程式拋出的任何可報告的例外資料和堆疊追蹤。


<a name="gate-watcher"></a>
### Gate 監測器

Gate 監測器會記錄您的應用程式 [Gate 和 Policy](/docs/{{version}}/authorization) 檢查的資料和結果。如果您想將某些權限從監測器記錄中排除，可以在 `config/telescope.php` 檔案的 `ignore_abilities` 選項中指定這些權限：

    'watchers' => [
        Watchers\GateWatcher::class => [
            'enabled' => env('TELESCOPE_GATE_WATCHER', true),
            'ignore_abilities' => ['viewNova'],
        ],
        ...
    ],


<a name="http-client-watcher"></a>
### HTTP 用戶端監測器

HTTP 用戶端監測器會記錄您的應用程式發出的傳出 [HTTP 用戶端請求](/docs/{{version}}/http-client)。


<a name="job-watcher"></a>
### 任務監測器

任務監測器會記錄您的應用程式分派的任何 [任務](/docs/{{version}}/queues) 的資料和狀態。


<a name="log-watcher"></a>
### 日誌監測器

日誌監測器會記錄您的應用程式寫入的任何 [日誌資料](/docs/{{version}}/logging)。

預設情況下，Telescope 只會記錄 `error` 級別及以上的日誌。但是，您可以修改應用程式 `config/telescope.php` 設定檔中的 `level` 選項來更改此行為：

    'watchers' => [
        Watchers\LogWatcher::class => [
            'enabled' => env('TELESCOPE_LOG_WATCHER', true),
            'level' => 'debug',
        ],

        // ...
    ],


<a name="mail-watcher"></a>
### 郵件監測器

郵件監測器允許您在瀏覽器中預覽您的應用程式傳送的 [電子郵件](/docs/{{version}}/mail) 及其相關資料。您也可以將電子郵件下載為 `.eml` 檔案。


<a name="model-watcher"></a>
### Model 監測器

Model 監測器會在 Eloquent [Model 事件](/docs/{{version}}/eloquent#events) 分派時記錄 Model 變更。您可以透過監測器的 `events` 選項指定應記錄哪些 Model 事件：

    'watchers' => [
        Watchers\ModelWatcher::class => [
            'enabled' => env('TELESCOPE_MODEL_WATCHER', true),
            'events' => ['eloquent.created*', 'eloquent.updated*'],
        ],
        ...
    ],

如果您想記錄在給定請求期間水合化的 Model 數量，請啟用 `hydrations` 選項：

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

通知監測器會記錄您的應用程式傳送的所有 [通知](/docs/{{version}}/notifications)。如果通知觸發電子郵件並且您已啟用郵件監測器，則該電子郵件也將可在郵件監測器畫面中預覽。


<a name="query-watcher"></a>
### 查詢監測器

查詢監測器會記錄您的應用程式執行的所有查詢的原始 SQL、繫結和執行時間。監測器還會將任何慢於 100 毫秒的查詢標記為 `slow`。您可以使用監測器的 `slow` 選項自訂慢速查詢的閾值：

    'watchers' => [
        Watchers\QueryWatcher::class => [
            'enabled' => env('TELESCOPE_QUERY_WATCHER', true),
            'slow' => 50,
        ],
        ...
    ],


<a name="redis-watcher"></a>
### Redis 監測器

Redis 監測器會記錄您的應用程式執行的所有 [Redis](/docs/{{version}}/redis) 指令。如果您使用 Redis 進行快取，快取指令也將由 Redis 監測器記錄。


<a name="request-watcher"></a>
### 請求監測器

請求監測器會記錄與應用程式處理的任何請求相關的請求、標頭、Session 和回應資料。您可以使用 `size_limit` (單位為 KB) 選項來限制記錄的回應資料：

    'watchers' => [
        Watchers\RequestWatcher::class => [
            'enabled' => env('TELESCOPE_REQUEST_WATCHER', true),
            'size_limit' => env('TELESCOPE_RESPONSE_SIZE_LIMIT', 64),
        ],
        ...
    ],


<a name="schedule-watcher"></a>
### 排程監測器

排程監測器會記錄您的應用程式執行的任何 [排程任務](/docs/{{version}}/scheduling) 的指令和輸出。


<a name="view-watcher"></a>
### 視圖監測器

視圖監測器會記錄渲染視圖時使用的 [視圖](/docs/{{version}}/views) 名稱、路徑、資料和「組合器」。

<a name="displaying-user-avatars"></a>
## 顯示使用者頭像

Telescope 儀表板會顯示當條目被儲存時，已驗證使用者對應的使用者頭像。預設情況下，Telescope 會使用 Gravatar 網路服務來取得頭像。然而，您可以透過在 `App\Providers\TelescopeServiceProvider` 類別中註冊回呼來自訂頭像 URL。這個回呼會接收使用者的 ID 和電子郵件地址，並應回傳使用者頭像的圖片 URL：

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