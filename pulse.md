# Laravel Pulse

- [簡介](#introduction)
- [安裝](#installation)
    - [設定](#configuration)
- [控制台](#dashboard)
    - [授權](#dashboard-authorization)
    - [客製化](#dashboard-customization)
    - [解析使用者](#dashboard-resolving-users)
    - [卡片](#dashboard-cards)
- [擷取項目](#capturing-entries)
    - [記錄器](#recorders)
    - [篩選](#filtering)
- [效能](#performance)
    - [使用不同的資料庫](#using-a-different-database)
    - [Redis 數據引入](#ingest)
    - [採樣](#sampling)
    - [修剪](#trimming)
    - [處理 Pulse 例外](#pulse-exceptions)
- [自訂卡片](#custom-cards)
    - [卡片元件](#custom-card-components)
    - [樣式](#custom-card-styling)
    - [資料擷取與聚合](#custom-card-data)

<a name="introduction"></a>
## 簡介

[Laravel Pulse](https://github.com/laravel/pulse) 提供應用程式效能與使用情況的即時洞察。透過 Pulse，您可以追查諸如慢速任務和端點等瓶頸、找出最活躍的使用者等等。

若要深入偵錯個別事件，請查閱 [Laravel Telescope](/docs/{{version}}/telescope)。


<a name="installation"></a>
## 安裝

> [!WARNING]
> Pulse 的第一方儲存實作目前需要 MySQL、MariaDB 或 PostgreSQL 資料庫。如果您使用不同的資料庫引擎，則需要為您的 Pulse 資料單獨建立一個 MySQL、MariaDB 或 PostgreSQL 資料庫。

您可以使用 Composer 套件管理器來安裝 Pulse：

```shell
composer require laravel/pulse
```

接著，您應該使用 `vendor:publish` Artisan 命令發佈 Pulse 的設定檔和資料庫遷移檔：

```shell
php artisan vendor:publish --provider="Laravel\Pulse\PulseServiceProvider"
```

最後，您應該執行 `migrate` 命令，以便建立儲存 Pulse 資料所需的資料表：

```shell
php artisan migrate
```

一旦 Pulse 的資料庫遷移檔執行完畢，您就可以透過 `/pulse` 路由存取 Pulse 控制台。

> [!NOTE]
> 如果您不想將 Pulse 資料儲存在應用程式的主資料庫中，您可以 [指定專用的資料庫連線](#using-a-different-database)。


<a name="configuration"></a>
### 設定

Pulse 的許多設定選項都可以透過環境變數進行控制。若要查看可用選項、註冊新的記錄器或設定進階選項，您可以發佈 `config/pulse.php` 設定檔：

```shell
php artisan vendor:publish --tag=pulse-config
```

<a name="dashboard"></a>
## 控制台


<a name="dashboard-authorization"></a>
### 授權

Pulse 控制台可透過 `/pulse` 路由存取。預設情況下，您只能在 `local` 環境中存取此控制台，因此您需要透過客製化 `'viewPulse'` 授權 gate 來為您的生產環境設定授權。您可以在應用程式的 `app/Providers/AppServiceProvider.php` 檔案中完成此操作：

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
### 客製化

Pulse 控制台卡片和佈局可以透過發佈控制台視圖進行設定。控制台視圖將會發佈到 `resources/views/vendor/pulse/dashboard.blade.php`：

```shell
php artisan vendor:publish --tag=pulse-dashboard
```

控制台由 [Livewire](https://livewire.laravel.com/) 提供支援，它允許您客製化卡片和佈局，而無需重新建置任何 JavaScript 資源。

在此檔案中，`<x-pulse>` 元件負責渲染控制台並為卡片提供網格佈局。如果您希望控制台跨越整個螢幕寬度，您可以為元件提供 `full-width` 屬性：

```blade
<x-pulse full-width>
    ...
</x-pulse>
```

預設情況下，`<x-pulse>` 元件將建立一個 12 欄網格，但您可以使用 `cols` 屬性進行客製化：

```blade
<x-pulse cols="16">
    ...
</x-pulse>
```

每張卡片都接受 `cols` 和 `rows` 屬性來控制空間和位置：

```blade
<livewire:pulse.usage cols="4" rows="2" />
```

大多數卡片也接受 `expand` 屬性來顯示完整卡片而不是滾動：

```blade
<livewire:pulse.slow-queries expand />
```


<a name="dashboard-resolving-users"></a>
### 解析使用者

對於顯示使用者資訊的卡片，例如應用程式使用卡片，Pulse 只會記錄使用者的 ID。在渲染控制台時，Pulse 將從您預設的 `Authenticatable` 模型中解析 `name` 和 `email` 欄位，並使用 Gravatar 網路服務顯示頭像。

您可以透過在應用程式的 `App\Providers\AppServiceProvider` 類別中呼叫 `Pulse::user` 方法來客製化欄位和頭像。

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
> 您可以透過實作 `Laravel\Pulse\Contracts\ResolvesUsers` 契約並將其綁定到 Laravel 的 [服務容器](/docs/{{version}}/container#binding-a-singleton) 中，來完全客製化驗證使用者的擷取和檢索方式。


<a name="dashboard-cards"></a>
### 卡片


<a name="servers-card"></a>
#### 伺服器

`<livewire:pulse.servers />` 卡片顯示所有執行 `pulse:check` 命令的伺服器系統資源使用情況。有關系統資源報告的更多資訊，請參閱 [伺服器記錄器](#servers-recorder) 的文件。

如果您更換基礎設施中的伺服器，您可能希望在指定持續時間後停止在 Pulse 控制台顯示非活動伺服器。您可以透過使用 `ignore-after` 屬性來實現，該屬性接受非活動伺服器應從 Pulse 控制台移除的秒數。或者，您可以提供一個相對時間格式的字串，例如 `1 hour` 或 `3 days and 1 hour`：

```blade
<livewire:pulse.servers ignore-after="3 hours" />
```


<a name="application-usage-card"></a>
#### 應用程式使用情況

`<livewire:pulse.usage />` 卡片顯示向您的應用程式發出請求、分派 jobs 以及遇到慢請求的前 10 名使用者。

如果您希望同時在螢幕上查看所有使用情況指標，您可以多次包含此卡片並指定 `type` 屬性：

```blade
<livewire:pulse.usage type="requests" />
<livewire:pulse.usage type="slow_requests" />
<livewire:pulse.usage type="jobs" />
```

要了解如何客製化 Pulse 檢索和顯示使用者資訊，請查閱我們關於 [解析使用者](#dashboard-resolving-users) 的文件。

> [!NOTE]
> 如果您的應用程式接收大量請求或分派大量 jobs，您可能希望啟用 [採樣](#sampling)。請參閱 [使用者請求記錄器](#user-requests-recorder)、[使用者 jobs 記錄器](#user-jobs-recorder) 和 [慢速 jobs 記錄器](#slow-jobs-recorder) 文件以獲取更多資訊。


<a name="exceptions-card"></a>
#### 例外

`<livewire:pulse.exceptions />` 卡片顯示應用程式中發生例外的頻率和近期情況。預設情況下，例外根據例外類別和發生位置進行分組。有關更多資訊，請參閱 [例外記錄器](#exceptions-recorder) 文件。


<a name="queues-card"></a>
#### 佇列

`<livewire:pulse.queues />` 卡片顯示應用程式中佇列的吞吐量，包括已排入佇列、正在處理、已處理、已釋放和失敗的 jobs 數量。有關更多資訊，請參閱 [佇列記錄器](#queues-recorder) 文件。


<a name="slow-requests-card"></a>
#### 慢請求

`<livewire:pulse.slow-requests />` 卡片顯示超過設定閾值 (預設為 1,000 毫秒) 的應用程式傳入請求。有關更多資訊，請參閱 [慢請求記錄器](#slow-requests-recorder) 文件。


<a name="slow-jobs-card"></a>
#### 慢速 Jobs

`<livewire:pulse.slow-jobs />` 卡片顯示應用程式中超過設定閾值 (預設為 1,000 毫秒) 的已排入佇列 jobs。有關更多資訊，請參閱 [慢速 Jobs 記錄器](#slow-jobs-recorder) 文件。


<a name="slow-queries-card"></a>
#### 慢查詢

`<livewire:pulse.slow-queries />` 卡片顯示應用程式中超過設定閾值 (預設為 1,000 毫秒) 的資料庫查詢。

預設情況下，慢查詢會根據 SQL 查詢 (不含綁定) 和發生位置進行分組，但如果您只希望根據 SQL 查詢進行分組，則可以選擇不擷取位置。

如果您因為極大的 SQL 查詢接收語法高亮而遇到渲染效能問題，您可以透過新增 `without-highlighting` 屬性來禁用高亮顯示：

```blade
<livewire:pulse.slow-queries without-highlighting />
```

有關更多資訊，請參閱 [慢查詢記錄器](#slow-queries-recorder) 文件。


<a name="slow-outgoing-requests-card"></a>
#### 慢速向外請求

`<livewire:pulse.slow-outgoing-requests />` 卡片顯示使用 Laravel 的 [HTTP client](/docs/{{version}}/http-client) 發出並超過設定閾值 (預設為 1,000 毫秒) 的向外請求。

預設情況下，項目將按完整 URL 分組。但是，您可能希望使用正規表達式來標準化或分組類似的向外請求。有關更多資訊，請參閱 [慢速向外請求記錄器](#slow-outgoing-requests-recorder) 文件。


<a name="cache-card"></a>
#### 快取

`<livewire:pulse.cache />` 卡片顯示應用程式的快取命中和未命中統計資訊，包括全域和個別鍵。

預設情況下，項目將按鍵分組。但是，您可能希望使用正規表達式來標準化或分組類似的鍵。有關更多資訊，請參閱 [快取互動記錄器](#cache-interactions-recorder) 文件。

<a name="capturing-entries"></a>
## 擷取項目

大多數 Pulse 記錄器會根據 Laravel 派送的框架事件自動擷取項目。然而，[伺服器記錄器](#servers-recorder)和某些第三方卡片必須定期輪詢資訊。若要使用這些卡片，您必須在所有個別應用程式伺服器上執行 `pulse:check` 守護行程：

```php
php artisan pulse:check
```

> [!NOTE]
> 為了讓 `pulse:check` 程序在背景永久執行，您應該使用程序監控器，例如 Supervisor，以確保該指令不會停止運行。

由於 `pulse:check` 指令是一個長期運行的程序，若不重新啟動，它將不會看到程式碼庫的變更。您應該在應用程式的部署過程中，透過呼叫 `pulse:restart` 指令來優雅地重新啟動指令：

```shell
php artisan pulse:restart
```

> [!NOTE]
> Pulse 使用 [快取](/docs/{{version}}/cache) 來儲存重新啟動訊號，因此您在使用此功能之前，應該驗證您的應用程式是否已正確設定快取驅動程式。

<a name="recorders"></a>
### 記錄器

記錄器 (Recorders) 負責從您的應用程式中擷取項目並記錄到 Pulse 資料庫中。記錄器在 [Pulse 設定檔](#configuration)的 `recorders` 區塊中註冊和設定。

<a name="cache-interactions-recorder"></a>
#### 快取互動

`CacheInteractions` 記錄器擷取應用程式中發生的 [快取](/docs/{{version}}/cache) 命中與未命中資訊，以便在 [快取](#cache-card) 卡片上顯示。

您可以選擇性地調整 [取樣率](#sampling) 和忽略的鍵模式。

您也可以設定鍵分組，將相似的鍵歸為單一項目。例如，您可能希望從快取相同類型資訊的鍵中移除唯一的 ID。分組是使用正規表達式配置的，用於「尋找並替換」鍵的一部分。設定檔中包含一個範例：

```php
Recorders\CacheInteractions::class => [
    // ...
    'groups' => [
        // '/:\d+/' => ':*',
    ],
],
```

第一個匹配的模式將被使用。如果沒有模式匹配，則鍵將原樣擷取。

<a name="exceptions-recorder"></a>
#### 例外

`Exceptions` 記錄器擷取應用程式中發生的可回報例外資訊，以便在 [例外](#exceptions-card) 卡片上顯示。

您可以選擇性地調整 [取樣率](#sampling) 和忽略的例外模式。您也可以設定是否擷取例外發生的位置。擷取到的位置將顯示在 Pulse 控制台 (dashboard) 上，這有助於追查例外來源；然而，如果同一個例外發生在多個位置，那麼它將針對每個唯一位置多次出現。

<a name="queues-recorder"></a>
#### 佇列

`Queues` 記錄器擷取應用程式佇列的資訊，以便在 [佇列](#queues-card) 卡片上顯示。

您可以選擇性地調整 [取樣率](#sampling) 和忽略的任務模式。

<a name="slow-jobs-recorder"></a>
#### 慢速任務

`SlowJobs` 記錄器擷取應用程式中發生的慢速任務資訊，以便在 [慢速任務](#slow-jobs-recorder) 卡片上顯示。

您可以選擇性地調整慢速任務閾值、[取樣率](#sampling) 和忽略的任務模式。

您可能有一些預期會花費更長時間的任務。在這些情況下，您可以配置每個任務的閾值：

```php
Recorders\SlowJobs::class => [
    // ...
    'threshold' => [
        '#^App\\Jobs\\GenerateYearlyReports$#' => 5000,
        'default' => env('PULSE_SLOW_JOBS_THRESHOLD', 1000),
    ],
],
```

如果沒有正規表達式模式匹配任務的類別名稱，則將使用 `'default'` 值。

<a name="slow-outgoing-requests-recorder"></a>
#### 慢速對外請求

`SlowOutgoingRequests` 記錄器擷取使用 Laravel 的 [HTTP Client](/docs/{{version}}/http-client) 發出的對外 HTTP 請求資訊，這些請求超過配置的閾值，以便在 [慢速對外請求](#slow-outgoing-requests-card) 卡片上顯示。

您可以選擇性地調整慢速對外請求閾值、[取樣率](#sampling) 和忽略的 URL 模式。

您可能有一些預期會花費更長時間的對外請求。在這些情況下，您可以配置每個請求的閾值：

```php
Recorders\SlowOutgoingRequests::class => [
    // ...
    'threshold' => [
        '#backup.zip$#' => 5000,
        'default' => env('PULSE_SLOW_OUTGOING_REQUESTS_THRESHOLD', 1000),
    ],
],
```

如果沒有正規表達式模式匹配請求的 URL，則將使用 `'default'` 值。

您也可以設定 URL 分組，將相似的 URL 歸為單一項目。例如，您可能希望從 URL 路徑中移除唯一的 ID，或僅按網域分組。分組是使用正規表達式配置的，用於「尋找並替換」URL 的一部分。設定檔中包含一些範例：

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

第一個匹配的模式將被使用。如果沒有模式匹配，則 URL 將原樣擷取。

<a name="slow-queries-recorder"></a>
#### 慢速查詢

`SlowQueries` 記錄器擷取應用程式中任何超過配置閾值的資料庫查詢資訊，以便在 [慢速查詢](#slow-queries-card) 卡片上顯示。

您可以選擇性地調整慢速查詢閾值、[取樣率](#sampling) 和忽略的查詢模式。您也可以設定是否擷取查詢位置。擷取到的位置將顯示在 Pulse 控制台 (dashboard) 上，這有助於追查查詢來源；然而，如果同一個查詢在多個位置執行，那麼它將針對每個唯一位置多次出現。

您可能有一些預期會花費更長時間的查詢。在這些情況下，您可以配置每個查詢的閾值：

```php
Recorders\SlowQueries::class => [
    // ...
    'threshold' => [
        '#^insert into `yearly_reports`#' => 5000,
        'default' => env('PULSE_SLOW_QUERIES_THRESHOLD', 1000),
    ],
],
```

如果沒有正規表達式模式匹配查詢的 SQL，則將使用 `'default'` 值。

<a name="slow-requests-recorder"></a>
#### 慢速請求

`Requests` 記錄器擷取應用程式收到的請求資訊，以便在 [慢速請求](#slow-requests-card) 和 [應用程式用量](#application-usage-card) 卡片上顯示。

您可以選擇性地調整慢速路由閾值、[取樣率](#sampling) 和忽略的路徑。

您可能有一些預期會花費更長時間的請求。在這些情況下，您可以配置每個請求的閾值：

```php
Recorders\SlowRequests::class => [
    // ...
    'threshold' => [
        '#^/admin/#' => 5000,
        'default' => env('PULSE_SLOW_REQUESTS_THRESHOLD', 1000),
    ],
],
```

如果沒有正規表達式模式匹配請求的 URL，則將使用 `'default'` 值。

<a name="servers-recorder"></a>
#### 伺服器

`Servers` 記錄器擷取驅動您應用程式的伺服器的 CPU、記憶體和儲存用量資訊，以便在 [伺服器](#servers-card) 卡片上顯示。此記錄器要求在您希望監控的每台伺服器上執行 [pulse:check 命令](#capturing-entries)。

每個回報伺服器必須有一個唯一的名稱。預設情況下，Pulse 將使用 PHP 的 `gethostname` 函數返回的值。如果您希望自訂此設定，您可以設定 `PULSE_SERVER_NAME` 環境變數：

```env
PULSE_SERVER_NAME=load-balancer
```

Pulse 設定檔也允許您自訂受監控的目錄。

<a name="user-jobs-recorder"></a>
#### 使用者任務

`UserJobs` 記錄器擷取應用程式中分派任務的使用者資訊，以便在 [應用程式用量](#application-usage-card) 卡片上顯示。

您可以選擇性地調整 [取樣率](#sampling) 和忽略的任務模式。

<a name="user-requests-recorder"></a>
#### 使用者請求

`UserRequests` 記錄器擷取向應用程式發出請求的使用者資訊，以便在 [應用程式用量](#application-usage-card) 卡片上顯示。

您可以選擇性地調整 [取樣率](#sampling) 和忽略的 URL 模式。

<a name="filtering"></a>
### 篩選

如前所述，許多 [記錄器](#recorders) 透過設定提供了「忽略」根據其值（例如請求的 URL）傳入項目功能。然而，有時候根據其他因素（例如當前已驗證的使用者）來篩選記錄會很有用。為了篩選掉這些記錄，您可以傳入一個閉包到 Pulse 的 `filter` 方法。通常，`filter` 方法應該在您應用程式的 `AppServiceProvider` 的 `boot` 方法中被呼叫：

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

Pulse 旨在無需額外基礎設施即可整合到現有應用程式中。然而，對於高流量應用程式，有多種方法可以消除 Pulse 對應用程式效能的任何影響。

<a name="using-a-different-database"></a>
### 使用不同的資料庫

對於高流量應用程式，您可能偏好為 Pulse 使用專用的資料庫連線，以避免影響您的應用程式資料庫。

您可以透過設定 `PULSE_DB_CONNECTION` 環境變數來客製化 Pulse 使用的[資料庫連線](/docs/{{version}}/database#configuration)。

```env
PULSE_DB_CONNECTION=pulse
```

<a name="ingest"></a>
### Redis 數據引入

> [!WARNING]
> Redis 數據引入需要 Redis 6.2 或更高版本，並且應用程式的 Redis 用戶端驅動器為 `phpredis` 或 `predis`。

預設情況下，Pulse 會在 HTTP 回應傳送給用戶端或任務已處理後，直接將項目儲存到[設定的資料庫連線](#using-a-different-database)中；然而，您可以使用 Pulse 的 Redis 數據引入驅動器將項目傳送到 Redis 串流。這可以透過設定 `PULSE_INGEST_DRIVER` 環境變數來啟用：

```ini
PULSE_INGEST_DRIVER=redis
```

Pulse 預設會使用您的預設 [Redis 連線](/docs/{{version}}/redis#configuration)，但您可以透過 `PULSE_REDIS_CONNECTION` 環境變數來客製化此設定：

```ini
PULSE_REDIS_CONNECTION=pulse
```

> [!WARNING]
> 使用 Redis 數據引入驅動器時，您的 Pulse 安裝應始終使用與您由 Redis 驅動的佇列不同的 Redis 連線（如果適用）。

使用 Redis 數據引入時，您需要執行 `pulse:work` 命令來監控串流，並將項目從 Redis 移至 Pulse 的資料庫表格中。

```php
php artisan pulse:work
```

> [!NOTE]
> 為了讓 `pulse:work` 程序在背景永久運行，您應該使用諸如 Supervisor 之類的程序監控器，以確保 Pulse 工作程式不會停止運行。

由於 `pulse:work` 命令是一個長時間執行的程序，它不會看到程式碼庫的變更而無需重新啟動。您應該在應用程式的部署過程中使用 `pulse:restart` 命令來優雅地重新啟動此命令：

```shell
php artisan pulse:restart
```

> [!NOTE]
> Pulse 使用[快取](/docs/{{version}}/cache)來儲存重新啟動信號，因此在使用此功能之前，您應該驗證應用程式是否正確設定了快取驅動器。

<a name="sampling"></a>
### 採樣

預設情況下，Pulse 會擷取應用程式中發生的每個相關事件。對於高流量應用程式，這可能會導致在控制台中聚合數百萬條資料庫列，尤其是在較長的時間段內。

您可以選擇在特定的 Pulse 資料記錄器上啟用「採樣」。例如，將 [User Requests 記錄器](#user-requests-recorder)的採樣率設定為 `0.1`，表示您僅記錄約 10% 的應用程式請求。在控制台中，這些值將按比例放大，並以 `~` 作為前綴，表示它們是近似值。

通常，特定指標的項目越多，您可以安全地將採樣率設定得越低，而不會犧牲太多準確性。

<a name="trimming"></a>
### 修剪

Pulse 會在其儲存的項目超出控制台窗口後自動修剪。修剪是在使用彩票系統引入資料時發生的，該系統可以在 Pulse [設定檔](#configuration)中客製化。

<a name="pulse-exceptions"></a>
### 處理 Pulse 例外

如果在擷取 Pulse 資料時發生例外狀況，例如無法連線到儲存資料庫，Pulse 將靜默失敗，以避免影響您的應用程式。

如果您希望客製化這些例外的處理方式，您可以提供一個閉包給 `handleExceptionsUsing` 方法：

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

Pulse 讓您可以建立自訂卡片，以顯示與應用程式特定需求相關的資料。Pulse 使用 [Livewire](https://livewire.laravel.com)，因此您可能需要在建立您的第一個自訂卡片之前，[查閱其文件](https://livewire.laravel.com/docs)。


<a name="custom-card-components"></a>
### 卡片元件

在 Laravel Pulse 中建立自訂卡片，首先是擴展基礎的 `Card` Livewire 元件，並定義一個對應的 View：

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

當使用 Livewire 的[延遲載入](https://livewire.laravel.com/docs/lazy)功能時，`Card` 元件會自動提供一個佔位符，該佔位符會遵循傳遞給元件的 `cols` 和 `rows` 屬性。

在撰寫 Pulse 卡片對應的 View 時，您可以利用 Pulse 的 Blade 元件以獲得一致的外觀和風格：

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

`$cols`、`$rows`、`$class` 和 `$expand` 變數應傳遞給各自的 Blade 元件，以便卡片佈局可從控制台 View 中客製化。您可能還希望在 View 中包含 `wire:poll.5s=""` 屬性，以便卡片自動更新。

一旦您定義了 Livewire 元件和模板，該卡片就可以包含在您的[控制台 View](#dashboard-customization) 中：

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

如果您的卡片除了 Pulse 內建的類別和元件之外，還需要額外樣式，那麼有幾種選項可以包含自訂 CSS。


<a name="custom-card-styling-vite"></a>
#### Laravel Vite 整合

如果您的自訂卡片位於應用程式的程式碼庫中，並且您正在使用 Laravel 的 [Vite 整合](/docs/{{version}}/vite)，您可以更新您的 `vite.config.js` 檔案，為您的卡片包含一個專用的 CSS 進入點：

```js
laravel({
    input: [
        'resources/css/pulse/top-sellers.css',
        // ...
    ],
}),
```

然後，您可以在您的[控制台 View](#dashboard-customization) 中使用 `@vite` Blade 指令，指定該卡片的 CSS 進入點：

```blade
<x-pulse>
    @vite('resources/css/pulse/top-sellers.css')

    ...
</x-pulse>
```


<a name="custom-card-styling-css"></a>
#### CSS 檔案

對於其他使用情境，包括包含在套件中的 Pulse 卡片，您可以透過在 Livewire 元件上定義一個 `css` 方法來指示 Pulse 載入額外的樣式表，該方法會返回 CSS 檔案的路徑：

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

當此卡片包含在控制台中時，Pulse 會自動將此檔案的內容包含在 `<style>` 標籤中，因此無需將其發佈到 `public` 目錄。


<a name="custom-card-styling-tailwind"></a>
#### Tailwind CSS

使用 Tailwind CSS 時，您應該建立一個專用的 Tailwind 設定檔，以避免載入不必要的 CSS 或與 Pulse 的 Tailwind 類別衝突：

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

然後，您可以在 CSS 進入點中指定設定檔：

```css
@config "../../tailwind.top-sellers.config.js";
@tailwind base;
@tailwind components;
@tailwind utilities;
```

您還需要在卡片的 View 中包含一個 `id` 或 `class` 屬性，該屬性與傳遞給 Tailwind [重要選擇器策略](https://tailwindcss.com/docs/configuration#selector-strategy)的選擇器匹配：

```blade
<x-pulse::card id="top-sellers" :cols="$cols" :rows="$rows" class="$class">
    ...
</x-pulse::card>
```

<a name="custom-card-data"></a>
### 資料擷取與聚合

自訂卡片可以從任何地方抓取並顯示資料；然而，您可能會希望運用 Pulse 強大且高效能的資料記錄與聚合系統。

<a name="custom-card-data-capture"></a>
#### 擷取項目

Pulse 允許您使用 `Pulse::record` 方法來記錄「項目」：

```php
use Laravel\Pulse\Facades\Pulse;

Pulse::record('user_sale', $user->id, $sale->amount)
    ->sum()
    ->count();
```

提供給 `record` 方法的第一個參數是您正在記錄的項目之 `type`，而第二個參數是決定聚合資料應如何分組的 `key`。對於大多數聚合方法，您還需要指定一個要聚合的 `value`。在上述範例中，被聚合的值是 `$sale->amount`。您可以隨後呼叫一個或多個聚合方法 (例如 `sum`)，以便 Pulse 可以將預先聚合的值擷取到「儲存桶 (bucket)」中，以便後續高效能地擷取。

可用的聚合方法有：

* `avg`
* `count`
* `max`
* `min`
* `sum`

> [!NOTE]
> 當建構擷取目前已驗證使用者 ID 的卡片套件時，您應該使用 `Pulse::resolveAuthenticatedUserId()` 方法，該方法會遵守應用程式中任何[使用者解析器客製化](#dashboard-resolving-users)。

<a name="custom-card-data-retrieval"></a>
#### 擷取聚合資料

當擴展 Pulse 的 `Card` Livewire 元件時，您可以使用 `aggregate` 方法來擷取控制台中正在檢視期間的聚合資料：

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

`aggregate` 方法會回傳一個 PHP `stdClass` 物件的集合。每個物件將包含先前擷取的 `key` 屬性，以及每個請求聚合的鍵：

```blade
@foreach ($topSellers as $seller)
    {{ $seller->key }}
    {{ $seller->sum }}
    {{ $seller->count }}
@endforeach
```

Pulse 將主要從預先聚合的儲存桶中擷取資料；因此，指定的聚合必須已預先使用 `Pulse::record` 方法擷取。最舊的儲存桶通常會部分超出該期間，因此 Pulse 將聚合最舊的項目以填補空白並提供整個期間的準確值，而無需在每次輪詢請求時聚合整個期間。

您也可以使用 `aggregateTotal` 方法來擷取給定類型的總值。例如，以下方法將擷取所有使用者銷售的總和，而不是按使用者分組。

```php
$total = $this->aggregateTotal('user_sale', 'sum');
```

<a name="custom-card-displaying-users"></a>
#### 顯示使用者

當處理以使用者 ID 作為鍵的聚合資料時，您可以使用 `Pulse::resolveUsers` 方法將鍵解析為使用者記錄：

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

`find` 方法會回傳一個包含 `name`、`extra` 和 `avatar` 鍵的物件，您可以選擇直接傳遞給 `<x-pulse::user-card>` Blade 元件：

```blade
<x-pulse::user-card :user="{{ $seller->user }}" :stats="{{ $seller->sum }}" />
```

<a name="custom-recorders"></a>
#### 自訂記錄器

套件開發者可能會希望提供記錄器類別，以允許使用者設定資料擷取。

記錄器註冊在應用程式的 `config/pulse.php` 設定檔之 `recorders` 區塊中：

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