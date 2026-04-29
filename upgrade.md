# 升級指南

- [從 12.x 升級至 13.0](#upgrade-13.0)
    - [使用 AI 進行升級](#upgrading-using-ai)

<a name="high-impact-changes"></a>
## 高影響變更

<div class="content-list" markdown="1">

- [更新依賴項目](#updating-dependencies)
- [更新 Laravel 安裝程式](#updating-the-laravel-installer)
- [請求偽造防護](#request-forgery-protection)

</div>


<a name="medium-impact-changes"></a>
## 中影響變更

<div class="content-list" markdown="1">

- [快取 `serializable_classes` 設定](#cache-serializable_classes-configuration)
- [在 MySQL 或 MariaDB 使用資料庫 `upsert`](#database-upsert-mariadb-mysql)

</div>


<a name="low-impact-changes"></a>
## 低影響變更

<div class="content-list" markdown="1">

- [快取前綴與 Session Cookie 名稱](#cache-prefixes-and-session-cookie-names)
- [Collection 模型序列化會還原預載關聯](#collection-model-serialization-restores-eager-loaded-relations)
- [`Container::call` 與可為 Null 的類別預設值](#containercall-and-nullable-class-defaults)
- [網域路由註冊優先順序](#domain-route-registration-precedence)
- [`JobAttempted` 事件異常酬載](#jobattempted-event-exception-payload)
- [Manager `extend` 回呼綁定](#manager-extend-callback-binding)
- [包含 `JOIN`、`ORDER BY` 與 `LIMIT` 的 MySQL `DELETE` 查詢](#mysql-delete-queries-with-join-order-by-and-limit)
- [分頁 Bootstrap 視圖名稱](#pagination-bootstrap-view-names)
- [多型樞紐表名稱生成](#polymorphic-pivot-table-name-generation)
- [`QueueBusy` 事件屬性重新命名](#queuebusy-event-property-rename)
- [`Str` 工廠在測試間重設](#str-factories-reset-between-tests)

</div>

<a name="upgrade-13.0"></a>
## 從 12.x 升級至 13.0


#### 預計升級時間：10 分鐘

> [!NOTE]
> 我們嘗試記錄每一個可能的破壞性變更 (Breaking Change)。由於其中某些破壞性變更位於框架中較冷門的部分，因此實際上只有一部分變更會影響您的應用程式。為了節省時間，您可以使用 [Shift](https://laravelshift.com)，這是一個由社群維護的服務，可自動化進行 Laravel 的升級。


<a name="upgrading-using-ai"></a>
### 使用 AI 進行升級

您可以使用 [Laravel Boost](https://github.com/laravel/boost) 來自動化升級。Boost 是一個第一方 MCP 伺服器，可為您的 AI 助手提供引導式的升級提示詞 — 只要在任何 Laravel 12 應用程式中安裝後，在 Claude Code、Cursor、OpenCode、Gemini 或 VS Code 中使用 `/upgrade-laravel-v13` 斜線命令，即可開始升級至 Laravel 13。此命令需要 Laravel Boost `^2.0`。


<a name="updating-dependencies"></a>
### 更新依賴項目

**影響可能性：高**

您應該更新應用程式 `composer.json` 檔案中的以下依賴項目：

<div class="content-list" markdown="1">

- `laravel/framework` 提升至 `^13.0`
- `laravel/boost` 提升至 `^2.0`
- `laravel/tinker` 提升至 `^3.0`
- `phpunit/phpunit` 提升至 `^12.0`
- `pestphp/pest` 提升至 `^4.0`

</div>


<a name="updating-the-laravel-installer"></a>
### 更新 Laravel 安裝程式

如果您正使用 Laravel 安裝程式 CLI 工具來建立新的 Laravel 應用程式，則應更新您的安裝程式以支援 Laravel 13.x 的相容性。

如果您是透過 `composer global require` 安裝 Laravel 安裝程式，可以使用 `composer global update` 來更新安裝程式：

```shell
composer global update laravel/installer
```

或者，如果您正使用 [Laravel Herd](https://herd.laravel.com) 內建的 Laravel 安裝程式，則應將您的 Herd 安裝更新至最新版本。


<a name="cache"></a>
### 快取


<a name="cache-prefixes-and-session-cookie-names"></a>
#### 快取前綴與 Session Cookie 名稱

**影響可能性：低**

Laravel 的預設快取與 Redis 鍵名前綴現在使用連字號 (Hyphen) 作為後綴。

在大多數應用程式中，這項變更不會產生影響，因為應用程式層級的設定檔已經定義了這些值。這主要會影響那些依賴框架層級備用 (Fallback) 設定（即應用程式設定值不存在時）的應用程式。

如果您的應用程式依賴這些自動產生的預設值，升級後快取鍵名與 Session Cookie 名稱可能會改變：

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

若要保留先前的行為，請在您的環境變數中明確設定 `CACHE_PREFIX`、`REDIS_PREFIX` 與 `SESSION_COOKIE`。


<a name="store-and-repository-contracts-touch"></a>
#### `Store` 與 `Repository` 契約(Contracts)：`touch`

**影響可能性：極低**

快取契約(Contracts) 現在包含一個 `touch` 方法，用於延長項目的存活時間 (TTL)。如果您有維護自定義的快取儲存實作，則應新增此方法：

```php
// Illuminate\Contracts\Cache\Store
public function touch($key, $seconds);
```


<a name="cache-serializable_classes-configuration"></a>
#### 快取 `serializable_classes` 設定

**影響可能性：中**

應用程式預設的 `cache` 設定現在包含一個設為 `false` 的 `serializable_classes` 選項。這強化了快取反序列化 (Unserialization) 行為，以助於在應用程式的 `APP_KEY` 洩漏時，防止 PHP 反序列化小工具鏈 (Gadget Chain) 攻擊。如果您的應用程式有刻意在快取中儲存 PHP 物件，則應明確列出允許反序列化的類別：

```php
'serializable_classes' => [
    App\Data\CachedDashboardStats::class,
    App\Support\CachedPricingSnapshot::class,
],
```

如果您的應用程式先前依賴反序列化任意快取物件，則需要將該用法遷移至明確的類別白名單，或是改用非物件的快取內容（例如陣列）。


<a name="container"></a>
### 容器 (Container)


<a name="containercall-and-nullable-class-defaults"></a>
#### `Container::call` 與可為 Null 的類別預設值

**影響可能性：低**

`Container::call` 現在於綁定 (Binding) 不存在時，會尊重可為 Null 的類別參數預設值，這與 Laravel 12 中引入的建構子注入行為一致：

```php
$container->call(function (?Carbon $date = null) {
    return $date;
});

// Laravel <= 12.x: Carbon instance
// Laravel >= 13.x: null
```

如果您的方法呼叫注入邏輯依賴先前的行為，則可能需要進行更新。


<a name="contracts"></a>
### 契約 (Contracts)


<a name="dispatcher-contract-dispatchafterresponse"></a>
#### `Dispatcher` 契約(Contracts)：`dispatchAfterResponse`

**影響可能性：極低**

`Illuminate\Contracts\Bus\Dispatcher` 契約(Contracts) 現在包含 `dispatchAfterResponse($command, $handler = null)` 方法。

如果您有維護自定義的 Dispatcher 實作，請將此方法新增至您的類別中。


<a name="responsefactory-contract-eventstream"></a>
#### `ResponseFactory` 契約(Contracts)：`eventStream`

**影響可能性：極低**

`Illuminate\Contracts\Routing\ResponseFactory` 契約(Contracts) 現在包含 `eventStream` 方法簽署。

如果您有維護此契約(Contracts) 的自定義實作，則應新增此方法。


<a name="mustverifyemail-contract-markemailasunverified"></a>
#### `MustVerifyEmail` 契約(Contracts)：`markEmailAsUnverified`

**影響可能性：極低**

`Illuminate\Contracts\Auth\MustVerifyEmail` 契約(Contracts) 現在包含 `markEmailAsUnverified()`。

如果您有提供此契約(Contracts) 的自定義實作，請新增此方法以保持相容性。


<a name="database"></a>
### 資料庫


<a name="database-upsert-mariadb-mysql"></a>
#### MySQL 或 MariaDB 的資料庫 `upsert`

**影響可能性：中**

Laravel 現在會驗證呼叫者是否為 `uniqueBy` 提供了一個非空值，若為空值則會拋出 `InvalidArgumentException`，而非產生無效的 SQL。

雖然 MariaDB 與 MySQL 資料庫驅動程式會忽略 `uniqueBy` 的值，並始終使用資料表的初級金鑰 (Primary Key) 與唯一索引來偵測現有記錄，但此驗證仍然適用。如果 `uniqueBy` 為空值，將會拋出 `InvalidArgumentException`。


<a name="mysql-delete-queries-with-join-order-by-and-limit"></a>
#### 包含 `JOIN`、`ORDER BY` 與 `LIMIT` 的 MySQL `DELETE` 查詢

**影響可能性：低**

Laravel 現在會針對 MySQL 語法編譯完整的 `DELETE ... JOIN` 查詢，包含 `ORDER BY` 與 `LIMIT`。

在先前的版本中，`ORDER BY` / `LIMIT` 子句在連結刪除 (Joined Deletes) 時可能會被無聲地忽略。在 Laravel 13 中，這些子句會被包含在產生的 SQL 中。因此，不支援此語法的資料庫引擎（例如標準的 MySQL / MariaDB 變體）現在可能會拋出 `QueryException`，而非執行一個未限定範圍的刪除。

<a name="eloquent"></a>
### Eloquent


<a name="model-booting-and-nested-instantiation"></a>
#### Model Booting and Nested Instantiation

**影響可能性：極低**

在 Model 仍在啟動 (Booting) 時建立新的 Model 實例現在是不被允許的，並且會拋出 `LogicException`。

這會影響在 Model 的 `boot` 方法或 Trait 的 `boot*` 方法中實例化 Model 的程式碼：

```php
protected static function boot()
{
    parent::boot();

    // No longer allowed during booting...
    (new static())->getTable();
}
```

請將此邏輯移出啟動週期以避免巢狀啟動。


<a name="polymorphic-pivot-table-name-generation"></a>
#### Polymorphic Pivot Table Name Generation

**影響可能性：低**

當使用自定義樞紐 Model 類別為多型樞紐 Model 推斷資料表名稱時，Laravel 現在會生成複數名稱。

如果您的應用程式依賴先前推斷出的單數多型樞紐資料表名稱，且使用了自定義樞紐類別，則應在樞紐模型中明確定義資料表名稱。


<a name="collection-model-serialization-restores-eager-loaded-relations"></a>
#### Collection Model Serialization Restores Eager-Loaded Relations

**影響可能性：低**

當 Eloquent Model Collection 被序列化並還原時（例如在佇列工作中），系統現在會為該 Collection 的 Model 還原預載 (Eager-loaded) 的關聯。

如果您的程式碼依賴反序列化後關聯不存在的行為，您可能需要調整該邏輯。


<a name="http-client"></a>
### HTTP Client


<a name="http-client-response-throw-and-throwif-signatures"></a>
#### HTTP Client `Response::throw` and `throwIf` Signatures

**影響可能性：極低**

HTTP Client 的回應方法現在於方法定義特徵 (Signatures) 中宣告了其回呼 (Callback) 參數：

```php
public function throw($callback = null);
public function throwIf($condition, $callback = null);
```

如果您在自定義的回應類別中重寫了這些方法，請確保您的方法定義特徵是相容的。


<a name="notifications"></a>
### Notifications


<a name="default-password-reset-subject"></a>
#### Default Password Reset Subject

**影響可能性：極低**

Laravel 預設的密碼重設郵件主旨已更改：

```text
// Laravel <= 12.x
Reset Password Notification

// Laravel >= 13.x
Reset your password
```

如果您的測試、斷言或翻譯覆蓋設定依賴先前的預設字串，請進行相應更新。


<a name="queued-notifications-and-missing-models"></a>
#### Queued Notifications and Missing Models

**影響可能性：極低**

佇列通知現在會遵循通知類別中定義的 `#[DeleteWhenMissingModels]` 屬性與 `$deleteWhenMissingModels` 變數。

在先前版本中，即便您預期缺失的 Model 應該被刪除，但在某些情況下仍可能導致佇列通知工作失敗。


<a name="queue"></a>
### Queue


<a name="jobattempted-event-exception-payload"></a>
#### `JobAttempted` Event Exception Payload

**影響可能性：低**

`Illuminate\Queue\Events\JobAttempted` 事件現在透過 `$exception` 公開例外物件 (或 `null`)，取代了先前布林值的 `$exceptionOccurred` 屬性：

```php
// Laravel <= 12.x
$event->exceptionOccurred;

// Laravel >= 13.x
$event->exception;
```

如果您有監聽此事件，請相應更新您的監聽器程式碼。


<a name="queuebusy-event-property-rename"></a>
#### `QueueBusy` Event Property Rename

**影響可能性：低**

`Illuminate\Queue\Events\QueueBusy` 事件的屬性 `$connection` 已重新命名為 `$connectionName`，以與其他佇列事件保持一致。

如果您的監聽器引用了 `$connection`，請更新為 `$connectionName`。


<a name="queue-contract-method-additions"></a>
#### `Queue` Contract Method Additions

**影響可能性：極低**

`Illuminate\Contracts\Queue\Queue` 契約 (Contracts) 現在包含了先前僅在 Docblocks 中宣告的佇列大小檢視方法。

如果您維護實作此契約的自定義佇列驅動程式，請為以下內容新增實作：

<div class="content-list" markdown="1">

- `pendingSize`
- `delayedSize`
- `reservedSize`
- `creationTimeOfOldestPendingJob`

</div>


<a name="routing"></a>
### Routing


<a name="domain-route-registration-precedence"></a>
#### Domain Route Registration Precedence

**影響可能性：低**

在路由匹配中，具有明確網域的路由現在優先於非網域路由。

這使得萬用字元 (Catch-all) 子網域路由即使在非網域路由較早註冊的情況下，也能表現一致。如果您的應用程式依賴先前網域與非網域路由之間的註冊優先順序，請檢視路由匹配行為。


<a name="scheduling"></a>
### Scheduling


<a name="withscheduling-registration-timing"></a>
#### `withScheduling` Registration Timing

**影響可能性：極低**

透過 `ApplicationBuilder::withScheduling()` 註冊的排程現在會延遲到 `Schedule` 被解析 (Resolved) 時才處理。

如果您的應用程式依賴引導 (Bootstrap) 期間的立即排程註冊時機，您可能需要調整該邏輯。


<a name="security"></a>
### Security


<a name="request-forgery-protection"></a>
#### Request Forgery Protection

**影響可能性：高**

Laravel 的 CSRF 中介層已從 `VerifyCsrfToken` 重新命名為 `PreventRequestForgery`，現在包含使用 `Sec-Fetch-Site` 標頭的請求來源驗證。

`VerifyCsrfToken` 與 `ValidateCsrfToken` 仍作為已棄用的別名保留，但應將直接引用更新為 `PreventRequestForgery`，特別是在測試或路由定義中排除中介層時：

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
#### Manager `extend` Callback Binding

**影響可能性：低**

透過 Manager 的 `extend` 方法註冊的自定義驅動程式閉包 (Closures) 現在會綁定到該 Manager 實例。

如果您先前在這些回呼中依賴另一個綁定物件（例如服務提供者實例）作為 `$this`，您應使用 `use (...)` 將這些值移入閉包擷取中。


<a name="str-factories-reset-between-tests"></a>
#### `Str` Factories Reset Between Tests

**影響可能性：低**

Laravel 現在會在測試拆解 (Teardown) 期間重置自定義的 `Str` 工廠。

如果您的測試依賴於在測試方法之間持久存在的自定義 UUID / ULID / 隨機字串工廠，您應該在每個相關測試或 Setup 鉤子中設定它們。


<a name="jsfrom-uses-unescaped-unicode-by-default"></a>
#### `Js::from` Uses Unescaped Unicode By Default

**影響可能性：極低**

`Illuminate\Support\Js::from` 現在預設使用 `JSON_UNESCAPED_UNICODE`。

如果您的測試或前端輸出比較依賴轉義的 Unicode 序列（例如 `\u00e8`），請更新您的預期值。


<a name="utilities"></a>
### Utilities


<a name="symfony-polyfill"></a>
#### Symfony PHP 8.5 Polyfill and Global Function Conflicts

**影響可能性：低**

Laravel 13 引入了對 `symfony/polyfill-php85` 的依賴。在低於 PHP 8.5 的版本上，此 Polyfill 會定義如 `array_first()` 與 `array_last()` 之類的全域函式，除非它們在引導期間已提前定義。

這些函式可能與舊有的輔助函式套件（如 `laravel/helpers`）或使用相同名稱的自定義全域輔助函式發生衝突。例如，歷史悠久的 `array_first()` 輔助函式接受一個回呼來回傳第一個匹配的元素，而 Polyfill 版本僅回傳陣列的第一個元素。

為了避免衝突並確保跨 PHP 版本的一致行為，您應該優先使用 `Illuminate\Support\Arr` 方法：

```php
use Illuminate\Support\Arr;

Arr::first($array, function ($value) {
  return /* condition */;
});
```

<a name="views"></a>
### 視圖 (Views)


<a name="pagination-bootstrap-view-names"></a>
#### Bootstrap 分頁視圖名稱

**影響可能性：低**

Bootstrap 3 預設的內部分頁視圖名稱現在已改為明確的名稱：

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
### 其他 (Miscellaneous)

我們也鼓勵您查看 `laravel/laravel` [GitHub 儲存庫](https://github.com/laravel/laravel) 中的變更。雖然其中許多變更並非必須，但您可能希望讓這些檔案與您的應用程式保持同步。本升級指南將涵蓋其中部分變更，但其他變更（例如對設定檔或註解的修改）則不會。您可以透過 [GitHub 比較工具](https://github.com/laravel/laravel/compare/12.x...13.x) 輕鬆查看變更，並選擇對您而言重要的更新。