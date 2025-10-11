# Laravel Pulse

- [簡介](#introduction)
- [安裝](#installation)
    - [設定](#configuration)
- [儀表板](#dashboard)
    - [授權](#dashboard-authorization)
    - [自訂](#dashboard-customization)
    - [解析使用者](#dashboard-resolving-users)
    - [卡片](#dashboard-cards)
- [擷取資料](#capturing-entries)
    - [紀錄器](#recorders)
    - [篩選](#filtering)
- [效能](#performance)
    - [使用不同的資料庫](#using-a-different-database)
    - [Redis 注入](#ingest)
    - [取樣](#sampling)
    - [修剪](#trimming)
    - [處理 Pulse 異常](#pulse-exceptions)
- [自訂卡片](#custom-cards)
    - [卡片元件](#custom-card-components)
    - [樣式](#custom-card-styling)
    - [資料擷取與聚合](#custom-card-data)

<a name="introduction"></a>
## 簡介

[Laravel Pulse](https://github.com/laravel/pulse) 提供一目了然的洞察，協助您了解應用程式的效能與使用情況。透過 Pulse，您可以找出緩慢的 Job 和端點等瓶頸，找到最活躍的使用者等等。

如需深入偵錯個別事件，請參閱 [Laravel Telescope](/docs/{{version}}/telescope)。


<a name="installation"></a>
## 安裝

> [!WARNING]  
> Pulse 的第一方儲存實作目前需要 MySQL、MariaDB 或 PostgreSQL 資料庫。如果您使用的是不同的資料庫引擎，您將需要一個獨立的 MySQL、MariaDB 或 PostgreSQL 資料庫來存放您的 Pulse 資料。

您可以使用 Composer 套件管理工具來安裝 Pulse：

```sh
composer require laravel/pulse
```

接著，您應該使用 `vendor:publish` Artisan 指令發佈 Pulse 的設定與遷移檔案：

```shell
php artisan vendor:publish --provider="Laravel\Pulse\PulseServiceProvider"
```

最後，您應該執行 `migrate` 指令，以建立儲存 Pulse 資料所需的資料表：

```shell
php artisan migrate
```

一旦 Pulse 的資料庫遷移已執行完畢，您就可以透過 `/pulse` 路由存取 Pulse 儀表板。

> [!NOTE]  
> 如果您不想將 Pulse 資料儲存在應用程式的主要資料庫中，您可以[指定一個專用的資料庫連線](#using-a-different-database)。


<a name="configuration"></a>
### 設定

Pulse 的許多設定選項都可以透過環境變數來控制。若要查看可用選項、註冊新的紀錄器或設定進階選項，您可以發佈 `config/pulse.php` 設定檔：

```sh
php artisan vendor:publish --tag=pulse-config
```

<a name="dashboard"></a>
## 儀表板


<a name="dashboard-authorization"></a>
### 授權

Pulse 儀表板可透過 `/pulse` 路由存取。依預設，您只能在 `local` 環境中存取此儀表板，因此您需要透過自訂 `'viewPulse'` 授權閘門來為您的正式環境設定授權。您可以在應用程式的 `app/Providers/AppServiceProvider.php` 檔案中完成此操作：

```php
use App\Models\User;
use Illuminate\Support\Facades\Gate;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Gate::define('viewPulse', function (User $user) {
        return $user->isAdmin();
    });

    // ...
}
```


<a name="dashboard-customization"></a>
### 自訂

Pulse 儀表板的卡片和版面配置可透過發佈儀表板視圖來進行設定。儀表板視圖將會發佈到 `resources/views/vendor/pulse/dashboard.blade.php`：

```sh
php artisan vendor:publish --tag=pulse-dashboard
```

儀表板由 [Livewire](https://livewire.laravel.com/) 驅動，允許您自訂卡片和版面配置，而無需重新建構任何 JavaScript 資產。

在此檔案中，`<x-pulse>` 元件負責渲染儀表板並為卡片提供網格佈局。如果您希望儀表板佔據螢幕的整個寬度，您可以為該元件提供 `full-width` 屬性：

```blade
<x-pulse full-width>
    ...
</x-pulse>
```

依預設，`<x-pulse>` 元件將會建立一個 12 欄的網格，但您可以使用 `cols` 屬性來自訂：

```blade
<x-pulse cols="16">
    ...
</x-pulse>
```

每張卡片都接受 `cols` 和 `rows` 屬性來控制空間和位置：

```blade
<livewire:pulse.usage cols="4" rows="2" />
```

大多數卡片也接受 `expand` 屬性來顯示完整卡片而非捲動：

```blade
<livewire:pulse.slow-queries expand />
```


<a name="dashboard-resolving-users"></a>
### 解析使用者

對於顯示使用者資訊的卡片，例如 Application Usage 卡片，Pulse 只會記錄使用者的 ID。在渲染儀表板時，Pulse 會從您的預設 `Authenticatable` 模型解析 `name` 和 `email` 欄位，並使用 Gravatar 網路服務顯示頭像。

您可以在應用程式的 `App\Providers\AppServiceProvider` 類別中呼叫 `Pulse::user` 方法來自訂欄位和頭像。

`user` 方法接受一個閉包，該閉包將接收要顯示的 `Authenticatable` 模型，並應返回一個包含使用者 `name`、`extra` 和 `avatar` 資訊的陣列：

```php
use Laravel\Pulse\Facades\Pulse;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Pulse::user(fn ($user) => [
        'name' => $user->name,
        'extra' => $user->email,
        'avatar' => $user->avatar_url,
    ]);

    // ...
}
```

> [!NOTE]  
> 您可以透過實作 `Laravel\Pulse\Contracts\ResolvesUsers` 契約並將其綁定在 Laravel 的 [服務容器](/docs/{{version}}/container#binding-a-singleton) 中，來完全自訂驗證使用者如何被擷取和檢索。


<a name="dashboard-cards"></a>
### 卡片


<a name="servers-card"></a>
#### 伺服器

`<livewire:pulse.servers />` 卡片顯示所有執行 `pulse:check` 命令的伺服器的系統資源使用情況。有關系統資源報告的更多資訊，請參閱 [伺服器紀錄器](#servers-recorder) 的文件。

如果您更換基礎設施中的伺服器，您可能希望在指定時間後停止在 Pulse 儀表板中顯示不活動的伺服器。您可以透過使用 `ignore-after` 屬性來完成此操作，該屬性接受一個秒數，表示不活動的伺服器應從 Pulse 儀表板中移除的時間。另外，您也可以提供一個相對時間格式的字串，例如 `1 hour` 或 `3 days and 1 hour`：

```blade
<livewire:pulse.servers ignore-after="3 hours" />
```


<a name="application-usage-card"></a>
#### 應用程式使用

`<livewire:pulse.usage />` 卡片顯示了向您的應用程式發出請求、分派任務以及遇到慢請求的前 10 名使用者。

如果您希望同時在螢幕上查看所有使用情況指標，您可以多次包含此卡片並指定 `type` 屬性：

```blade
<livewire:pulse.usage type="requests" />
<livewire:pulse.usage type="slow_requests" />
<livewire:pulse.usage type="jobs" />
```

要了解如何自訂 Pulse 檢索和顯示使用者資訊，請查閱我們關於 [解析使用者](#dashboard-resolving-users) 的文件。

> [!NOTE]  
> 如果您的應用程式接收大量請求或分派大量任務，您可能希望啟用 [取樣](#sampling) 功能。請參閱 [使用者請求紀錄器](#user-requests-recorder)、[使用者任務紀錄器](#user-jobs-recorder) 和 [慢任務紀錄器](#slow-jobs-recorder) 文件以獲取更多資訊。


<a name="exceptions-card"></a>
#### 異常

`<livewire:pulse.exceptions />` 卡片顯示應用程式中發生異常的頻率和最新情況。依預設，異常會根據異常類別和發生位置進行分組。請參閱 [異常紀錄器](#exceptions-recorder) 文件以獲取更多資訊。


<a name="queues-card"></a>
#### 佇列

`<livewire:pulse.queues />` 卡片顯示應用程式中佇列的吞吐量，包括排入佇列、處理中、已處理、已釋放和失敗的任務數量。請參閱 [佇列紀錄器](#queues-recorder) 文件以獲取更多資訊。


<a name="slow-requests-card"></a>
#### 慢請求

`<livewire:pulse.slow-requests />` 卡片顯示應用程式中超過設定閾值 (預設為 1,000 毫秒) 的傳入請求。請參閱 [慢請求紀錄器](#slow-requests-recorder) 文件以獲取更多資訊。


<a name="slow-jobs-card"></a>
#### 慢任務

`<livewire:pulse.slow-jobs />` 卡片顯示應用程式中超過設定閾值 (預設為 1,000 毫秒) 的排入佇列任務。請參閱 [慢任務紀錄器](#slow-jobs-recorder) 文件以獲取更多資訊。


<a name="slow-queries-card"></a>
#### 慢查詢

`<livewire:pulse.slow-queries />` 卡片顯示應用程式中超過設定閾值 (預設為 1,000 毫秒) 的資料庫查詢。

依預設，慢查詢會根據 SQL 查詢 (不含綁定) 和發生位置進行分組，但如果您只希望根據 SQL 查詢進行分組，則可以選擇不擷取位置。

如果您遇到因極大的 SQL 查詢接收語法高亮顯示而導致的渲染效能問題，您可以透過添加 `without-highlighting` 屬性來禁用高亮顯示：

```blade
<livewire:pulse.slow-queries without-highlighting />
```

請參閱 [慢查詢紀錄器](#slow-queries-recorder) 文件以獲取更多資訊。


<a name="slow-outgoing-requests-card"></a>
#### 慢速外送請求

`<livewire:pulse.slow-outgoing-requests />` 卡片顯示使用 Laravel 的 [HTTP client](/docs/{{version}}/http-client) 發出且超過設定閾值 (預設為 1,000 毫秒) 的外送請求。

依預設，條目將按完整 URL 分組。但是，您可能希望使用正規表達式來規範化或分組類似的外送請求。請參閱 [慢速外送請求紀錄器](#slow-outgoing-requests-recorder) 文件以獲取更多資訊。


<a name="cache-card"></a>
#### 快取

`<livewire:pulse.cache />` 卡片顯示應用程式的快取命中和未命中統計資訊，包括全域和個別鍵值。

依預設，條目將按鍵值分組。但是，您可能希望使用正規表達式來規範化或分組類似的鍵值。請參閱 [快取互動紀錄器](#cache-interactions-recorder) 文件以獲取更多資訊。

<a name="capturing-entries"></a>
## 擷取資料

大多數 Pulse 紀錄器會根據由 Laravel 觸發的框架事件自動擷取資料。然而，[伺服器紀錄器](#servers-recorder) 和一些第三方卡片必須定期輪詢資訊。若要使用這些卡片，您必須在所有個別的應用程式伺服器上執行 `pulse:check` 守護行程：

```php
php artisan pulse:check
```

> [!NOTE]  
> 若要讓 `pulse:check` 程序在背景永久執行，您應該使用程序監控器，例如 Supervisor，來確保該指令不會停止執行。

由於 `pulse:check` 指令是一個長期運行的程序，它在不重新啟動的情況下將無法偵測到程式碼庫的變更。您應該在應用程式的部署流程中，透過呼叫 `pulse:restart` 指令來優雅地重新啟動該指令：

```sh
php artisan pulse:restart
```

> [!NOTE]  
> Pulse 使用 [快取](/docs/{{version}}/cache) 來儲存重新啟動訊號，因此您應該確認應用程式的快取驅動程式已正確配置，才能使用此功能。

<a name="recorders"></a>
### 紀錄器

紀錄器負責從應用程式中擷取資料並儲存至 Pulse 資料庫。紀錄器會在 [Pulse 設定檔](#configuration) 的 `recorders` 區塊中進行註冊與設定。


<a name="cache-interactions-recorder"></a>
#### 快取互動

`CacheInteractions` 紀錄器會擷取關於應用程式中 [快取](/docs/{{version}}/cache) 的命中與未命中資訊，以便在 [快取](#cache-card) 卡片上顯示。

您可以選擇性地調整 [取樣率](#sampling) 和忽略的鍵值模式。

您還可以設定鍵值分組，讓相似的鍵值分組為單一項目。例如，您可能希望從快取相同類型資訊的鍵值中移除唯一 ID。分組是透過使用正規表達式「尋找並取代」鍵值中的部分來進行設定。範例已包含在設定檔中：

```php
Recorders\CacheInteractions::class => [
    // ...
    'groups' => [
        // '/:\d+/' => ':*',
    ],
],
```

將會使用第一個匹配到的模式。如果沒有任何模式匹配，則該鍵值將會被完整擷取。


<a name="exceptions-recorder"></a>
#### 異常

`Exceptions` 紀錄器會擷取關於應用程式中可報告異常的資訊，以便在 [異常](#exceptions-card) 卡片上顯示。

您可以選擇性地調整 [取樣率](#sampling) 和忽略的異常模式。您還可以設定是否擷取異常發生的位置。擷取到的位置將顯示在 Pulse 儀表板上，有助於追蹤異常來源；然而，如果同一個異常在多個位置發生，則會針對每個唯一位置多次出現。


<a name="queues-recorder"></a>
#### 佇列

`Queues` 紀錄器會擷取關於應用程式佇列的資訊，以便在 [佇列](#queues-card) 卡片上顯示。

您可以選擇性地調整 [取樣率](#sampling) 和忽略的任務模式。


<a name="slow-jobs-recorder"></a>
#### 慢速任務

`SlowJobs` 紀錄器會擷取關於應用程式中慢速任務的資訊，以便在 [慢速任務](#slow-jobs-recorder) 卡片上顯示。

您可以選擇性地調整慢速任務閾值、[取樣率](#sampling) 和忽略的任務模式。

您可能會有某些任務預期會花費比其他任務更長的時間。在這些情況下，您可以設定每個任務的閾值：

```php
Recorders\SlowJobs::class => [
    // ...
    'threshold' => [
        '#^App\\Jobs\\GenerateYearlyReports$#' => 5000,
        'default' => env('PULSE_SLOW_JOBS_THRESHOLD', 1000),
    ],
],
```

如果沒有任何正規表達式模式匹配該任務的類別名稱，則會使用 `'default'` 值。


<a name="slow-outgoing-requests-recorder"></a>
#### 慢速對外請求

`SlowOutgoingRequests` 紀錄器會擷取關於使用 Laravel [HTTP client](/docs/{{version}}/http-client) 發出，且超出設定閾值的對外 HTTP 請求資訊，以便在 [慢速對外請求](#slow-outgoing-requests-card) 卡片上顯示。

您可以選擇性地調整慢速對外請求閾值、[取樣率](#sampling) 和忽略的 URL 模式。

您可能會有某些對外請求預期會花費比其他請求更長的時間。在這些情況下，您可以設定每個請求的閾值：

```php
Recorders\SlowOutgoingRequests::class => [
    // ...
    'threshold' => [
        '#backup.zip$#' => 5000,
        'default' => env('PULSE_SLOW_OUTGOING_REQUESTS_THRESHOLD', 1000),
    ],
],
```

如果沒有任何正規表達式模式匹配該請求的 URL，則會使用 `'default'` 值。

您還可以設定 URL 分組，讓相似的 URL 分組為單一項目。例如，您可能希望從 URL 路徑中移除唯一 ID，或者僅按網域分組。分組是透過使用正規表達式「尋找並取代」URL 中的部分來進行設定。一些範例已包含在設定檔中：

```php
Recorders\SlowOutgoingRequests::class => [
    // ...
    'groups' => [
        // '#^https://api\.github\.com/repos/.*$#' => 'api.github.com/repos/*',
        // '#^https?://([^/]*).*$#' => '\1',
        // '#/\d+#' => '/*',
    ],
],
```

將會使用第一個匹配到的模式。如果沒有任何模式匹配，則該 URL 將會被完整擷取。


<a name="slow-queries-recorder"></a>
#### 慢速查詢

`SlowQueries` 紀錄器會擷取應用程式中超出設定閾值的任何資料庫查詢，以便在 [慢速查詢](#slow-queries-card) 卡片上顯示。

您可以選擇性地調整慢速查詢閾值、[取樣率](#sampling) 和忽略的查詢模式。您還可以設定是否擷取查詢的位置。擷取到的位置將顯示在 Pulse 儀表板上，有助於追蹤查詢來源；然而，如果同一個查詢在多個位置執行，則會針對每個唯一位置多次出現。

您可能會有某些查詢預期會花費比其他查詢更長的時間。在這些情況下，您可以設定每個查詢的閾值：

```php
Recorders\SlowQueries::class => [
    // ...
    'threshold' => [
        '#^insert into `yearly_reports`#' => 5000,
        'default' => env('PULSE_SLOW_QUERIES_THRESHOLD', 1000),
    ],
],
```

如果沒有任何正規表達式模式匹配該查詢的 SQL，則會使用 `'default'` 值。


<a name="slow-requests-recorder"></a>
#### 慢速請求

`Requests` 紀錄器會擷取關於應用程式請求的資訊，以便在 [慢速請求](#slow-requests-card) 和 [應用程式使用量](#application-usage-card) 卡片上顯示。

您可以選擇性地調整慢速路由閾值、[取樣率](#sampling) 和忽略的路徑。

您可能會有某些請求預期會花費比其他請求更長的時間。在這些情況下，您可以設定每個請求的閾值：

```php
Recorders\SlowRequests::class => [
    // ...
    'threshold' => [
        '#^/admin/#' => 5000,
        'default' => env('PULSE_SLOW_REQUESTS_THRESHOLD', 1000),
    ],
],
```

如果沒有任何正規表達式模式匹配該請求的 URL，則會使用 `'default'` 值。


<a name="servers-recorder"></a>
#### 伺服器

`Servers` 紀錄器會擷取為應用程式提供服務的伺服器的 CPU、記憶體和儲存空間使用量，以便在 [伺服器](#servers-card) 卡片上顯示。此紀錄器需要您希望監控的每個伺服器上都執行 [`pulse:check` 指令](#capturing-entries)。

每個報告伺服器都必須具有唯一的名稱。預設情況下，Pulse 會使用 PHP `gethostname` 函數返回的值。如果您希望自訂此設定，可以設定 `PULSE_SERVER_NAME` 環境變數：

```env
PULSE_SERVER_NAME=load-balancer
```

Pulse 設定檔也允許您自訂要監控的目錄。


<a name="user-jobs-recorder"></a>
#### 使用者任務

`UserJobs` 紀錄器會擷取關於在應用程式中分派任務的使用者資訊，以便在 [應用程式使用量](#application-usage-card) 卡片上顯示。

您可以選擇性地調整 [取樣率](#sampling) 和忽略的任務模式。


<a name="user-requests-recorder"></a>
#### 使用者請求

`UserRequests` 紀錄器會擷取關於對應用程式發出請求的使用者資訊，以便在 [應用程式使用量](#application-usage-card) 卡片上顯示。

您可以選擇性地調整 [取樣率](#sampling) 和忽略的 URL 模式。

<a name="filtering"></a>
### 篩選

如我們所見，許多 [紀錄器](#recorders) 透過設定提供「忽略」傳入的資料項目的能力，這些忽略的依據是資料項目本身的值，例如請求的 URL。然而，有時根據其他因素（例如目前已驗證的使用者）來篩選記錄會很有用。要篩選掉這些記錄，您可以將閉包傳遞給 Pulse 的 `filter` 方法。通常，`filter` 方法應在應用程式的 `AppServiceProvider` 檔案中的 `boot` 方法內呼叫：

```php
use Illuminate\Support\Facades\Auth;
use Laravel\Pulse\Entry;
use Laravel\Pulse\Facades\Pulse;
use Laravel\Pulse\Value;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Pulse::filter(function (Entry|Value $entry) {
        return Auth::user()->isNotAdmin();
    });

    // ...
}
```

<a name="performance"></a>
## 效能

Pulse 的設計旨在無須額外基礎設施，即可輕鬆整合至現有應用程式。然而，對於高流量的應用程式，有多種方法可以消除 Pulse 對應用程式效能可能造成的任何影響。

<a name="using-a-different-database"></a>
### 使用不同的資料庫

對於高流量的應用程式，您可能會希望為 Pulse 使用專用的資料庫連線，以避免影響您的應用程式資料庫。

您可以透過設定 `PULSE_DB_CONNECTION` 環境變數，來自訂 Pulse 所使用的 [資料庫連線](/docs/{{version}}/database#configuration)。

```env
PULSE_DB_CONNECTION=pulse
```

<a name="ingest"></a>
### Redis 注入

> [!WARNING]
> Redis 注入 (Redis Ingest) 需要 Redis 6.2 或更高版本，並且應用程式必須設定 `phpredis` 或 `predis` 作為 Redis 用戶端驅動程式。

預設情況下，Pulse 會在 HTTP 回應傳送給用戶端或處理完畢工作 (job) 後，直接將資料儲存到 [已設定的資料庫連線](#using-a-different-database) 中；然而，您可以使用 Pulse 的 Redis ingest driver 將資料傳送到 Redis stream。您可以透過設定 `PULSE_INGEST_DRIVER` 環境變數來啟用此功能：

```
PULSE_INGEST_DRIVER=redis
```

Pulse 預設將使用您的預設 [Redis 連線](/docs/{{version}}/redis#configuration)，但您可以透過 `PULSE_REDIS_CONNECTION` 環境變數來進行自訂：

```
PULSE_REDIS_CONNECTION=pulse
```

當使用 Redis ingest 時，您需要執行 `pulse:work` 指令來監控資料流，並將資料從 Redis 移至 Pulse 的資料庫表格中。

```php
php artisan pulse:work
```

> [!NOTE]
> 為了讓 `pulse:work` 程序在背景永久運行，您應該使用像 Supervisor 這樣的程序監控器，以確保 Pulse worker 不會停止運行。

由於 `pulse:work` 指令是一個長時間運行的程序，若不重新啟動，它將不會感知程式碼庫的變更。您應該在應用程式的部署過程中，透過呼叫 `pulse:restart` 指令來優雅地重新啟動此指令：

```sh
php artisan pulse:restart
```

> [!NOTE]
> Pulse 使用 [快取](/docs/{{version}}/cache) 來儲存重新啟動的訊號，因此在使用此功能之前，您應該驗證您的應用程式是否已正確設定快取驅動程式。

<a name="sampling"></a>
### 取樣

預設情況下，Pulse 會擷取應用程式中發生的每個相關事件。對於高流量的應用程式，這可能導致儀表板需要聚合數百萬條資料庫行，尤其是在較長的時間段內。

您也可以選擇在某些 Pulse 資料記錄器上啟用「取樣」功能。例如，在 [`使用者請求`](#user-requests-recorder) 記錄器上將取樣率設定為 `0.1`，表示您只會記錄應用程式大約 10% 的請求。在儀表板中，這些值將會被放大並加上 `~` 前綴，以表示它們是近似值。

一般而言，特定指標的資料點越多，您就可以在不犧牲太多準確性的前提下，安全地將取樣率設定得更低。

<a name="trimming"></a>
### 修剪

Pulse 會在儲存的資料點超出儀表板顯示範圍時自動修剪。修剪 (Trimming) 是在注入資料時透過抽獎系統進行的，此系統可以在 Pulse [設定檔](#configuration) 中進行自訂。

<a name="pulse-exceptions"></a>
### 處理 Pulse 異常

如果在擷取 Pulse 資料時發生異常，例如無法連接到儲存資料庫，Pulse 將會靜默失敗，以避免影響您的應用程式。

如果您希望自訂這些異常的處理方式，您可以為 `handleExceptionsUsing` 方法提供一個閉包 (closure)：

```php
use Laravel\Pulse\Facades\Pulse;
use Illuminate\Support\Facades\Log;

Pulse::handleExceptionsUsing(function ($e) {
    Log::debug('An exception happened in Pulse', [
        'message' => $e->getMessage(),
        'stack' => $e->getTraceAsString(),
    ]);
});
```

<a name="custom-cards"></a>
## 自訂卡片

Pulse 讓您可以建立自訂卡片，以顯示與您應用程式特定需求相關的資料。Pulse 使用 [Livewire](https://livewire.laravel.com)，因此您可能需要先 [查閱其文件](https://livewire.laravel.com/docs)，然後再建立您的第一張自訂卡片。


<a name="custom-card-components"></a>
### 卡片元件

在 Laravel Pulse 中建立自訂卡片，首先要擴展基礎的 `Card` Livewire 元件並定義一個對應的視圖：

```php
namespace App\Livewire\Pulse;

use Laravel\Pulse\Livewire\Card;
use Livewire\Attributes\Lazy;

#[Lazy]
class TopSellers extends Card
{
    public function render()
    {
        return view('livewire.pulse.top-sellers');
    }
}
```

當使用 Livewire 的 [延遲載入](https://livewire.laravel.com/docs/lazy) 功能時，`Card` 元件會自動提供一個預留位置，該位置遵循傳遞給您元件的 `cols` 和 `rows` 屬性。

在編寫 Pulse 卡片對應的視圖時，您可以利用 Pulse 的 Blade 元件來獲得一致的外觀和風格：

```blade
<x-pulse::card :cols="$cols" :rows="$rows" :class="$class" wire:poll.5s="">
    <x-pulse::card-header name="Top Sellers">
        <x-slot:icon>
            ...
        </x-slot:icon>
    </x-pulse::card-header>

    <x-pulse::scroll :expand="$expand">
        ...
    </x-pulse::scroll>
</x-pulse::card>
```

`$cols`、`$rows`、`$class` 和 `$expand` 變數應傳遞給其各自的 Blade 元件，以便卡片佈局可以在儀表板視圖中自訂。您可能還希望在視圖中包含 `wire:poll.5s=""` 屬性，以讓卡片自動更新。

一旦您定義了您的 Livewire 元件和範本，該卡片就可以包含在您的 [儀表板視圖](#dashboard-customization) 中：

```blade
<x-pulse>
    ...

    <livewire:pulse.top-sellers cols="4" />
</x-pulse>
```

> [!NOTE]  
> 如果您的卡片包含在套件中，您需要使用 `Livewire::component` 方法向 Livewire 註冊該元件。


<a name="custom-card-styling"></a>
### 樣式

如果您的卡片需要 Pulse 內建的類別和元件之外的額外樣式，有幾種選項可以為您的卡片包含自訂 CSS。


<a name="custom-card-styling-vite"></a>
#### Laravel Vite 整合

如果您的自訂卡片位於您應用程式的程式碼庫中，並且您正在使用 Laravel 的 [Vite 整合](/docs/{{version}}/vite)，您可以更新您的 `vite.config.js` 檔案，以包含一個專用的 CSS 進入點給您的卡片：

```js
laravel({
    input: [
        'resources/css/pulse/top-sellers.css',
        // ...
    ],
}),
```

然後您可以在您的 [儀表板視圖](#dashboard-customization) 中使用 `@vite` Blade 指令，指定該卡片的 CSS 進入點：

```blade
<x-pulse>
    @vite('resources/css/pulse/top-sellers.css')

    ...
</x-pulse>
```


<a name="custom-card-styling-css"></a>
#### CSS 檔案

對於其他使用情境，包括包含在套件中的 Pulse 卡片，您可以指示 Pulse 載入額外的樣式表，方法是在您的 Livewire 元件上定義一個 `css` 方法，該方法回傳您的 CSS 檔案路徑：

```php
class TopSellers extends Card
{
    // ...

    protected function css()
    {
        return __DIR__.'/../../dist/top-sellers.css';
    }
}
```

當此卡片包含在儀表板上時，Pulse 將自動在 `<style>` 標籤內包含此檔案的內容，因此它不需要發佈到 `public` 目錄。


<a name="custom-card-styling-tailwind"></a>
#### Tailwind CSS

使用 Tailwind CSS 時，您應該建立一個專用的 Tailwind 設定檔，以避免載入不必要的 CSS 或與 Pulse 的 Tailwind 類別產生衝突：

```js
export default {
    darkMode: 'class',
    important: '#top-sellers',
    content: [
        './resources/views/livewire/pulse/top-sellers.blade.php',
    ],
    corePlugins: {
        preflight: false,
    },
};
```

然後您可以在您的 CSS 進入點中指定該設定檔：

```css
@config "../../tailwind.top-sellers.config.js";
@tailwind base;
@tailwind components;
@tailwind utilities;
```

您還需要在您的卡片視圖中包含一個 `id` 或 `class` 屬性，該屬性與傳遞給 Tailwind 的 [`important` 選擇器策略](https://tailwindcss.com/docs/configuration#selector-strategy) 的選擇器匹配：

```blade
<x-pulse::card id="top-sellers" :cols="$cols" :rows="$rows" class="$class">
    ...
</x-pulse::card>
```

<a name="custom-card-data"></a>
### 資料擷取與聚合

自訂卡片可以從任何地方擷取並顯示資料；然而，您可能會希望利用 Pulse 強大且高效的資料記錄與聚合系統。


<a name="custom-card-data-capture"></a>
#### 擷取資料

Pulse 允許您使用 `Pulse::record` 方法記錄「資料項目 (entries)」：

```php
use Laravel\Pulse\Facades\Pulse;

Pulse::record('user_sale', $user->id, $sale->amount)
    ->sum()
    ->count();
```

提供給 `record` 方法的第一個引數是您正在記錄的資料項目的 `type` (類型)，而第二個引數是決定聚合資料如何分組的 `key` (鍵)。對於大多數聚合方法，您還需要指定一個 `value` (值) 以進行聚合。在上述範例中，正在聚合的值是 `$sale->amount`。然後，您可以呼叫一個或多個聚合方法 (例如 `sum`)，以便 Pulse 可以將預先聚合的值擷取到「儲存桶 (buckets)」中，以便日後高效檢索。

可用的聚合方法包括：

*   `avg`
*   `count`
*   `max`
*   `min`
*   `sum`

> [!NOTE]  
> 當建立一個擷取目前已驗證使用者 ID 的卡片套件時，您應該使用 `Pulse::resolveAuthenticatedUserId()` 方法，該方法會尊重應用程式中對 [使用者解析器所做的自訂](#dashboard-resolving-users)。


<a name="custom-card-data-retrieval"></a>
#### 擷取聚合資料

當擴充 Pulse 的 `Card` Livewire 元件時，您可以使用 `aggregate` 方法擷取儀表板中正在查看期間的聚合資料：

```php
class TopSellers extends Card
{
    public function render()
    {
        return view('livewire.pulse.top-sellers', [
            'topSellers' => $this->aggregate('user_sale', ['sum', 'count'])
        ]);
    }
}
```

`aggregate` 方法會回傳一個 PHP `stdClass` 物件的集合。每個物件都將包含先前擷取的 `key` 屬性，以及每個請求聚合的鍵：

```
@foreach ($topSellers as $seller)
    {{ $seller->key }}
    {{ $seller->sum }}
    {{ $seller->count }}
@endforeach
```

Pulse 將主要從預先聚合的儲存桶中擷取資料；因此，指定的聚合必須已使用 `Pulse::record` 方法預先擷取。最舊的儲存桶通常會部分超出指定期間，因此 Pulse 會聚合最舊的資料項目以填補空白，並為整個期間提供準確的值，而無需在每次輪詢請求時聚合整個期間。

您也可以使用 `aggregateTotal` 方法擷取給定類型的總值。例如，以下方法將擷取所有使用者銷售的總和，而不是按使用者分組。

```php
$total = $this->aggregateTotal('user_sale', 'sum');
```


<a name="custom-card-displaying-users"></a>
#### 顯示使用者

當處理將使用者 ID 記錄為鍵的聚合時，您可以使用 `Pulse::resolveUsers` 方法將鍵解析為使用者記錄：

```php
$aggregates = $this->aggregate('user_sale', ['sum', 'count']);

$users = Pulse::resolveUsers($aggregates->pluck('key'));

return view('livewire.pulse.top-sellers', [
    'sellers' => $aggregates->map(fn ($aggregate) => (object) [
        'user' => $users->find($aggregate->key),
        'sum' => $aggregate->sum,
        'count' => $aggregate->count,
    ])
]);
```

`find` 方法會回傳一個包含 `name`、`extra` 和 `avatar` 鍵的物件，您可以選擇直接將其傳遞給 `<x-pulse::user-card>` Blade 元件：

```blade
<x-pulse::user-card :user="{{ $seller->user }}" :stats="{{ $seller->sum }}" />
```


<a name="custom-recorders"></a>
#### 自訂紀錄器

套件作者可能希望提供紀錄器類別，以允許使用者設定資料的擷取。

紀錄器在應用程式的 `config/pulse.php` 設定檔中的 `recorders` 區塊中註冊：

```php
[
    // ...
    'recorders' => [
        Acme\Recorders\Deployments::class => [
            // ...
        ],

        // ...
    ],
]
```

紀錄器可以透過指定 `$listen` 屬性來監聽事件。Pulse 將自動註冊監聽器並呼叫紀錄器的 `record` 方法：

```php
<?php

namespace Acme\Recorders;

use Acme\Events\Deployment;
use Illuminate\Support\Facades\Config;
use Laravel\Pulse\Facades\Pulse;

class Deployments
{
    /**
     * The events to listen for.
     *
     * @var array<int, class-string>
     */
    public array $listen = [
        Deployment::class,
    ];

    /**
     * Record the deployment.
     */
    public function record(Deployment $event): void
    {
        $config = Config::get('pulse.recorders.'.static::class);

        Pulse::record(
            // ...
        );
    }
}
```