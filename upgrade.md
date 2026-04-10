# 升級指南

- [從 12.x 升級至 13.0](#upgrade-13.0)
    - [使用 AI 進行升級](#upgrading-using-ai)

<a name="high-impact-changes"></a>
## 高影響變更

<div class="content-list" markdown="1">

- [更新依賴項目](#updating-dependencies)
- [更新 Laravel 安裝程式](#updating-the-laravel-installer)
- [請求偽造保護](#request-forgery-protection)

</div>


<a name="medium-impact-changes"></a>
## 中影響變更

<div class="content-list" markdown="1">

- [快取 `serializable_classes` 設定](#cache-serializable_classes-configuration)

</div>


<a name="low-impact-changes"></a>
## 低影響變更

<div class="content-list" markdown="1">

- [快取前綴與工作階段 Cookie 名稱](#cache-prefixes-and-session-cookie-names)
- [集合模型序列化會還原預先載入的關聯](#collection-model-serialization-restores-eager-loaded-relations)
- [`Container::call` 與可為 null 的類別預設值](#containercall-and-nullable-class-defaults)
- [網域路由註冊優先順序](#domain-route-registration-precedence)
- [`JobAttempted` 事件的異常酬載](#jobattempted-event-exception-payload)
- [Manager `extend` 回呼綁定](#manager-extend-callback-binding)
- [包含 `JOIN`、`ORDER BY` 與 `LIMIT` 的 MySQL `DELETE` 查詢](#mysql-delete-queries-with-join-order-by-and-limit)
- [分頁 Bootstrap 視圖名稱](#pagination-bootstrap-view-names)
- [多型樞紐表名稱生成](#polymorphic-pivot-table-name-generation)
- [`QueueBusy` 事件屬性重新命名](#queuebusy-event-property-rename)
- [測試之間重設 `Str` 工廠](#str-factories-reset-between-tests)

</div>

<a name="upgrade-13.0"></a>
## 從 12.x 升級至 13.0


#### 預計升級時間：10 分鐘

> [!NOTE]
> 我們嘗試記錄每一個可能的重大變更。由於其中一些重大變更位於框架中較冷門的部分，因此只有一部分的變更實際上會影響您的應用程式。為了節省時間，您可以使用 [Shift](https://laravelshift.com)。Shift 是一個由社群維護的服務，可自動執行 Laravel 升級。


<a name="upgrading-using-ai"></a>
### 使用 AI 進行升級

您可以使用 [Laravel Boost](https://github.com/laravel/boost) 來自動化您的升級。Boost 是一個官方的 MCP 伺服器，可為您的 AI 助手提供引導式升級提示 — 一旦安裝在任何 Laravel 12 應用程式中，在 Claude Code、Cursor、OpenCode、Gemini 或 VS Code 中使用 `/upgrade-laravel-v13` 斜線指令即可開始升級至 Laravel 13。此指令需要 Laravel Boost `^2.0`。


<a name="updating-dependencies"></a>
### 更新相依套件

**影響可能性：高**

您應該更新應用程式 `composer.json` 檔案中的以下相依套件：

<div class="content-list" markdown="1">

- `laravel/framework` 至 `^13.0`
- `laravel/boost` 至 `^2.0`
- `laravel/tinker` 至 `^3.0`
- `phpunit/phpunit` 至 `^12.0`
- `pestphp/pest` 至 `^4.0`

</div>


<a name="updating-the-laravel-installer"></a>
### 更新 Laravel 安裝程式

如果您使用 Laravel 安裝程式 CLI 工具來建立新的 Laravel 應用程式，您應該更新安裝程式以相容於 Laravel 13.x。

如果您是透過 `composer global require` 安裝 Laravel 安裝程式，可以使用 `composer global update` 來更新安裝程式：

```shell
composer global update laravel/installer
```

或者，如果您使用的是 [Laravel Herd](https://herd.laravel.com) 內建的 Laravel 安裝程式副本，您應該將 Herd 安裝更新至最新版本。


<a name="cache"></a>
### 快取


<a name="cache-prefixes-and-session-cookie-names"></a>
#### 快取前綴與工作階段 Cookie 名稱

**影響可能性：低**

Laravel 預設的快取與 Redis 金鑰前綴現在使用連字號後綴。此外，預設的工作階段 Cookie 名稱現在對應用程式名稱使用 `Str::snake(...)`。

在大多數應用程式中，此變更不會生效，因為應用程式層級的設定檔已經定義了這些值。這主要影響那些在對應的應用程式設定值不存在時，依賴框架層級後備設定的應用程式。

如果您的應用程式依賴於這些產生的預設值，升級後快取金鑰與工作階段 Cookie 名稱可能會發生變化：

```php
// Laravel <= 12.x
Str::slug((string) env('APP_NAME', 'laravel'), '_').'_cache_';
Str::slug((string) env('APP_NAME', 'laravel'), '_').'_database_';
Str::slug((string) env('APP_NAME', 'laravel'), '_').'_session';

// Laravel >= 13.x
Str::slug((string) env('APP_NAME', 'laravel')).'-cache-';
Str::slug((string) env('APP_NAME', 'laravel')).'-database-';
Str::snake((string) env('APP_NAME', 'laravel')).'_session';
```

若要保留先前的行為，請在您的環境中明確設定 `CACHE_PREFIX`、`REDIS_PREFIX` 與 `SESSION_COOKIE`。


<a name="store-and-repository-contracts-touch"></a>
#### `Store` 與 `Repository` 契約(Contracts)：`touch`

**影響可能性：極低**

快取契約現在包含一個 `touch` 方法用於延長項目的 TTL。如果您維護自定義的快取儲存實作，應新增此方法：

```php
// Illuminate\Contracts\Cache\Store
public function touch($key, $seconds);
```


<a name="cache-serializable_classes-configuration"></a>
#### 快取 `serializable_classes` 設定

**影響可能性：中**

預設的應用程式 `cache` 設定現在包含一個設定為 `false` 的 `serializable_classes` 選項。這強化了快取反序列化的行為，以幫助在您的 `APP_KEY` 洩露時防止 PHP 反序列化 Gadget 鏈攻擊。如果您的應用程式有意將 PHP 物件儲存在快取中，您應該明確列出可被反序列化的類別：

```php
'serializable_classes' => [
    App\Data\CachedDashboardStats::class,
    App\Support\CachedPricingSnapshot::class,
],
```

如果您的應用程式先前依賴於反序列化任意的快取物件，您需要將該用法遷移至明確的類別允許清單或非物件的快取負載（例如陣列）。


<a name="container"></a>
### 容器


<a name="containercall-and-nullable-class-defaults"></a>
#### `Container::call` 與可為 null 的類別預設值

**影響可能性：低**

`Container::call` 現在在不存在綁定時會遵循可為 null 的類別參數預設值，這與 Laravel 12 引入的建構子注入行為一致：

```php
$container->call(function (?Carbon $date = null) {
    return $date;
});

// Laravel <= 12.x: Carbon instance
// Laravel >= 13.x: null
```

如果您的方法呼叫注入邏輯依賴於先前的行為，您可能需要對其進行更新。


<a name="contracts"></a>
### 契約(Contracts)


<a name="dispatcher-contract-dispatchafterresponse"></a>
#### `Dispatcher` 契約(Contracts)：`dispatchAfterResponse`

**影響可能性：極低**

`Illuminate\Contracts\Bus\Dispatcher` 契約現在包含 `dispatchAfterResponse($command, $handler = null)` 方法。

如果您維護自定義的 dispatcher 實作，請將此方法新增至您的類別中。


<a name="responsefactory-contract-eventstream"></a>
#### `ResponseFactory` 契約(Contracts)：`eventStream`

**影響可能性：極低**

`Illuminate\Contracts\Routing\ResponseFactory` 契約現在包含 `eventStream` 簽署。

如果您維護此契約的自定義實作，應該新增此方法。


<a name="mustverifyemail-contract-markemailasunverified"></a>
#### `MustVerifyEmail` 契約(Contracts)：`markEmailAsUnverified`

**影響可能性：極低**

`Illuminate\Contracts\Auth\MustVerifyEmail` 契約現在包含 `markEmailAsUnverified()`。

如果您提供此契約的自定義實作，請新增此方法以保持相容。


<a name="database"></a>
### 資料庫


<a name="mysql-delete-queries-with-join-order-by-and-limit"></a>
#### MySQL 包含 `JOIN`、`ORDER BY` 與 `LIMIT` 的 `DELETE` 查詢

**影響可能性：低**

Laravel 現在針對 MySQL 語法編譯完整的 `DELETE ... JOIN` 查詢，包括 `ORDER BY` 與 `LIMIT`。

在先前版本中，`ORDER BY` / `LIMIT` 子句在結合刪除 (joined deletes) 時可能會被靜默忽略。在 Laravel 13 中，這些子句被包含在產生的 SQL 中。因此，不支援此語法的資料庫引擎（例如標準的 MySQL / MariaDB 變體）現在可能會拋出 `QueryException` 而不是執行無限制的刪除。

<a name="eloquent"></a>
### Eloquent


<a name="model-booting-and-nested-instantiation"></a>
#### 模型啟動與巢狀實例化

**影響可能性：極低**

現在不允許在模型仍在啟動 (booting) 時建立新的模型實例，否則會拋出 `LogicException`。

這會影響到在模型 `boot` 方法或 trait 的 `boot*` 方法中實例化模型的程式碼：

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
#### 多型樞紐表名稱生成

**影響可能性：低**

當使用自定義樞紐模型類別為多型樞紐模型推斷表名時，Laravel 現在會生成複數名稱。

如果您的應用程式依賴於先前多型樞紐表的單數推斷名稱且使用了自定義樞紐類別，您應該在樞紐模型上明確定義表名。


<a name="collection-model-serialization-restores-eager-loaded-relations"></a>
#### 集合模型序列化將還原預先載入的關聯

**影響可能性：低**

當 Eloquent 模型集合被序列化並還原時（例如在佇列工作中），集合中的模型現在會還原其預先載入的關聯。

如果您的程式碼依賴於反序列化後關聯不存在的情況，您可能需要調整該邏輯。


<a name="http-client"></a>
### HTTP Client


<a name="http-client-response-throw-and-throwif-signatures"></a>
#### HTTP Client `Response::throw` 與 `throwIf` 簽名

**影響可能性：極低**

HTTP 用戶端的回應方法現在在方法簽名中宣告其回呼參數：

```php
public function throw($callback = null);
public function throwIf($condition, $callback = null);
```

如果您在自定義回應類別中覆寫這些方法，請確保您的方法簽名是相容的。


<a name="notifications"></a>
### Notifications


<a name="default-password-reset-subject"></a>
#### 預設的密碼重設主旨

**影響可能性：極低**

Laravel 預設的密碼重設郵件主旨已變更：

```text
// Laravel <= 12.x
Reset Password Notification

// Laravel >= 13.x
Reset your password
```

如果您的測試、斷言或翻譯覆寫依賴於之前的預設字串，請相應地進行更新。


<a name="queued-notifications-and-missing-models"></a>
#### 佇列通知與缺失的模型

**影響可能性：極低**

佇列通知現在會遵循定義在通知類別上的 `#[DeleteWhenMissingModels]` 屬性與 `$deleteWhenMissingModels` 屬性。

在之前的版本中，在您預期模型應被刪除的情況下，缺失的模型仍可能導致佇列通知工作失敗。


<a name="queue"></a>
### Queue


<a name="jobattempted-event-exception-payload"></a>
#### `JobAttempted` 事件的異常負載

**影響可能性：低**

`Illuminate\Queue\Events\JobAttempted` 事件現在透過 `$exception` 揭露異常物件（或 `null`），取代了之前的布林值 `$exceptionOccurred` 屬性：

```php
// Laravel <= 12.x
$event->exceptionOccurred;

// Laravel >= 13.x
$event->exception;
```

如果您在監聽此事件，請相應地更新您的監聽器程式碼。


<a name="queuebusy-event-property-rename"></a>
#### `QueueBusy` 事件屬性重新命名

**影響可能性：低**

`Illuminate\Queue\Events\QueueBusy` 事件的 `$connection` 屬性已重新命名為 `$connectionName`，以與其他佇列事件保持一致。

如果您的監聽器引用了 `$connection`，請將其更新為 `$connectionName`。


<a name="queue-contract-method-additions"></a>
#### `Queue` 契約方法新增

**影響可能性：極低**

`Illuminate\Contracts\Queue\Queue` 契約現在包含了先前僅在 docblocks 中宣告的佇列大小檢查方法。

如果您維護此契約的自定義佇列驅動實作，請新增以下方法的實作：

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

**影響可能性：低**

在路由比對中，具有明確網域的路由現在優先於非網域路由。

這使得全匹配子網域路由即使在非網域路由較早註冊時也能保持一致的行為。如果您的應用程式依賴於先前網域路由與非網域路由之間的註冊優先順序，請審查路由比對行為。


<a name="scheduling"></a>
### Scheduling


<a name="withscheduling-registration-timing"></a>
#### `withScheduling` 註冊時機

**影響可能性：極低**

透過 `ApplicationBuilder::withScheduling()` 註冊的排程現在會延遲直到 `Schedule` 被解析。

如果您的應用程式依賴於啟動期間立即進行的排程註冊時機，您可能需要調整該邏輯。


<a name="security"></a>
### Security


<a name="request-forgery-protection"></a>
#### 請求偽造保護

**影響可能性：高**

Laravel 的 CSRF 中介層已從 `VerifyCsrfToken` 重新命名為 `PreventRequestForgery`，並且現在包含使用 `Sec-Fetch-Site` 標頭的請求來源驗證。

`VerifyCsrfToken` 與 `ValidateCsrfToken` 仍作為已棄用的別名保留，但直接引用應更新為 `PreventRequestForgery`，特別是在測試或路由定義中排除中介層時：

```php
use Illuminate\Foundation\Http\Middleware\PreventRequestForgery;
use Illuminate\Foundation\Http\Middleware\VerifyCsrfToken;

// Laravel <= 12.x
->withoutMiddleware([VerifyCsrfToken::class]);

// Laravel >= 13.x
->withoutMiddleware([PreventRequestForgery::class]);
```

中介層配置 API 現在也提供了 `preventRequestForgery(...)`。


<a name="support"></a>
### Support


<a name="manager-extend-callback-binding"></a>
#### Manager `extend` 回呼綁定

**影響可能性：低**

透過 manager `extend` 方法註冊的自定義驅動閉包現在會綁定到 manager 實例。

如果您先前在這些回呼內部依賴另一個綁定物件（例如服務提供者實例）作為 `$this`，您應該使用 `use (...)` 將這些值移至閉包捕獲中。


<a name="str-factories-reset-between-tests"></a>
#### 測試之間重設 `Str` 工廠

**影響可能性：低**

Laravel 現在會在測試清理 (teardown) 期間重設自定義 `Str` 工廠。

如果您的測試依賴於自定義 UUID / ULID / 隨機字串工廠在測試方法之間持續存在，您應該在每個相關測試或 setup 鉤子中設定它們。


<a name="jsfrom-uses-unescaped-unicode-by-default"></a>
#### `Js::from` 預設使用未轉義的 Unicode

**影響可能性：極低**

`Illuminate\Support\Js::from` 現在預設使用 `JSON_UNESCAPED_UNICODE`。

如果您的測試或前端輸出比較依賴於轉義的 Unicode 序列（例如 `\u00e8`），請更新您的預期結果。


<a name="views"></a>
### Views


<a name="pagination-bootstrap-view-names"></a>
#### 分頁 Bootstrap 視圖名稱

**影響可能性：低**

Bootstrap 3 預設的分頁內部視圖名稱現在已明確化：

```nothing
// Laravel <= 12.x
pagination::default
pagination::simple-default

// Laravel >= 13.x
pagination::bootstrap-3
pagination::simple-bootstrap-3
```

如果您的應用程式直接引用了舊的分頁視圖名稱，請更新這些引用。

<a name="miscellaneous"></a>
### 其他

我們也鼓勵您查看 `laravel/laravel` [GitHub 儲存庫](https://github.com/laravel/laravel) 中的變更。雖然其中許多變更並非必要，但您可能希望讓這些檔案與您的應用程式保持同步。這些變更中的一部分會包含在本升級指南中，但其他變更（例如設定檔或註解的變更）則不會。您可以使用 [GitHub 比較工具](https://github.com/laravel/laravel/compare/12.x...13.x)輕鬆查看變更，並選擇對您而言重要的更新。