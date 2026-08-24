# 升級指南

- [從 12.x 升級至 13.0](#upgrade-13.0)
    - [使用 AI 進行升級](#upgrading-using-ai)

<a name="high-impact-changes"></a>
## 高影響變更

<div class="content-list" markdown="1">

- [更新相依套件](#updating-dependencies)
- [更新 Laravel 安裝程式](#updating-the-laravel-installer)
- [請求偽造保護](#request-forgery-protection)

</div>


<a name="medium-impact-changes"></a>
## 中影響變更

<div class="content-list" markdown="1">

- [快取 `serializable_classes` 設定](#cache-serializable_classes-configuration)
- [在使用 MySQL 或 MariaDB 時進行資料庫 `upsert`](#database-upsert-mariadb-mysql)

</div>


<a name="low-impact-changes"></a>
## 低影響變更

<div class="content-list" markdown="1">

- [快取前綴與 Session Cookie 名稱](#cache-prefixes-and-session-cookie-names)
- [集合 Model 序列化還原預先載入的關聯](#collection-model-serialization-restores-eager-loaded-relations)
- [`Container::call` 與可為 Null 的類別預設值](#containercall-and-nullable-class-defaults)
- [網域路由註冊優先順序](#domain-route-registration-precedence)
- [`JobAttempted` 事件 Exception 酬載](#jobattempted-event-exception-payload)
- [Manager `extend` 回呼綁定](#manager-extend-callback-binding)
- [包含 `JOIN`、`ORDER BY` 與 `LIMIT` 的 MySQL `DELETE` 查詢](#mysql-delete-queries-with-join-order-by-and-limit)
- [分頁 Bootstrap 視圖名稱](#pagination-bootstrap-view-names)
- [多型樞紐表名稱產生](#polymorphic-pivot-table-name-generation)
- [`QueueBusy` 事件屬性重新命名](#queuebusy-event-property-rename)
- [Session `serialization` 設定](#session-serialization-configuration)
- [測試之間重置 `Str` 工廠](#str-factories-reset-between-tests)

</div>

<a name="upgrade-13.0"></a>
## 從 12.x 升級至 13.0

#### 預估升級時間：10 分鐘

> [!NOTE]
> 我們嘗試記錄每個可能造成重大變更 (Breaking Change) 的地方。由於部分破壞性變更僅存在於框架中較少被使用的部分，因此這些變更可能只有一小部分會影響您的應用程式。為了節省時間，您可以使用 [Shift](https://laravelshift.com)。Shift 是一個由社群維護的服務，可自動化進行 Laravel 升級。

<a name="upgrading-using-ai"></a>
### 使用 AI 進行升級

您可以使用 [Laravel Boost](https://github.com/laravel/boost) 來自動化您的升級。Boost 是一個官方第一方提供的 MCP (模型上下文協議) 伺服器，能為您的 AI 助手提供引導式的升級提示詞 — 一旦安裝於任何 Laravel 12 應用程式中，即可在 Claude Code、Cursor、OpenCode、Gemini 或 VS Code 中使用 `/upgrade-laravel-v13` 斜線命令來開始升級至 Laravel 13。此命令需要 Laravel Boost `^2.0`。

<a name="updating-dependencies"></a>
### 更新依賴項目

**影響可能性：高**

您應該更新應用程式 `composer.json` 檔案中的以下依賴項目：

<div class="content-list" markdown="1">

- `laravel/framework` 至 `^13.0`
- `laravel/boost` 至 `^2.0`
- `laravel/tinker` 至 `^3.0`
- `phpunit/phpunit` 至 `^12.0`
- `pestphp/pest` 至 `^4.0`

</div>

<a name="updating-the-laravel-installer"></a>
### 更新 Laravel 安裝程式

如果您正在使用 Laravel installer CLI 工具來建立全新的 Laravel 應用程式，則應該更新您的安裝程式以相容於 Laravel 13.x。

如果您是透過 `composer global require` 安裝 Laravel installer，您可以使用 `composer global update` 來更新安裝程式：

```shell
composer global update laravel/installer
```

或者，如果您使用的是 [Laravel Herd](https://herd.laravel.com) 隨附的 Laravel installer，則應該將 Herd 更新至最新版本。

<a name="cache"></a>
### 快取

<a name="cache-prefixes-and-session-cookie-names"></a>
#### 快取前綴與 Session Cookie 名稱

**影響可能性：低**

Laravel 的預設快取與 Redis 金鑰前綴現在使用連字號 (Hyphen) 作為結尾。

在大多數應用程式中，此變更不會產生影響，因為應用程式層級的設定檔通常已經定義了這些數值。這主要影響那些在缺乏相應設定值時、依賴框架層級預備設定 (Fallback configuration) 的應用程式。

如果您的應用程式依賴這些自動產生的預設值，升級後快取金鑰與 Session Cookie 名稱可能會有所改變：

```php
// Laravel <= 12.x
Str::slug((string) env('APP_NAME', 'laravel'), '_').'_cache_';
Str::slug((string) env('APP_NAME', 'laravel'), '_').'_database_';
Str::slug((string) env('APP_NAME', 'laravel'), '_').'_session';

// Laravel >= 13.x
Str::slug((string) env('APP_NAME', 'laravel')).'-cache-';
Str::slug((string) env('APP_NAME', 'laravel')).'-database-';
Str::slug((string) env('APP_NAME', 'laravel')).'-session';
```

若要保持先前的行為，請在環境變數中明確設定 `CACHE_PREFIX`、`REDIS_PREFIX` 以及 `SESSION_COOKIE`。

<a name="store-and-repository-contracts-touch"></a>
#### `Store` 與 `Repository` 契約(Contracts)：`touch`

**影響可能性：極低**

快取契約(Contracts)現在包含一個 `touch` 方法，用於延長項目的生存時間 (TTL)。如果您維護了自訂的快取儲存區 (Store) 實作，應該新增此方法：

```php
// Illuminate\Contracts\Cache\Store
public function touch($key, $seconds);
```

<a name="cache-serializable_classes-configuration"></a>
#### 快取 `serializable_classes` 設定

**影響可能性：中**

預設的應用程式 `cache` 設定現在包含一個設定為 `false` 的 `serializable_classes` 選項。這強化了快取的反序列化行為，以協助防止在應用程式的 `APP_KEY` 外洩時發生 PHP 反序列化道具鏈 (Gadget chain) 攻擊。如果您的應用程式有刻意在快取中儲存 PHP 物件，您應該明確列出允許被反序列化的類別：

```php
'serializable_classes' => [
    App\Data\CachedDashboardStats::class,
    App\Support\CachedPricingSnapshot::class,
],
```

如果您的應用程式先前依賴反序列化任意快取物件，您需要將該用法遷移至明確的類別白名單，或者改用非物件的快取有效負載（例如陣列）。

<a name="container"></a>
### 容器

<a name="containercall-and-nullable-class-defaults"></a>
#### `Container::call` 與可為 Null 的類別預設值

**影響可能性：低**

`Container::call` 現在當沒有任何綁定存在時，會尊重可為 Null 的類別參數預設值，這與 Laravel 12 中引進的建構子注入行為一致：

```php
$container->call(function (?Carbon $date = null) {
    return $date;
});

// Laravel <= 12.x: Carbon instance
// Laravel >= 13.x: null
```

如果您的方法呼叫注入邏輯依賴先前的行為，您可能需要進行更新。

<a name="contracts"></a>
### 契約(Contracts)

<a name="dispatcher-contract-dispatchafterresponse"></a>
#### `Dispatcher` 契約(Contracts)：`dispatchAfterResponse`

**影響可能性：極低**

`Illuminate\Contracts\Bus\Dispatcher` 契約(Contracts)現在包含了 `dispatchAfterResponse($command, $handler = null)` 方法。

如果您維護自訂的分發器 (Dispatcher) 實作，請將此方法新增至您的類別中。

<a name="responsefactory-contract-eventstream"></a>
#### `ResponseFactory` 契約(Contracts)：`eventStream`

**影響可能性：極低**

`Illuminate\Contracts\Routing\ResponseFactory` 契約(Contracts)現在包含了 `eventStream` 方法簽章。

如果您維護此契約(Contracts)的自訂實作，您應該新增此方法。

<a name="mustverifyemail-contract-markemailasunverified"></a>
#### `MustVerifyEmail` 契約(Contracts)：`markEmailAsUnverified`

**影響可能性：極低**

`Illuminate\Contracts\Auth\MustVerifyEmail` 契約(Contracts)現在包含了 `markEmailAsUnverified()`。

如果您有提供此契約(Contracts)的自訂實作，請新增此方法以保持相容性。

<a name="database"></a>
### 資料庫

<a name="database-upsert-mariadb-mysql"></a>
#### 在 MySQL 或 MariaDB 使用資料庫 `upsert`

**影響可能性：中**

Laravel 現在會驗證呼叫者是否為 `uniqueBy` 提供非空數值，若為空將會拋出 `InvalidArgumentException`，而非生成無效的 SQL。

雖然 MariaDB 與 MySQL 資料庫驅動程式會忽略 `uniqueBy` 的數值，且總是使用資料表的主鍵與唯一索引來偵測已存在的紀錄，但此驗證仍然適用。如果 `uniqueBy` 為空，將會拋出 `InvalidArgumentException`。

<a name="mysql-delete-queries-with-join-order-by-and-limit"></a>
#### 包含 `JOIN`、`ORDER BY` 及 `LIMIT` 的 MySQL `DELETE` 查詢

**影響可能性：低**

Laravel 現在針對 MySQL 文法會編譯包含 `ORDER BY` 和 `LIMIT` 的完整 `DELETE ... JOIN` 查詢。

在先前的版本中，`ORDER BY` / `LIMIT` 子句在帶有 JOIN 的刪除語法中可能會被靜默忽略。在 Laravel 13 中，這些子句會被包含在產生的 SQL 中。因此，不支援此語法的資料庫引擎（例如標準 MySQL / MariaDB 衍生版本）現在可能會拋出 `QueryException`，而不是執行沒有界限範圍的刪除。

<a name="eloquent"></a>
### Eloquent


<a name="model-booting-and-nested-instantiation"></a>
#### Model 啟動與巢狀實例化

**受影響的可能性：極低**

當 Model 仍在啟動 (Booting) 時建立新的 Model 實例現在已被禁止，且會拋出 `LogicException`。

這會影響到從 Model 的 `boot` 方法或 Trait 的 `boot*` 方法內部實例化 Model 的程式碼：

```php
protected static function boot()
{
    parent::boot();

    // No longer allowed during booting...
    (new static())->getTable();
}
```

請將此邏輯移至啟動週期之外，以避免巢狀啟動。


<a name="polymorphic-pivot-table-name-generation"></a>
#### 多型樞紐資料表名稱生成

**受影響的可能性：低**

當使用自訂樞紐 (Pivot) Model 類別推導多型樞紐 Model 的資料表名稱時，Laravel 現在會生成複數名稱。

如果您的應用程式依賴以往多型樞紐資料表被推導為單數名稱的行為，並且使用了自訂樞紐類別，您應該在 Pivot Model 上明確定義資料表名稱。


<a name="collection-model-serialization-restores-eager-loaded-relations"></a>
#### Collection Model 序列化會還原預先載入的關聯

**受影響的可能性：低**

當 Eloquent Model Collection 被序列化並還原時（例如在佇列任務中），集合中的 Model 現在會還原其預先載入的關聯 (Eager-loaded relations)。

如果您的程式碼依賴反序列化後不保留關聯的行為，您可能需要調整該邏輯。


<a name="http-client"></a>
### HTTP Client


<a name="http-client-response-throw-and-throwif-signatures"></a>
#### HTTP Client `Response::throw` 與 `throwIf` 的方法簽名

**受影響的可能性：極低**

HTTP Client 的回應方法現在在其方法簽名中宣告了 Callback 參數：

```php
public function throw($callback = null);
public function throwIf($condition, $callback = null);
```

如果您在自訂的回應類別中覆寫了這些方法，請確保您的方法簽名保持相容。


<a name="notifications"></a>
### Notifications


<a name="default-password-reset-subject"></a>
#### 預設密碼重置郵件主旨

**受影響的可能性：極低**

Laravel 預設的重置密碼郵件主旨已變更：

```text
// Laravel <= 12.x
Reset Password Notification

// Laravel >= 13.x
Reset your password
```

如果您的測試、斷言 (Assertions) 或語系翻譯覆寫檔依賴之前的預設字串，請相應地更新它們。


<a name="queued-notifications-and-missing-models"></a>
#### 佇列通知與遺失的 Model

**受影響的可能性：極低**

佇列通知現在會遵守定義在 Notification 類別上的 `#[DeleteWhenMissingModels]` 屬性與 `$deleteWhenMissingModels` 屬性。

在先前版本中，即使您期望佇列通知任務在 Model 遺失時被刪除，缺少 Model 仍可能導致該任務失敗。


<a name="queue"></a>
### Queue


<a name="jobattempted-event-exception-payload"></a>
#### `JobAttempted` 事件 Exception Payload

**受影響的可能性：低**

`Illuminate\Queue\Events\JobAttempted` 事件現在透過 `$exception` 公開例外物件（或 `null`），取代了先前布林值的 `$exceptionOccurred` 屬性：

```php
// Laravel <= 12.x
$event->exceptionOccurred;

// Laravel >= 13.x
$event->exception;
```

如果您有監聽此事件，請相應地更新您的監聽器程式碼。


<a name="queuebusy-event-property-rename"></a>
#### `QueueBusy` 事件屬性重新命名

**受影響的可能性：低**

為了與其他佇列事件保持一致，`Illuminate\Queue\Events\QueueBusy` 事件的 `$connection` 屬性已重新命名為 `$connectionName`。

如果您的監聽器參考了 `$connection`，請將其更新為 `$connectionName`。


<a name="queue-contract-method-additions"></a>
#### `Queue` 契約 (Contract) 方法新增

**受影響的可能性：極低**

`Illuminate\Contracts\Queue\Queue` 契約 (Contract) 現在包含了佇列大小檢查方法，這些方法先前僅在 Docblock 中宣告。

如果您維護實作此契約的自訂佇列驅動程式，請為以下方法新增實作：

<div class="content-list" markdown="1">

- `pendingSize`
- `delayedSize`
- `reservedSize`
- `creationTimeOfOldestPendingJob`

</div>


<a name="routing"></a>
### Routing


<a name="domain-route-registration-precedence"></a>
#### 網域路由註冊優先順序

**受影響的可能性：低**

具有明確網域的路由現在在路由比對時優先於非網域路由。

這使得萬用 (Catch-all) 子網域路由即使在非網域路由較早註冊的情況下也能保持行為一致。如果您的應用程式依賴先前網域與非網域路由之間的註冊優先順序，請檢視路由比對行為。


<a name="session"></a>
### Session


<a name="session-serialization-configuration"></a>
#### Session `serialization` 設定

**受影響的可能性：低**

為有助於防止 PHP 反序列化 Gadget Chain 攻擊，預設的應用程式骨架現在將 `config/session.php` 檔案中的 Session `serialization` 選項設為 `json`。

如果您正在升級現有的應用程式，並將設定檔與 Laravel 13 骨架同步，將此值從 `php` 更新為 `json` 會使所有目前活躍的使用者 Session 失效。

如果您希望在升級過程中無縫保持活躍的 Session，應確保此值保持設為 `php`。然而，如果您的應用程式未在 Session 中儲存 PHP 物件，且您不介意要求使用者重新進行認證，我們建議將此值更新為 `json` 以提高安全性。


<a name="scheduling"></a>
### Scheduling


<a name="withscheduling-registration-timing"></a>
#### `withScheduling` 註冊時機

**受影響的可能性：極低**

透過 `ApplicationBuilder::withScheduling()` 註冊的排程現在會延遲到 `Schedule` 解析時才執行。

如果您的應用程式依賴引導 (Bootstrap) 期間的即時排程註冊時機，您可能需要調整該邏輯。


<a name="security"></a>
### Security


<a name="request-forgery-protection"></a>
#### 請求偽造保護 (Request Forgery Protection)

**受影響的可能性：高**

Laravel 的 CSRF 中介層已從 `VerifyCsrfToken` 重新命名為 `PreventRequestForgery`，現在還包含使用 `Sec-Fetch-Site` 標頭的請求來源驗證。

`VerifyCsrfToken` 與 `ValidateCsrfToken` 仍作為已棄用 (Deprecated) 的別名保留，但應將直接參考更新為 `PreventRequestForgery`，特別是在測試或路由定義中排除中介層時：

```php
use Illuminate\Foundation\Http\Middleware\PreventRequestForgery;
use Illuminate\Foundation\Http\Middleware\VerifyCsrfToken;

// Laravel <= 12.x
->withoutMiddleware([VerifyCsrfToken::class]);

// Laravel >= 13.x
->withoutMiddleware([PreventRequestForgery::class]);
```

中介層設定 API 現在也提供了 `preventRequestForgery(...)`。


<a name="support"></a>
### Support


<a name="manager-extend-callback-binding"></a>
#### Manager `extend` Callback 綁定

**受影響的可能性：低**

透過 Manager 的 `extend` 方法註冊的自訂驅動程式 Closure 現在會綁定至 Manager 實例。

如果您先前依賴這些 Callback 內部的 `$this` 作為另一個綁定物件（例如服務提供者實例），您應該使用 `use (...)` 將這些值傳入 Closure 擷取中。


<a name="str-factories-reset-between-tests"></a>
#### 測試之間重置 `Str` 工廠 (Factory)

**受影響的可能性：低**

Laravel 現在會在測試拆卸 (Teardown) 期間重置自訂的 `Str` 工廠 (Factory)。

如果您的測試依賴於自訂 UUID / ULID / 隨機字串工廠在測試方法之間的持久化，您應該在每個相關的測試或 setup 鉤子 (Hook) 中進行設定。


<a name="jsfrom-uses-unescaped-unicode-by-default"></a>
#### `Js::from` 預設使用 Unescaped Unicode

**受影響的可能性：極低**

`Illuminate\Support\Js::from` 現在預設使用 `JSON_UNESCAPED_UNICODE`。

如果您的測試或前端輸出比較依賴轉義的 Unicode 序列（例如 `\u00e8`），請更新您的預期值。

<a name="utilities"></a>
### 公用工具 (Utilities)


<a name="symfony-polyfill"></a>
#### Symfony PHP 8.5 Polyfill 與全域函式衝突

**影響可能性：低**

Laravel 13 引入了對 `symfony/polyfill-php85` 的依賴。在低於 8.5 的 PHP 版本上，除非這些全域函式已在啟動（bootstrap）過程中提早定義，否則此 polyfill 會定義如 `array_first()` 與 `array_last()` 等全域函式。

這些函式可能會與舊版的輔助函式套件（例如 `laravel/helpers`）或使用相同名稱的自訂全域輔助函式發生衝突。例如，過往的 `array_first()` 輔助函式可接受一個回呼函式（callback）來回傳第一個符合條件的元素，而 polyfill 版本的函式則僅會回傳陣列的第一個元素。

為了避免衝突並確保跨 PHP 版本的一致行為，您應該優先使用 `Illuminate\Support\Arr` 的方法：

```php
use Illuminate\Support\Arr;

Arr::first($array, function ($value) {
  return /* condition */;
});
```


<a name="views"></a>
### 視圖 (Views)


<a name="pagination-bootstrap-view-names"></a>
#### 分頁 Bootstrap 視圖名稱

**影響可能性：低**

Bootstrap 3 預設的內部分頁視圖名稱現在變得更加明確：

```nothing
// Laravel <= 12.x
pagination::default
pagination::simple-default

// Laravel >= 13.x
pagination::bootstrap-3
pagination::simple-bootstrap-3
```

若您的應用程式有直接引用舊的分頁視圖名稱，請更新這些引用。


<a name="miscellaneous"></a>
### 雜項

我們也鼓勵您查看 `laravel/laravel` [GitHub 儲存庫](https://github.com/laravel/laravel) 中的變更。雖然其中許多變更並非強制性要求，但您可能希望讓這些檔案與您的應用程式保持同步。這份升級指南會涵蓋部分的變更，但其他變更（例如設定檔或註解的變更）則不會包含在內。您可以輕鬆地透過 [GitHub 比較工具](https://github.com/laravel/laravel/compare/12.x...13.x) 查看這些變更，並選擇對您重要的更新項目。