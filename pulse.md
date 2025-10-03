# Laravel Pulse

- [簡介](#introduction)
- [安裝](#installation)
    - [設定](#configuration)
- [儀表板](#dashboard)
    - [授權](#dashboard-authorization)
    - [自訂](#dashboard-customization)
    - [解析使用者](#dashboard-resolving-users)
    - [卡片](#dashboard-cards)
- [擷取記錄](#capturing-entries)
    - [記錄器](#recorders)
    - [過濾](#filtering)
- [效能](#performance)
    - [使用不同資料庫](#using-a-different-database)
    - [Redis 攝取](#ingest)
    - [抽樣](#sampling)
    - [修剪](#trimming)
    - [處理 Pulse 異常](#pulse-exceptions)
- [自訂卡片](#custom-cards)
    - [卡片元件](#custom-card-components)
    - [樣式](#custom-card-styling)
    - [資料擷取與聚合](#custom-card-data)

<a name="introduction"></a>
## 簡介

[Laravel Pulse](https://github.com/laravel/pulse) 提供您應用程式效能與使用情況的一目瞭然洞察。透過 Pulse，您可以追查執行緩慢的 jobs 與端點等瓶頸，找出最活躍的使用者等等。

若要對個別事件進行深入偵錯，請參閱 [Laravel Telescope](/docs/{{version}}/telescope)。


<a name="installation"></a>
## 安裝

> [!WARNING]
> Pulse 的第一方儲存實作目前需要 MySQL、MariaDB 或 PostgreSQL 資料庫。如果您使用的是不同的資料庫引擎，您將需要為您的 Pulse 資料準備一個獨立的 MySQL、MariaDB 或 PostgreSQL 資料庫。

您可以使用 Composer 套件管理器安裝 Pulse：

```shell
composer require laravel/pulse
```

接著，您應該使用 `vendor:publish` Artisan 指令來發佈 Pulse 的設定檔與遷移檔：

```shell
php artisan vendor:publish --provider="Laravel\Pulse\PulseServiceProvider"
```

最後，您應該執行 `migrate` 指令以建立儲存 Pulse 資料所需的資料表：

```shell
php artisan migrate
```

一旦 Pulse 的資料庫遷移已執行完畢，您就可以透過 `/pulse` 路由存取 Pulse 儀表板。

> [!NOTE]
> 如果您不想將 Pulse 資料儲存在應用程式的主要資料庫中，您可以[指定一個專屬的資料庫連線](#using-a-different-database)。


<a name="configuration"></a>
### 設定

許多 Pulse 的設定選項都可以透過環境變數來控制。若要查看可用選項、註冊新的記錄器，或配置進階選項，您可以發佈 `config/pulse.php` 設定檔：

```shell
php artisan vendor:publish --tag=pulse-config
```

<a name="dashboard"></a>
## 儀表板


<a name="dashboard-authorization"></a>
### 授權

Pulse 儀表板可透過 `/pulse` 路由存取。預設情況下，您只能在 `local` 環境中存取此儀表板，因此您需要透過自訂 `'viewPulse'` 授權 gate 來為生產環境配置授權。您可以在應用程式的 `app/Providers/AppServiceProvider.php` 檔案中完成此操作：

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

Pulse 儀表板的卡片和佈局可以透過發佈儀表板視圖來配置。儀表板視圖將會發佈到 `resources/views/vendor/pulse/dashboard.blade.php`：

```shell
php artisan vendor:publish --tag=pulse-dashboard
```

儀表板由 [Livewire](https://livewire.laravel.com/) 提供支援，允許您自訂卡片和佈局，而無需重建任何 JavaScript 資源。

在此檔案中，`<x-pulse>` 元件負責呈現儀表板並為卡片提供網格佈局。如果您希望儀表板佔滿整個螢幕寬度，您可以為元件提供 `full-width` prop：

```blade
<x-pulse full-width>
    ...
</x-pulse>
```

預設情況下，`<x-pulse>` 元件將建立一個 12 欄的網格，但您可以使用 `cols` prop 自訂此設定：

```blade
<x-pulse cols="16">
    ...
</x-pulse>
```

每個卡片都接受 `cols` 和 `rows` prop 來控制空間和定位：

```blade
<livewire:pulse.usage cols="4" rows="2" />
```

大多數卡片也接受 `expand` prop 以顯示完整卡片而非捲動：

```blade
<livewire:pulse.slow-queries expand />
```


<a name="dashboard-resolving-users"></a>
### 解析使用者

對於顯示使用者資訊的卡片，例如「應用程式使用量」卡片，Pulse 只會記錄使用者的 ID。渲染儀表板時，Pulse 會從您預設的 `Authenticatable` 模型解析 `name` 和 `email` 欄位，並使用 Gravatar 網頁服務顯示頭像。

您可以在應用程式的 `App\Providers\AppServiceProvider` 類別中調用 `Pulse::user` 方法來自訂欄位和頭像。

`user` 方法接受一個閉包，該閉包將接收要顯示的 `Authenticatable` 模型，並且應返回一個包含使用者 `name`、`extra` 和 `avatar` 資訊的陣列：

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
> 您可以透過實作 `Laravel\Pulse\Contracts\ResolvesUsers` 契約並將其綁定到 Laravel 的 [服務容器](/docs/{{version}}/container#binding-a-singleton) 中，來完全自訂驗證使用者如何被擷取和檢索。


<a name="dashboard-cards"></a>
### 卡片


<a name="servers-card"></a>
#### 伺服器

`<livewire:pulse.servers />` 卡片顯示所有執行 `pulse:check` 命令的伺服器的系統資源使用情況。請參考 [伺服器記錄器](#servers-recorder) 的文件，以獲取有關系統資源報告的更多資訊。

如果您更換了基礎設施中的伺服器，您可能希望在指定的時間後停止在 Pulse 儀表板中顯示非活躍的伺服器。您可以使用 `ignore-after` prop 來完成此操作，該 prop 接受非活躍伺服器應從 Pulse 儀表板中移除的秒數。或者，您可以提供一個相對時間格式的字串，例如 `1 hour` 或 `3 days and 1 hour`：

```blade
<livewire:pulse.servers ignore-after="3 hours" />
```


<a name="application-usage-card"></a>
#### 應用程式使用量

`<livewire:pulse.usage />` 卡片顯示前 10 名向您的應用程式發出請求、分派工作和經歷慢速請求的使用者。

如果您希望同時在螢幕上查看所有使用量指標，您可以多次包含此卡片並指定 `type` 屬性：

```blade
<livewire:pulse.usage type="requests" />
<livewire:pulse.usage type="slow_requests" />
<livewire:pulse.usage type="jobs" />
```

要了解如何自訂 Pulse 檢索和顯示使用者資訊的方式，請查閱我們關於 [解析使用者](#dashboard-resolving-users) 的文件。

> [!NOTE]
> 如果您的應用程式接收大量請求或分派大量工作，您可能希望啟用 [抽樣](#sampling)。請參考 [使用者請求記錄器](#user-requests-recorder)、[使用者工作記錄器](#user-jobs-recorder) 和 [慢速工作記錄器](#slow-jobs-recorder) 文件以獲取更多資訊。


<a name="exceptions-card"></a>
#### 異常

`<livewire:pulse.exceptions />` 卡片顯示您的應用程式中發生異常的頻率和最新情況。預設情況下，異常會根據異常類別和發生的位置進行分組。請參考 [異常記錄器](#exceptions-recorder) 文件以獲取更多資訊。


<a name="queues-card"></a>
#### 佇列

`<livewire:pulse.queues />` 卡片顯示您應用程式中佇列的吞吐量，包括已排入佇列、正在處理、已處理、已釋放和失敗的工作數量。請參考 [佇列記錄器](#queues-recorder) 文件以獲取更多資訊。


<a name="slow-requests-card"></a>
#### 慢速請求

`<livewire:pulse.slow-requests />` 卡片顯示您的應用程式中超過預設 1,000 毫秒配置閾值的傳入請求。請參考 [慢速請求記錄器](#slow-requests-recorder) 文件以獲取更多資訊。


<a name="slow-jobs-card"></a>
#### 慢速工作

`<livewire:pulse.slow-jobs />` 卡片顯示您的應用程式中超過預設 1,000 毫秒配置閾值的佇列工作。請參考 [慢速工作記錄器](#slow-jobs-recorder) 文件以獲取更多資訊。


<a name="slow-queries-card"></a>
#### 慢速查詢

`<livewire:pulse.slow-queries />` 卡片顯示您的應用程式中超過預設 1,000 毫秒配置閾值的資料庫查詢。

預設情況下，慢速查詢會根據 SQL 查詢（不含綁定）及其發生的位置進行分組，但如果您希望僅依 SQL 查詢進行分組，則可以選擇不擷取位置。

如果您因極大的 SQL 查詢接收語法高亮顯示而遇到渲染效能問題，您可以透過新增 `without-highlighting` prop 來停用高亮顯示：

```blade
<livewire:pulse.slow-queries without-highlighting />
```

請參考 [慢速查詢記錄器](#slow-queries-recorder) 文件以獲取更多資訊。


<a name="slow-outgoing-requests-card"></a>
#### 慢速對外請求

`<livewire:pulse.slow-outgoing-requests />` 卡片顯示使用 Laravel 的 [HTTP client](/docs/{{version}}/http-client) 發出的對外請求，這些請求超過預設 1,000 毫秒的配置閾值。

預設情況下，條目將按完整 URL 分組。不過，您可能希望使用正規表達式來標準化或分組類似的對外請求。請參考 [慢速對外請求記錄器](#slow-outgoing-requests-recorder) 文件以獲取更多資訊。


<a name="cache-card"></a>
#### 快取

`<livewire:pulse.cache />` 卡片顯示您的應用程式的快取命中和未命中統計資料，包括全域和個別鍵。

預設情況下，條目將按鍵分組。不過，您可能希望使用正規表達式來標準化或分組類似的鍵。請參考 [快取互動記錄器](#cache-interactions-recorder) 文件以獲取更多資訊。

<a name="capturing-entries"></a>
## 擷取記錄

大多數 Pulse 記錄器會根據 Laravel 觸發的框架事件自動擷取記錄。然而，[伺服器記錄器](#servers-recorder)和一些第三方卡片必須定期輪詢資訊。要使用這些卡片，您必須在所有個別的應用程式伺服器上執行 `pulse:check` 守護行程：

```php
php artisan pulse:check
```

> [!NOTE]
> 為了讓 `pulse:check` 程序在背景永久執行，您應該使用程序監控器，例如 Supervisor，以確保該指令不會停止執行。

由於 `pulse:check` 指令是一個長時執行的程序，它在不重新啟動的情況下，將無法偵測到程式碼庫的變更。您應該在應用程式的部署過程中，透過呼叫 `pulse:restart` 指令來優雅地重新啟動它：

```shell
php artisan pulse:restart
```

> [!NOTE]
> Pulse 使用 [快取](/docs/{{version}}/cache)來儲存重新啟動訊號，因此在您使用此功能之前，應該驗證應用程式的快取驅動程式是否已正確設定。

<a name="recorders"></a>
### 記錄器

記錄器負責擷取您應用程式中的記錄，以便記錄到 Pulse 資料庫中。記錄器會在 [Pulse 設定檔](#configuration) 的 `recorders` 區塊中註冊並設定。

<a name="cache-interactions-recorder"></a>
#### 快取互動

`CacheInteractions` 記錄器會擷取關於您的應用程式中發生的 [快取](/docs/{{version}}/cache) 命中與未命中的資訊，以顯示在 [快取](#cache-card) 卡片上。

您可選擇調整 [抽樣率](#sampling) 和忽略的鍵模式。

您也可以設定鍵分組，以便將相似的鍵分組為單一記錄。例如，您可能希望從快取相同資訊類型的鍵中移除唯一的 ID。分組是使用正規表達式來設定，以「尋找並取代」鍵中的部分。設定檔中包含一個範例：

```php
Recorders\CacheInteractions::class => [
    // ...
    'groups' => [
        // '/:\d+/' => ':*',
    ],
],
```

第一個匹配的模式將會被使用。如果沒有任何模式匹配，那麼該鍵將會按原樣擷取。

<a name="exceptions-recorder"></a>
#### 異常

`Exceptions` 記錄器會擷取關於您的應用程式中發生的可報告異常的資訊，以顯示在 [異常](#exceptions-card) 卡片上。

您可選擇調整 [抽樣率](#sampling) 和忽略的異常模式。您也可以設定是否要擷取異常的原始位置。擷取的位置將會顯示在 Pulse 儀表板上，這有助於追蹤異常的來源；然而，如果相同的異常發生在多個位置，那麼它將會針對每個唯一位置顯示多次。

<a name="queues-recorder"></a>
#### 佇列

`Queues` 記錄器會擷取關於您應用程式佇列的資訊，以顯示在 [佇列](#queues-card) 卡片上。

您可選擇調整 [抽樣率](#sampling) 和忽略的任務模式。

<a name="slow-jobs-recorder"></a>
#### 慢任務

`SlowJobs` 記錄器會擷取關於您的應用程式中發生的慢任務的資訊，以顯示在 [慢任務](#slow-jobs-recorder) 卡片上。

您可選擇調整慢任務閾值、[抽樣率](#sampling) 和忽略的任務模式。

您可能有一些預計會比其他任務花費更長時間的任務。在這些情況下，您可以設定單一任務的閾值：

```php
Recorders\SlowJobs::class => [
    // ...
    'threshold' => [
        '#^App\\Jobs\\GenerateYearlyReports$#' => 5000,
        'default' => env('PULSE_SLOW_JOBS_THRESHOLD', 1000),
    ],
],
```

如果沒有任何正規表達式模式匹配該任務的類別名稱，那麼將會使用 `'default'` 值。

<a name="slow-outgoing-requests-recorder"></a>
#### 慢向外請求

`SlowOutgoingRequests` 記錄器會擷取關於使用 Laravel 的 [HTTP 用戶端](/docs/{{version}}/http-client) 發出的向外 HTTP 請求的資訊，這些請求超過了設定的閾值，以顯示在 [慢向外請求](#slow-outgoing-requests-card) 卡片上。

您可選擇調整慢向外請求閾值、[抽樣率](#sampling) 和忽略的 URL 模式。

您可能有一些預計會比其他請求花費更長時間的向外請求。在這些情況下，您可以設定單一請求的閾值：

```php
Recorders\SlowOutgoingRequests::class => [
    // ...
    'threshold' => [
        '#backup.zip$#' => 5000,
        'default' => env('PULSE_SLOW_OUTGOING_REQUESTS_THRESHOLD', 1000),
    ],
],
```

如果沒有任何正規表達式模式匹配該請求的 URL，那麼將會使用 `'default'` 值。

您也可以設定 URL 分組，以便將相似的 URL 分組為單一記錄。例如，您可能希望從 URL 路徑中移除唯一的 ID，或僅按網域分組。分組是使用正規表達式來設定，以「尋找並取代」URL 中的部分。設定檔中包含一些範例：

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

第一個匹配的模式將會被使用。如果沒有任何模式匹配，那麼該 URL 將會按原樣擷取。

<a name="slow-queries-recorder"></a>
#### 慢查詢

`SlowQueries` 記錄器會擷取您應用程式中任何超過設定閾值的資料庫查詢，以顯示在 [慢查詢](#slow-queries-card) 卡片上。

您可選擇調整慢查詢閾值、[抽樣率](#sampling) 和忽略的查詢模式。您也可以設定是否要擷取查詢位置。擷取的位置將會顯示在 Pulse 儀表板上，這有助於追蹤查詢來源；然而，如果相同的查詢在多個位置執行，那麼它將會針對每個唯一位置顯示多次。

您可能有一些預計會比其他查詢花費更長時間的查詢。在這些情況下，您可以設定單一查詢的閾值：

```php
Recorders\SlowQueries::class => [
    // ...
    'threshold' => [
        '#^insert into `yearly_reports`#' => 5000,
        'default' => env('PULSE_SLOW_QUERIES_THRESHOLD', 1000),
    ],
],
```

如果沒有任何正規表達式模式匹配該查詢的 SQL，那麼將會使用 `'default'` 值。

<a name="slow-requests-recorder"></a>
#### 慢請求

`Requests` 記錄器會擷取關於向您的應用程式發出的請求的資訊，以顯示在 [慢請求](#slow-requests-card) 和 [應用程式使用](#application-usage-card) 卡片上。

您可選擇調整慢路由閾值、[抽樣率](#sampling) 和忽略的路徑。

您可能有一些預計會比其他請求花費更長時間的請求。在這些情況下，您可以設定單一請求的閾值：

```php
Recorders\SlowRequests::class => [
    // ...
    'threshold' => [
        '#^/admin/#' => 5000,
        'default' => env('PULSE_SLOW_REQUESTS_THRESHOLD', 1000),
    ],
],
```

如果沒有任何正規表達式模式匹配該請求的 URL，那麼將會使用 `'default'` 值。

<a name="servers-recorder"></a>
#### 伺服器

`Servers` 記錄器會擷取驅動您應用程式的伺服器的 CPU、記憶體和儲存空間使用率，以顯示在 [伺服器](#servers-card) 卡片上。此記錄器需要在您希望監控的每台伺服器上執行 [pulse:check 指令](#capturing-entries)。

每個報告伺服器必須有一個唯一的名稱。預設情況下，Pulse 將使用 PHP 的 `gethostname` 函式所回傳的值。如果您希望自訂此項，可以設定 `PULSE_SERVER_NAME` 環境變數：

```env
PULSE_SERVER_NAME=load-balancer
```

Pulse 設定檔也允許您自訂要監控的目錄。

<a name="user-jobs-recorder"></a>
#### 使用者任務

`UserJobs` 記錄器會擷取關於在您的應用程式中派送任務的使用者的資訊，以顯示在 [應用程式使用](#application-usage-card) 卡片上。

您可選擇調整 [抽樣率](#sampling) 和忽略的任務模式。

<a name="user-requests-recorder"></a>
#### 使用者請求

`UserRequests` 記錄器會擷取關於向您的應用程式發出請求的使用者的資訊，以顯示在 [應用程式使用](#application-usage-card) 卡片上。

您可選擇調整 [抽樣率](#sampling) 和忽略的 URL 模式。

<a name="filtering"></a>
### 過濾

如我們所見，許多 [記錄器](#recorders) 透過配置，提供根據其值（例如請求的 URL）來「忽略」傳入記錄的能力。然而，有時根據其他因素，例如當前已驗證使用者，來過濾掉記錄可能會很有用。為了過濾掉這些記錄，您可以將一個閉包傳遞給 Pulse 的 `filter` 方法。通常，`filter` 方法應該在您的應用程式的 `AppServiceProvider` 中的 `boot` 方法內被調用：

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

Pulse 旨在無縫融入現有應用程式，無需任何額外的基礎設施。然而，對於高流量應用程式而言，有幾種方法可以消除 Pulse 可能對應用程式效能造成的任何影響。

<a name="using-a-different-database"></a>
### 使用不同資料庫

對於高流量應用程式，您可能偏好為 Pulse 使用專用資料庫連線，以避免影響您的應用程式資料庫。

您可以透過設定 `PULSE_DB_CONNECTION` 環境變數，自訂 Pulse 使用的[資料庫連線](/docs/{{version}}/database#configuration)。

```env
PULSE_DB_CONNECTION=pulse
```

<a name="ingest"></a>
### Redis 攝取

> [!WARNING]
> Redis 攝取功能需要 Redis 6.2 或更高版本，並且應用程式已將 `phpredis` 或 `predis` 設定為 Redis 用戶端驅動器。

預設情況下，Pulse 會在 HTTP 回應發送給客戶端或工作處理完畢後，將記錄直接儲存到[已設定的資料庫連線](#using-a-different-database)；然而，您可以改為使用 Pulse 的 Redis 攝取驅動器將記錄發送到 Redis 串流。這可以透過設定 `PULSE_INGEST_DRIVER` 環境變數來啟用：

```ini
PULSE_INGEST_DRIVER=redis
```

Pulse 預設將使用您的預設 [Redis 連線](/docs/{{version}}/redis#configuration)，但您可以透過 `PULSE_REDIS_CONNECTION` 環境變數來自訂此項：

```ini
PULSE_REDIS_CONNECTION=pulse
```

> [!WARNING]
> 當使用 Redis 攝取驅動器時，您的 Pulse 安裝（若適用）應始終使用與 Redis 驅動的佇列不同的 Redis 連線。

使用 Redis 攝取時，您需要執行 `pulse:work` 指令來監控串流，並將記錄從 Redis 移至 Pulse 的資料庫資料表。

```php
php artisan pulse:work
```

> [!NOTE]
> 為使 `pulse:work` 處理程序永久在背景執行，您應該使用程序監控器 (例如 Supervisor) 以確保 Pulse 工作者不會停止執行。

由於 `pulse:work` 指令是一個常駐處理程序，它在不重新啟動的情況下將無法察覺程式碼庫的變更。您應該在應用程式的部署流程中，透過呼叫 `pulse:restart` 指令來優雅地重新啟動此指令：

```shell
php artisan pulse:restart
```

> [!NOTE]
> Pulse 使用 [快取](/docs/{{version}}/cache) 來儲存重新啟動訊號，因此在使用此功能之前，您應該驗證您的應用程式是否已正確設定快取驅動器。

<a name="sampling"></a>
### 抽樣

預設情況下，Pulse 將擷取應用程式中發生的每個相關事件。對於高流量應用程式，這可能會導致需要在儀表板中聚合數百萬條資料庫記錄，特別是對於較長的時間週期。

您可以選擇在某些 Pulse 資料記錄器上啟用「抽樣」。例如，在 [User Requests](#user-requests-recorder) 記錄器上將抽樣率設定為 `0.1`，將意味著您僅記錄應用程式約 10% 的請求。在儀表板中，這些值將會被放大並帶有 `~` 前綴，以表示它們是近似值。

通常，特定指標的記錄越多，您就可以越安全地降低抽樣率，而不會犧牲太多準確性。

<a name="trimming"></a>
### 修剪

Pulse 將會自動修剪其儲存的記錄，一旦它們超出儀表板視窗。修剪在攝取資料時發生，採用抽籤系統，該系統可以在 Pulse [設定檔](#configuration) 中自訂。

<a name="pulse-exceptions"></a>
### 處理 Pulse 異常

如果擷取 Pulse 資料時發生異常，例如無法連線到儲存資料庫，Pulse 將會靜默失敗以避免影響您的應用程式。

如果您希望自訂這些異常的處理方式，您可以為 `handleExceptionsUsing` 方法提供一個閉包：

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

Pulse 允許您建置自訂卡片，以顯示與您應用程式特定需求相關的資料。Pulse 使用 [Livewire](https://livewire.laravel.com)，因此您可能希望在建置您的第一個自訂卡片之前，[查閱其文件](https://livewire.laravel.com/docs)。

<a name="custom-card-components"></a>
### 卡片元件

在 Laravel Pulse 中建立自訂卡片，首先要繼承基礎 `Card` Livewire 元件並定義對應的視圖：

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

當使用 Livewire 的 [延遲載入](https://livewire.laravel.com/docs/lazy) 功能時，`Card` 元件將自動提供一個佔位符，該佔位符會尊重傳遞給您元件的 `cols` 和 `rows` 屬性。

編寫您的 Pulse 卡片對應的視圖時，您可以利用 Pulse 的 Blade 元件以獲得一致的外觀和風格：

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

`$cols`、`$rows`、`$class` 和 `$expand` 變數應該傳遞給各自的 Blade 元件，這樣卡片佈局就可以從儀表板視圖中自訂。您也可以在您的視圖中包含 `wire:poll.5s=""` 屬性，以使卡片自動更新。

一旦您定義了您的 Livewire 元件和範本，該卡片就可以包含在您的 [儀表板視圖](#dashboard-customization) 中：

```blade
<x-pulse>
    ...

    <livewire:pulse.top-sellers cols="4" />
</x-pulse>
```

> [!NOTE]
> 如果您的卡片包含在套件中，您將需要使用 `Livewire::component` 方法向 Livewire 註冊該元件。

<a name="custom-card-styling"></a>
### 樣式

如果您的卡片需要額外的樣式，超出 Pulse 所包含的類別和元件，那麼有幾種選項可以為您的卡片包含自訂 CSS。

<a name="custom-card-styling-vite"></a>
#### Laravel Vite 整合

如果您的自訂卡片位於您應用程式的程式碼庫中，並且您正在使用 Laravel 的 [Vite 整合](/docs/{{version}}/vite)，您可以更新您的 `vite.config.js` 檔案，以包含一個專用於您卡片的 CSS 進入點：

```js
laravel({
    input: [
        'resources/css/pulse/top-sellers.css',
        // ...
    ],
}),
```

然後，您可以使用 `@vite` Blade 指令在您的 [儀表板視圖](#dashboard-customization) 中，並指定您卡片的 CSS 進入點：

```blade
<x-pulse>
    @vite('resources/css/pulse/top-sellers.css')

    ...
</x-pulse>
```

<a name="custom-card-styling-css"></a>
#### CSS 檔案

對於其他使用情境，包括包含在套件中的 Pulse 卡片，您可以指示 Pulse 載入額外的樣式表，方法是在您的 Livewire 元件上定義一個 `css` 方法，該方法會回傳您的 CSS 檔案路徑：

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

當此卡片包含在儀表板上時，Pulse 將自動把此檔案的內容包含在 `<style>` 標籤中，因此它不需要發佈到 `public` 目錄。

<a name="custom-card-styling-tailwind"></a>
#### Tailwind CSS

當使用 Tailwind CSS 時，您應該建立一個專用的 Tailwind 設定檔，以避免載入不必要的 CSS 或與 Pulse 的 Tailwind 類別產生衝突：

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

然後，您可以在您的 CSS 進入點中指定該設定檔：

```css
@config "../../tailwind.top-sellers.config.js";
@tailwind base;
@tailwind components;
@tailwind utilities;
```

您還需要將 `id` 或 `class` 屬性包含在您的卡片視圖中，該屬性要與傳遞給 Tailwind [重要選擇器策略](https://tailwindcss.com/docs/configuration#selector-strategy) 的選擇器相符：

```blade
<x-pulse::card id="top-sellers" :cols="$cols" :rows="$rows" class="$class">
    ...
</x-pulse::card>
```

<a name="custom-card-data"></a>
### 資料擷取與聚合

自訂卡片可以從任何地方獲取並顯示資料；然而，您可能希望利用 Pulse 強大而高效的資料記錄與聚合系統。

<a name="custom-card-data-capture"></a>
#### 擷取記錄

Pulse 允許您使用 `Pulse::record` 方法記錄「記錄 (entries)」：

```php
use Laravel\Pulse\Facades\Pulse;

Pulse::record('user_sale', $user->id, $sale->amount)
    ->sum()
    ->count();
```

提供給 `record` 方法的第一個引數是您正在記錄的 `type`，而第二個引數是決定聚合資料如何分組的 `key`。對於大多數聚合方法，您還需要指定一個要聚合的 `value`。在上面的範例中，正在聚合的值是 `$sale->amount`。然後您可以調用一個或多個聚合方法 (例如 `sum`)，以便 Pulse 可以將預聚合的值擷取到「資料桶 (buckets)」中，以便日後高效地檢索。

可用的聚合方法包括：

*   `avg`
*   `count`
*   `max`
*   `min`
*   `sum`

> [!NOTE]
> 建立擷取目前已驗證使用者 ID 的卡片套件時，您應該使用 `Pulse::resolveAuthenticatedUserId()` 方法，該方法會遵循應用程式對 [使用者解析器所做的自訂](#dashboard-resolving-users)。

<a name="custom-card-data-retrieval"></a>
#### 擷取聚合資料

當擴展 Pulse 的 `Card` Livewire 元件時，您可以使用 `aggregate` 方法來檢索儀表板中正在查看期間的聚合資料：

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

`aggregate` 方法回傳 PHP `stdClass` 物件的集合。每個物件都將包含先前擷取的 `key` 屬性，以及每個請求聚合的鍵：

```blade
@foreach ($topSellers as $seller)
    {{ $seller->key }}
    {{ $seller->sum }}
    {{ $seller->count }}
@endforeach
```

Pulse 將主要從預聚合的資料桶中檢索資料；因此，指定的聚合必須已使用 `Pulse::record` 方法預先擷取。最舊的資料桶通常會部分超出該期間，因此 Pulse 將聚合最舊的記錄以填補空缺，並為整個期間提供準確的值，而無需在每次輪詢請求時聚合整個期間。

您也可以使用 `aggregateTotal` 方法檢索給定類型的總值。例如，以下方法將檢索所有使用者銷售的總數，而不是按使用者分組。

```php
$total = $this->aggregateTotal('user_sale', 'sum');
```

<a name="custom-card-displaying-users"></a>
#### 顯示使用者

當處理將使用者 ID 記錄為鍵的聚合時，您可以使用 `Pulse::resolveUsers` 方法將這些鍵解析為使用者記錄：

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

`find` 方法回傳一個包含 `name`、`extra` 和 `avatar` 鍵的物件，您可以選擇將其直接傳遞給 `<x-pulse::user-card>` Blade 元件：

```blade
<x-pulse::user-card :user="{{ $seller->user }}" :stats="{{ $seller->sum }}" />
```

<a name="custom-recorders"></a>
#### 自訂記錄器

套件開發者可能希望提供記錄器類別，以允許使用者設定資料的擷取。

記錄器在應用程式的 `config/pulse.php` 設定檔的 `recorders` 區塊中註冊：

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

記錄器可以透過指定 `$listen` 屬性來監聽事件。Pulse 將自動註冊監聽器並呼叫記錄器的 `record` 方法：

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