# 升級指南

- [從 10.x 升級到 11.0](#upgrade-11.0)

<a name="high-impact-changes"></a>
## 高影響變更

<div class="content-list" markdown="1">

- [更新依賴套件](#updating-dependencies)
- [應用程式結構](#application-structure)
- [浮點數型別](#floating-point-types)
- [修改欄位](#modifying-columns)
- [SQLite 最低版本要求](#sqlite-minimum-version)
- [更新 Sanctum](#updating-sanctum)

</div>

<a name="medium-impact-changes"></a>
## 中影響變更

<div class="content-list" markdown="1">

- [Carbon 3](#carbon-3)
- [密碼重新雜湊](#password-rehashing)
- [每秒速率限制](#per-second-rate-limiting)
- [Spatie Once 套件](#spatie-once-package)

</div>

<a name="low-impact-changes"></a>
## 低影響變更

<div class="content-list" markdown="1">

- [移除 Doctrine DBAL](#doctrine-dbal-removal)
- [Eloquent Model 的 `casts` 方法](#eloquent-model-casts-method)
- [空間型別](#spatial-types)
- [`Enumerable` 契約](#the-enumerable-contract)
- [`UserProvider` 契約](#the-user-provider-contract)
- [`Authenticatable` 契約](#the-authenticatable-contract)

</div>

<a name="upgrade-11.0"></a>
## 從 10.x 升級到 11.0

<a name="estimated-upgrade-time-??-minutes"></a>
#### 預計升級時間：15 分鐘

> [!NOTE]
> 我們試圖記錄所有可能導致破壞性變更的內容。由於其中一些破壞性變更位於框架中較為隱蔽的部分，因此實際上可能只會影響您應用程式的一部分。想要節省時間嗎？您可以使用 [Laravel Shift](https://laravelshift.com/) 來協助自動化您的應用程式升級。

<a name="updating-dependencies"></a>
### 更新依賴套件

**影響可能性：高**

#### PHP 8.2.0 或更高版本要求

Laravel 現在要求 PHP 8.2.0 或更高版本。

#### curl 7.34.0 或更高版本要求

Laravel 的 HTTP Client 現在要求 curl 7.34.0 或更高版本。

#### Composer 依賴套件

您應該更新應用程式 `composer.json` 檔案中的以下依賴套件：

<div class="content-list" markdown="1">

- `laravel/framework` 到 `^11.0`
- `nunomaduro/collision` 到 `^8.1`
- `laravel/breeze` 到 `^2.0` (如果已安裝)
- `laravel/cashier` 到 `^15.0` (如果已安裝)
- `laravel/dusk` 到 `^8.0` (如果已安裝)
- `laravel/jetstream` 到 `^5.0` (如果已安裝)
- `laravel/octane` 到 `^2.3` (如果已安裝)
- `laravel/passport` 到 `^12.0` (如果已安裝)
- `laravel/sanctum` 到 `^4.0` (如果已安裝)
- `laravel/scout` 到 `^10.0` (如果已安裝)
- `laravel/spark-stripe` 到 `^5.0` (如果已安裝)
- `laravel/telescope` 到 `^5.0` (如果已安裝)
- `livewire/livewire` 到 `^3.4` (如果已安裝)
- `inertiajs/inertia-laravel` 到 `^1.0` (如果已安裝)

</div>

如果您的應用程式使用 Laravel Cashier Stripe、Passport、Sanctum、Spark Stripe 或 Telescope，您將需要將它們的 Migration 發佈到您的應用程式中。Cashier Stripe、Passport、Sanctum、Spark Stripe 和 Telescope **不再自動從它們自己的 Migration 目錄載入 Migration**。因此，您應該執行以下指令將它們的 Migration 發佈到您的應用程式中：

```bash
php artisan vendor:publish --tag=cashier-migrations
php artisan vendor:publish --tag=passport-migrations
php artisan vendor:publish --tag=sanctum-migrations
php artisan vendor:publish --tag=spark-migrations
php artisan vendor:publish --tag=telescope-migrations
```

此外，您應該查閱這些套件各自的升級指南，以確保您了解任何額外的破壞性變更：

- [Laravel Cashier Stripe](#cashier-stripe)
- [Laravel Passport](#passport)
- [Laravel Sanctum](#sanctum)
- [Laravel Spark Stripe](#spark-stripe)
- [Laravel Telescope](#telescope)

如果您已手動安裝 Laravel Installer，您應該透過 Composer 更新 Installer：

```bash
composer global require laravel/installer:^5.6
```

最後，如果您之前已將 `doctrine/dbal` Composer 依賴套件新增到您的應用程式中，現在可以將其移除，因為 Laravel 不再依賴此套件。

<a name="application-structure"></a>
### 應用程式結構

Laravel 11 引入了新的預設應用程式結構，其中包含較少的預設檔案。具體來說，新的 Laravel 應用程式包含較少的 Service Provider、Middleware 和設定檔。

然而，我們**不建議**將 Laravel 10 應用程式升級到 Laravel 11 時嘗試遷移其應用程式結構，因為 Laravel 11 已經過精心調整，也能支援 Laravel 10 的應用程式結構。

<a name="authentication"></a>
### 認證

<a name="password-rehashing"></a>
#### 密碼重新雜湊

**影響可能性：低**

如果您的雜湊演算法「工作因子」自密碼上次雜湊以來已更新，Laravel 11 將在認證期間自動重新雜湊您使用者的密碼。

通常，這不應中斷您的應用程式；但是，如果您的 `User` Model 的「password」欄位名稱不是 `password`，您應該透過 Model 的 `authPasswordName` 屬性指定欄位名稱：

    protected $authPasswordName = 'custom_password_field';

或者，您可以透過將 `rehash_on_login` 選項新增到應用程式的 `config/hashing.php` 設定檔中來停用密碼重新雜湊：

    'rehash_on_login' => false,

<a name="the-user-provider-contract"></a>
#### `UserProvider` 契約

**影響可能性：低**

`Illuminate\Contracts\Auth\UserProvider` 契約已收到一個新的 `rehashPasswordIfRequired` 方法。此方法負責在應用程式的雜湊演算法工作因子發生變更時，重新雜湊並將使用者密碼儲存在儲存中。

如果您的應用程式或套件定義了一個實作此介面的類別，您應該將新的 `rehashPasswordIfRequired` 方法新增到您的實作中。參考實作可以在 `Illuminate\Auth\EloquentUserProvider` 類別中找到：

```php
public function rehashPasswordIfRequired(Authenticatable $user, array $credentials, bool $force = false);
```

<a name="the-authenticatable-contract"></a>
#### `Authenticatable` 契約

**影響可能性：低**

`Illuminate\Contracts\Auth\Authenticatable` 契約已收到一個新的 `getAuthPasswordName` 方法。此方法負責回傳您的可認證實體的密碼欄位名稱。

如果您的應用程式或套件定義了一個實作此介面的類別，您應該將新的 `getAuthPasswordName` 方法新增到您的實作中：

```php
public function getAuthPasswordName()
{
    return 'password';
}
```

Laravel 隨附的預設 `User` Model 會自動接收此方法，因為該方法包含在 `Illuminate\Auth\Authenticatable` Trait 中。

<a name="the-authentication-exception-class"></a>
#### `AuthenticationException` 類別

**影響可能性：非常低**

`Illuminate\Auth\AuthenticationException` 類別的 `redirectTo` 方法現在需要一個 `Illuminate\Http\Request` 實例作為其第一個引數。如果您手動捕獲此例外並呼叫 `redirectTo` 方法，您應該相應地更新您的程式碼：

```php
if ($e instanceof AuthenticationException) {
    $path = $e->redirectTo($request);
}
```

<a name="email-verification-notification-on-registration"></a>
#### 註冊時的電子郵件驗證通知

**影響可能性：非常低**

如果 `SendEmailVerificationNotification` Listener 尚未由您的應用程式的 `EventServiceProvider` 註冊，它現在會自動為 `Registered` 事件註冊。如果您的應用程式的 `EventServiceProvider` 沒有註冊此 Listener，並且您不希望 Laravel 自動為您註冊它，您應該在應用程式的 `EventServiceProvider` 中定義一個空的 `configureEmailVerification` 方法：

```php
protected function configureEmailVerification()
{
    // ...
}
```

<a name="cache"></a>
### 快取

<a name="cache-key-prefixes"></a>
#### 快取鍵前綴

**影響可能性：非常低**

以前，如果為 DynamoDB、Memcached 或 Redis 快取儲存定義了快取鍵前綴，Laravel 會在該前綴後附加一個 `:`。在 Laravel 11 中，快取鍵前綴不再接收 `:` 後綴。如果您想維持以前的前綴行為，您可以手動將 `:` 後綴新增到您的快取鍵前綴中。

<a name="collections"></a>
### 集合

<a name="the-enumerable-contract"></a>
#### `Enumerable` 契約

**影響可能性：低**

`Illuminate\Support\Enumerable` 契約的 `dump` 方法已更新為接受可變引數 `...$args`。如果您正在實作此介面，您應該相應地更新您的實作：

```php
public function dump(...$args);
```

<a name="database"></a>
### 資料庫

<a name="sqlite-minimum-version"></a>
#### SQLite 3.26.0+

**影響可能性：高**

如果您的應用程式使用 SQLite 資料庫，則需要 SQLite 3.26.0 或更高版本。

<a name="eloquent-model-casts-method"></a>
#### Eloquent Model 的 `casts` 方法

**影響可能性：低**

基礎 Eloquent Model 類別現在定義了一個 `casts` 方法，以支援屬性 Cast 的定義。如果您應用程式的其中一個 Model 定義了一個 `casts` 關聯，它可能會與現在基礎 Eloquent Model 類別中存在的 `casts` 方法衝突。

<a name="modifying-columns"></a>
#### 修改欄位

**影響可能性：高**

修改欄位時，您現在必須明確包含所有您希望在欄位變更後保留的修飾符。任何遺失的屬性都將被移除。例如，要保留 `unsigned`、`default` 和 `comment` 屬性，即使這些屬性已由先前的 Migration 分配給欄位，您也必須在變更欄位時明確呼叫每個修飾符。

例如，假設您有一個 Migration 建立了一個具有 `unsigned`、`default` 和 `comment` 屬性的 `votes` 欄位：

```php
Schema::create('users', function (Blueprint $table) {
    $table->integer('votes')->unsigned()->default(1)->comment('The vote count');
});
```

稍後，您編寫了一個 Migration，將該欄位也變更為 `nullable`：

```php
Schema::table('users', function (Blueprint $table) {
    $table->integer('votes')->nullable()->change();
});
```

在 Laravel 10 中，此 Migration 將保留欄位上的 `unsigned`、`default` 和 `comment` 屬性。然而，在 Laravel 11 中，Migration 現在還必須包含所有先前在欄位上定義的屬性。否則，它們將被移除：

```php
Schema::table('users', function (Blueprint $table) {
    $table->integer('votes')
        ->unsigned()
        ->default(1)
        ->comment('The vote count')
        ->nullable()
        ->change();
});
```

`change` 方法不會變更欄位的索引。因此，您可以使用索引修飾符在修改欄位時明確新增或移除索引：

```php
// Add an index...
$table->bigIncrements('id')->primary()->change();

// Drop an index...
$table->char('postal_code', 10)->unique(false)->change();
```

如果您不想更新應用程式中所有現有的「change」Migration 以保留欄位的現有屬性，您可以簡單地[壓縮您的 Migration](/docs/{{version}}/migrations#squashing-migrations)：

```bash
php artisan schema:dump
```

一旦您的 Migration 被壓縮，Laravel 將使用您應用程式的 Schema 檔案「遷移」資料庫，然後執行任何待處理的 Migration。

<a name="floating-point-types"></a>
#### 浮點數型別

**影響可能性：高**

`double` 和 `float` Migration 欄位型別已重寫，以在所有資料庫中保持一致。

`double` 欄位型別現在建立一個沒有總位數和小數位數的 `DOUBLE` 等效欄位，這是標準 SQL 語法。因此，您可以移除 `$total` 和 `$places` 的引數：

```php
$table->double('amount');
```

`float` 欄位型別現在建立一個沒有總位數和小數位數的 `FLOAT` 等效欄位，但帶有一個可選的 `$precision` 規範，以根據您的資料庫文件確定儲存大小為 4 位元組單精度欄位或 8 位元組雙精度欄位。因此，您可以移除 `$total` 和 `$places` 的引數，並將可選的 `$precision` 指定為您所需的值：

```php
$table->float('amount', precision: 53);
```

`unsignedDecimal`、`unsignedDouble` 和 `unsignedFloat` 方法已被移除，因為這些欄位型別的 `unsigned` 修飾符已被 MySQL 棄用，並且從未在其他資料庫系統上標準化。但是，如果您希望繼續使用這些欄位型別的已棄用 `unsigned` 屬性，您可以將 `unsigned` 方法鏈接到欄位的定義上：

```php
$table->decimal('amount', total: 8, places: 2)->unsigned();
$table->double('amount')->unsigned();
$table->float('amount', precision: 53)->unsigned();
```

<a name="dedicated-mariadb-driver"></a>
#### 專用 MariaDB Driver

**影響可能性：非常低**

Laravel 11 不再在連接 MariaDB 資料庫時始終使用 MySQL Driver，而是為 MariaDB 新增了一個專用的資料庫 Driver。

如果您的應用程式連接到 MariaDB 資料庫，您可以將連接設定更新為新的 `mariadb` Driver，以便將來受益於 MariaDB 特定的功能：

    'driver' => 'mariadb',
    'url' => env('DB_URL'),
    'host' => env('DB_HOST', '127.0.0.1'),
    'port' => env('DB_PORT', '3306'),
    // ...

目前，新的 MariaDB Driver 的行為與目前的 MySQL Driver 相同，但有一個例外：`uuid` Schema Builder 方法會建立原生的 UUID 欄位，而不是 `char(36)` 欄位。

如果您現有的 Migration 使用 `uuid` Schema Builder 方法，並且您選擇使用新的 `mariadb` 資料庫 Driver，您應該將 Migration 中 `uuid` 方法的呼叫更新為 `char`，以避免破壞性變更或意外行為：

```php
Schema::table('users', function (Blueprint $table) {
    $table->char('uuid', 36);

    // ...
});
```

<a name="spatial-types"></a>
#### 空間型別

**影響可能性：低**

資料庫 Migration 的空間欄位型別已重寫，以在所有資料庫中保持一致。因此，您可以從 Migration 中移除 `point`、`lineString`、`polygon`、`geometryCollection`、`multiPoint`、`multiLineString`、`multiPolygon` 和 `multiPolygonZ` 方法，改用 `geometry` 或 `geography` 方法：

```php
$table->geometry('shapes');
$table->geography('coordinates');
```

要在 MySQL、MariaDB 和 PostgreSQL 上明確限制儲存在欄位中的值的型別或空間參考系統識別碼，您可以將 `subtype` 和 `srid` 傳遞給該方法：

```php
$table->geometry('dimension', subtype: 'polygon', srid: 0);
$table->geography('latitude', subtype: 'point', srid: 4326);
```

PostgreSQL Grammar 的 `isGeometry` 和 `projection` 欄位修飾符已相應移除。

<a name="doctrine-dbal-removal"></a>
#### 移除 Doctrine DBAL

**影響可能性：低**

以下 Doctrine DBAL 相關的類別和方法已被移除。Laravel 不再依賴此套件，並且不再需要註冊自訂 Doctrine 型別來正確建立和修改以前需要自訂型別的各種欄位型別：

<div class="content-list" markdown="1">

- `Illuminate\Database\Schema\Builder::$alwaysUsesNativeSchemaOperationsIfPossible` 類別屬性
- `Illuminate\Database\Schema\Builder::useNativeSchemaOperationsIfPossible()` 方法
- `Illuminate\Database\Connection::usingNativeSchemaOperations()` 方法
- `Illuminate\Database\Connection::isDoctrineAvailable()` 方法
- `Illuminate\Database\Connection::getDoctrineConnection()` 方法
- `Illuminate\Database\Connection::getDoctrineSchemaManager()` 方法
- `Illuminate\Database\Connection::getDoctrineColumn()` 方法
- `Illuminate\Database\Connection::registerDoctrineType()` 方法
- `Illuminate\Database\DatabaseManager::registerDoctrineType()` 方法
- `Illuminate\Database\PDO` 目錄
- `Illuminate\Database\DBAL\TimestampType` 類別
- `Illuminate\Database\Schema\Grammars\ChangeColumn` 類別
- `Illuminate\Database\Schema\Grammars\RenameColumn` 類別
- `Illuminate\Database\Schema\Grammars\Grammar::getDoctrineTableDiff()` 方法

</div>

此外，不再需要透過應用程式 `database` 設定檔中的 `dbal.types` 註冊自訂 Doctrine 型別。

如果您以前使用 Doctrine DBAL 來檢查您的資料庫及其相關表格，您可以使用 Laravel 新的原生 Schema 方法（`Schema::getTables()`、`Schema::getColumns()`、`Schema::getIndexes()`、`Schema::getForeignKeys()` 等）來代替。

<a name="deprecated-schema-methods"></a>
#### 已棄用的 Schema 方法

**影響可能性：非常低**

已棄用的、基於 Doctrine 的 `Schema::getAllTables()`、`Schema::getAllViews()` 和 `Schema::getAllTypes()` 方法已被移除，取而代之的是新的 Laravel 原生 `Schema::getTables()`、`Schema::getViews()` 和 `Schema::getTypes()` 方法。

當使用 PostgreSQL 和 SQL Server 時，新的 Schema 方法都不會接受三部分參考（例如 `database.schema.table`）。因此，您應該使用 `connection()` 來宣告資料庫：

```php
Schema::connection('database')->hasTable('schema.table');
```

<a name="get-column-types"></a>
#### Schema Builder 的 `getColumnType()` 方法

**影響可能性：非常低**

`Schema::getColumnType()` 方法現在始終回傳給定欄位的實際型別，而不是 Doctrine DBAL 等效型別。

<a name="database-connection-interface"></a>
#### 資料庫連接介面

**影響可能性：非常低**

`Illuminate\Database\ConnectionInterface` 介面已收到一個新的 `scalar` 方法。如果您正在定義此介面的自己的實作，您應該將 `scalar` 方法新增到您的實作中：

```php
public function scalar($query, $bindings = [], $useReadPdo = true);
```

<a name="dates"></a>
### 日期

<a name="carbon-3"></a>
#### Carbon 3

**影響可能性：中**

Laravel 11 支援 Carbon 2 和 Carbon 3。Carbon 是一個日期操作函式庫，被 Laravel 和整個生態系統中的套件廣泛使用。如果您升級到 Carbon 3，請注意 `diffIn*` 方法現在回傳浮點數，並且可能回傳負值以指示時間方向，這與 Carbon 2 有顯著差異。請查閱 Carbon 的[變更日誌](https://github.com/briannesbitt/Carbon/releases/tag/3.0.0)和[說明文件](https://carbon.nesbot.com/docs/#api-carbon-3)，以獲取有關如何處理這些和其他變更的詳細資訊。

<a name="mail"></a>
### 郵件

<a name="the-mailer-contract"></a>
#### `Mailer` 契約

**影響可能性：非常低**

`Illuminate\Contracts\Mail\Mailer` 契約已收到一個新的 `sendNow` 方法。如果您的應用程式或套件正在手動實作此契約，您應該將新的 `sendNow` 方法新增到您的實作中：

```php
public function sendNow($mailable, array $data = [], $callback = null);
```

<a name="packages"></a>
### 套件

<a name="publishing-service-providers-to-the-application"></a>
#### 將 Service Provider 發佈到應用程式

**影響可能性：非常低**

如果您編寫了一個 Laravel 套件，該套件手動將 Service Provider 發佈到應用程式的 `app/Providers` 目錄，並手動修改應用程式的 `config/app.php` 設定檔以註冊 Service Provider，您應該更新您的套件以利用新的 `ServiceProvider::addProviderToBootstrapFile` 方法。

`addProviderToBootstrapFile` 方法會自動將您發佈的 Service Provider 新增到應用程式的 `bootstrap/providers.php` 檔案中，因為在新的 Laravel 11 應用程式中，`providers` 陣列不存在於 `config/app.php` 設定檔中。

```php
use Illuminate\Support\ServiceProvider;

ServiceProvider::addProviderToBootstrapFile(Provider::class);
```

<a name="queues"></a>
### 佇列

<a name="the-batch-repository-interface"></a>
#### `BatchRepository` 介面

**影響可能性：非常低**

`Illuminate\Bus\BatchRepository` 介面已收到一個新的 `rollBack` 方法。如果您正在自己的套件或應用程式中實作此介面，您應該將此方法新增到您的實作中：

```php
public function rollBack();
```

<a name="synchronous-jobs-in-database-transactions"></a>
#### 資料庫交易中的同步 Job

**影響可能性：非常低**

以前，同步 Job（使用 `sync` 佇列 Driver 的 Job）會立即執行，無論佇列連接的 `after_commit` 設定選項是否設定為 `true`，或者 Job 上是否呼叫了 `afterCommit` 方法。

在 Laravel 11 中，同步佇列 Job 現在將遵守佇列連接或 Job 的「after commit」設定。

<a name="rate-limiting"></a>
### 速率限制

<a name="per-second-rate-limiting"></a>
#### 每秒速率限制

**影響可能性：中**

Laravel 11 支援每秒速率限制，而不是僅限於每分鐘的粒度。您應該注意與此變更相關的各種潛在破壞性變更。

`GlobalLimit` 類別建構函式現在接受秒數而不是分鐘數。此類別未經文件記錄，通常不會由您的應用程式使用：

```php
new GlobalLimit($attempts, 2 * 60);
```

`Limit` 類別建構函式現在接受秒數而不是分鐘數。此類別的所有文件記錄用法都僅限於靜態建構函式，例如 `Limit::perMinute` 和 `Limit::perSecond`。但是，如果您手動實例化此類別，您應該更新您的應用程式以向類別的建構函式提供秒數：

```php
new Limit($key, $attempts, 2 * 60);
```

`Limit` 類別的 `decayMinutes` 屬性已重新命名為 `decaySeconds`，現在包含秒數而不是分鐘數。

`Illuminate\Queue\Middleware\ThrottlesExceptions` 和 `Illuminate\Queue\Middleware\ThrottlesExceptionsWithRedis` 類別建構函式現在接受秒數而不是分鐘數：

```php
new ThrottlesExceptions($attempts, 2 * 60);
new ThrottlesExceptionsWithRedis($attempts, 2 * 60);
```

<a name="cashier-stripe"></a>
### Cashier Stripe

<a name="updating-cashier-stripe"></a>
#### 更新 Cashier Stripe

**影響可能性：高**

Laravel 11 不再支援 Cashier Stripe 14.x。因此，您應該在 `composer.json` 檔案中將應用程式的 Laravel Cashier Stripe 依賴套件更新為 `^15.0`。

Cashier Stripe 15.0 不再自動從其自己的 Migration 目錄載入 Migration。相反，您應該執行以下指令將 Cashier Stripe 的 Migration 發佈到您的應用程式中：

```shell
php artisan vendor:publish --tag=cashier-migrations
```

請查閱完整的 [Cashier Stripe 升級指南](https://github.com/laravel/cashier-stripe/blob/15.x/UPGRADE.md) 以獲取額外的破壞性變更。

<a name="spark-stripe"></a>
### Spark (Stripe)

<a name="updating-spark-stripe"></a>
#### 更新 Spark Stripe

**影響可能性：高**

Laravel 11 不再支援 Laravel Spark Stripe 4.x。因此，您應該在 `composer.json` 檔案中將應用程式的 Laravel Spark Stripe 依賴套件更新為 `^5.0`。

Spark Stripe 5.0 不再自動從其自己的 Migration 目錄載入 Migration。相反，您應該執行以下指令將 Spark Stripe 的 Migration 發佈到您的應用程式中：

```shell
php artisan vendor:publish --tag=spark-migrations
```

請查閱完整的 [Spark Stripe 升級指南](https://spark.laravel.com/docs/spark-stripe/upgrade.html) 以獲取額外的破壞性變更。

<a name="passport"></a>
### Passport

<a name="updating-telescope"></a>
#### 更新 Passport

**影響可能性：高**

Laravel 11 不再支援 Laravel Passport 11.x。因此，您應該在 `composer.json` 檔案中將應用程式的 Laravel Passport 依賴套件更新為 `^12.0`。

Passport 12.0 不再自動從其自己的 Migration 目錄載入 Migration。相反，您應該執行以下指令將 Passport 的 Migration 發佈到您的應用程式中：

```shell
php artisan vendor:publish --tag=passport-migrations
```

此外，預設情況下密碼授權類型是停用的。您可以透過在應用程式的 `AppServiceProvider` 的 `boot` 方法中呼叫 `enablePasswordGrant` 方法來啟用它：

    public function boot(): void
    {
        Passport::enablePasswordGrant();
    }

<a name="sanctum"></a>
### Sanctum

<a name="updating-sanctum"></a>
#### 更新 Sanctum

**影響可能性：高**

Laravel 11 不再支援 Laravel Sanctum 3.x。因此，您應該在 `composer.json` 檔案中將應用程式的 Laravel Sanctum 依賴套件更新為 `^4.0`。

Sanctum 4.0 不再自動從其自己的 Migration 目錄載入 Migration。相反，您應該執行以下指令將 Sanctum 的 Migration 發佈到您的應用程式中：

```shell
php artisan vendor:publish --tag=sanctum-migrations
```

然後，在應用程式的 `config/sanctum.php` 設定檔中，您應該將對 `authenticate_session`、`encrypt_cookies` 和 `validate_csrf_token` Middleware 的參考更新為以下內容：

    'middleware' => [
        'authenticate_session' => Laravel\Sanctum\Http\Middleware\AuthenticateSession::class,
        'encrypt_cookies' => Illuminate\Cookie\Middleware\EncryptCookies::class,
        'validate_csrf_token' => Illuminate\Foundation\Http\Middleware\ValidateCsrfToken::class,
    ],

<a name="telescope"></a>
### Telescope

<a name="updating-telescope"></a>
#### 更新 Telescope

**影響可能性：高**

Laravel 11 不再支援 Laravel Telescope 4.x。因此，您應該在 `composer.json` 檔案中將應用程式的 Laravel Telescope 依賴套件更新為 `^5.0`。

Telescope 5.0 不再自動從其自己的 Migration 目錄載入 Migration。相反，您應該執行以下指令將 Telescope 的 Migration 發佈到您的應用程式中：

```shell
php artisan vendor:publish --tag=telescope-migrations
```

<a name="spatie-once-package"></a>
### Spatie Once 套件

**影響可能性：中**

Laravel 11 現在提供了自己的 [`once` 函式](/docs/{{version}}/helpers#method-once) 以確保給定的 Closure 只執行一次。因此，如果您的應用程式依賴於 `spatie/once` 套件，您應該將其從應用程式的 `composer.json` 檔案中移除，以避免衝突。

<a name="miscellaneous"></a>
### 其他

我們也鼓勵您查看 `laravel/laravel` [GitHub 儲存庫](https://github.com/laravel/laravel)中的變更。雖然其中許多變更並非強制性，但您可能希望使這些檔案與您的應用程式保持同步。其中一些變更將在此升級指南中涵蓋，但其他變更，例如設定檔或註解的變更，則不會。您可以輕鬆使用 [GitHub 比較工具](https://github.com/laravel/laravel/compare/10.x...11.x) 查看變更，並選擇對您重要的更新。
