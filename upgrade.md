# 升級指南

- [從 10.x 升級到 11.0](#upgrade-11.0)

<a name="high-impact-changes"></a>
## 高影響變更

<div class="content-list" markdown="1">

- [更新依賴套件](#updating-dependencies)
- [應用程式架構](#application-structure)
- [浮點型別](#floating-point-types)
- [修改欄位](#modifying-columns)
- [SQLite 最低版本](#sqlite-minimum-version)
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
- [The `Enumerable` 契約](#the-enumerable-contract)
- [The `UserProvider` 契約](#the-user-provider-contract)
- [The `Authenticatable` 契約](#the-authenticatable-contract)

</div>

<a name="upgrade-11.0"></a>
## 從 10.x 升級到 11.0

<a name="estimated-upgrade-time-??-minutes"></a>
#### 預估升級時間：15 分鐘

> [!NOTE]  
> 我們會盡力記錄所有可能導致破壞性變更的內容。由於其中一些破壞性變更發生於框架中較不常見的部分，因此這些變更可能只會影響您的應用程式一小部分。想節省時間嗎？您可以使用 [Laravel Shift](https://laravelshift.com/) 來自動化應用程式的升級。

<a name="updating-dependencies"></a>
### 更新依賴套件

**影響可能性：高**

#### 必備 PHP 8.2.0

Laravel 現在需要 PHP 8.2.0 或更高版本。

#### 必備 curl 7.34.0

Laravel 的 HTTP Client 現在需要 curl 7.34.0 或更高版本。

#### Composer 依賴套件

您應更新應用程式 `composer.json` 檔案中的下列依賴套件：

<div class="content-list" markdown="1">

- `laravel/framework` to `^11.0`
- `nunomaduro/collision` to `^8.1`
- `laravel/breeze` to `^2.0` (如果已安裝)
- `laravel/cashier` to `^15.0` (如果已安裝)
- `laravel/dusk` to `^8.0` (如果已安裝)
- `laravel/jetstream` to `^5.0` (如果已安裝)
- `laravel/octane` to `^2.3` (如果已安裝)
- `laravel/passport` to `^12.0` (如果已安裝)
- `laravel/sanctum` to `^4.0` (如果已安裝)
- `laravel/scout` to `^10.0` (如果已安裝)
- `laravel/spark-stripe` to `^5.0` (如果已安裝)
- `laravel/telescope` to `^5.0` (如果已安裝)
- `livewire/livewire` to `^3.4` (如果已安裝)
- `inertiajs/inertia-laravel` to `^1.0` (如果已安裝)

</div>

如果您的應用程式正在使用 Laravel Cashier Stripe, Passport, Sanctum, Spark Stripe, 或 Telescope，您將需要將它們的遷移檔案發布到您的應用程式。Cashier Stripe, Passport, Sanctum, Spark Stripe, 及 Telescope **不再自動從其自身的遷移檔案**目錄中載入遷移。因此，您應該執行以下命令將它們的遷移檔案發布到您的應用程式：

```bash
php artisan vendor:publish --tag=cashier-migrations
php artisan vendor:publish --tag=passport-migrations
php artisan vendor:publish --tag=sanctum-migrations
php artisan vendor:publish --tag=spark-migrations
php artisan vendor:publish --tag=telescope-migrations
```

此外，您應該審閱每個套件的升級指南，以確保您了解任何額外的破壞性變更：

- [Laravel Cashier Stripe](#cashier-stripe)
- [Laravel Passport](#passport)
- [Laravel Sanctum](#sanctum)
- [Laravel Spark Stripe](#spark-stripe)
- [Laravel Telescope](#telescope)

如果您已手動安裝 Laravel installer，您應該透過 Composer 更新 installer：

```bash
composer global require laravel/installer:^5.6
```

最後，如果您先前已將 `doctrine/dbal` Composer 依賴套件添加到您的應用程式中，則可以將其移除，因為 Laravel 不再依賴此套件。

<a name="application-structure"></a>
### 應用程式結構

Laravel 11 引入了一個新的預設應用程式結構，其預設檔案較少。也就是說，新的 Laravel 應用程式包含較少的 Service Provider、中介層 (middleware) 和組態檔 (configuration files)。

然而，我們**不建議**從 Laravel 10 升級到 Laravel 11 的應用程式嘗試遷移其應用程式結構，因為 Laravel 11 已經過精心調整，以同時支援 Laravel 10 的應用程式結構。

<a name="authentication"></a>
### 認證

<a name="password-rehashing"></a>
#### 密碼重新雜湊

**影響可能性：低**

如果您的雜湊演算法 (hashing algorithm) 的「工作因子 (work factor)」自上次雜湊密碼後已更新，Laravel 11 將在認證期間自動重新雜湊您使用者的密碼。

通常，這不會中斷您的應用程式；但是，如果您的 `User` 模型的「password」欄位名稱不是 `password`，您應該透過模型的 `authPasswordName` 屬性指定該欄位的名稱：

    protected $authPasswordName = 'custom_password_field';

或者，您可以透過在應用程式的 `config/hashing.php` 組態檔中添加 `rehash_on_login` 選項來禁用密碼重新雜湊：

    'rehash_on_login' => false,

<a name="the-user-provider-contract"></a>
#### `UserProvider` 契約

**影響可能性：低**

`Illuminate\Contracts\Auth\UserProvider` 契約已收到一個新的 `rehashPasswordIfRequired` 方法。當應用程式的雜湊演算法工作因子發生變更時，此方法負責重新雜湊並將使用者的密碼儲存在儲存空間中。

如果您的應用程式或套件定義了一個實作此介面的類別，您應該將新的 `rehashPasswordIfRequired` 方法添加到您的實作中。參考實作可以在 `Illuminate\Auth\EloquentUserProvider` 類別中找到：

```php
public function rehashPasswordIfRequired(Authenticatable $user, array $credentials, bool $force = false);
```

<a name="the-authenticatable-contract"></a>
#### `Authenticatable` 契約

**影響可能性：低**

`Illuminate\Contracts\Auth\Authenticatable` 契約已收到一個新的 `getAuthPasswordName` 方法。此方法負責返回您的可認證實體 (authenticatable entity) 的密碼欄位名稱。

如果您的應用程式或套件定義了一個實作此介面的類別，您應該將新的 `getAuthPasswordName` 方法添加到您的實作中：

```php
public function getAuthPasswordName()
{
    return 'password';
}
```

Laravel 附帶的預設 `User` 模型會自動接收此方法，因為該方法包含在 `Illuminate\Auth\Authenticatable` Trait 中。

<a name="the-authentication-exception-class"></a>
#### `AuthenticationException` 類別

**影響可能性：非常低**

`Illuminate\Auth\AuthenticationException` 類別的 `redirectTo` 方法現在需要一個 `Illuminate\Http\Request` 實例作為其第一個參數。如果您手動捕捉此例外並呼叫 `redirectTo` 方法，您應該相應地更新您的程式碼：

```php
if ($e instanceof AuthenticationException) {
    $path = $e->redirectTo($request);
}
```

<a name="email-verification-notification-on-registration"></a>
#### 註冊時的 Email 驗證通知

**影響可能性：非常低**

如果 `Registered` 事件尚未由您的應用程式的 `EventServiceProvider` 註冊，`SendEmailVerificationNotification` 監聽器現在會自動為其註冊。如果您的應用程式的 `EventServiceProvider` 沒有註冊此監聽器，並且您不希望 Laravel 自動為您註冊，您應該在應用程式的 `EventServiceProvider` 中定義一個空的 `configureEmailVerification` 方法：

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

之前，如果為 DynamoDB、Memcached 或 Redis 快取儲存定義了快取鍵前綴，Laravel 會在該前綴後附加一個 `:`。在 Laravel 11 中，快取鍵前綴不會收到 `:` 後綴。如果您想保持之前的前綴行為，您可以手動在您的快取鍵前綴中添加 `:` 後綴。

<a name="collections"></a>
### 集合 (Collections)

<a name="the-enumerable-contract"></a>
#### `Enumerable` 契約

**影響可能性：低**

`Illuminate\Support\Enumerable` 契約的 `dump` 方法已更新為接受一個變數參數 `...$args`。如果您正在實作此介面，您應該相應地更新您的實作：

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
#### Eloquent Model `casts` 方法

**影響可能性：低**

基礎 Eloquent 模型類別現在定義了一個 `casts` 方法，以支援屬性型別轉換的定義。如果您的應用程式的其中一個模型定義了 `casts` 關聯，它可能會與基礎 Eloquent 模型類別中現在存在的 `casts` 方法衝突。

<a name="modifying-columns"></a>
#### 修改欄位

**影響可能性：高**

修改欄位時，您現在必須明確包含變更後希望保留在欄位定義上的所有修飾符。任何遺失的屬性都將被捨棄。例如，為了保留 `unsigned`、`default` 和 `comment` 屬性，即使這些屬性已經由先前的 migration 分配給欄位，您也必須在變更欄位時明確呼叫每個修飾符。

例如，假設您有一個 migration 創建了一個 `votes` 欄位，並帶有 `unsigned`、`default` 和 `comment` 屬性：

```php
Schema::create('users', function (Blueprint $table) {
    $table->integer('votes')->unsigned()->default(1)->comment('The vote count');
});
```

之後，您編寫了一個 migration，也將該欄位變更為 `nullable`：

```php
Schema::table('users', function (Blueprint $table) {
    $table->integer('votes')->nullable()->change();
});
```

在 Laravel 10 中，這個 migration 會保留欄位的 `unsigned`、`default` 和 `comment` 屬性。然而，在 Laravel 11 中，migration 現在還必須包含所有先前在欄位上定義的屬性。否則，它們將被捨棄：

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

`change` 方法不會變更欄位的索引。因此，您可以在修改欄位時使用索引修飾符來明確添加或捨棄索引：

```php
// Add an index...
$table->bigIncrements('id')->primary()->change();

// Drop an index...
$table->char('postal_code', 10)->unique(false)->change();
```

如果您不想更新應用程式中所有現有的「變更」migrations 來保留欄位的現有屬性，您可以直接 [壓縮您的 migrations](/docs/{{version}}/migrations#squashing-migrations)：

```bash
php artisan schema:dump
```

一旦您的 migrations 被壓縮，Laravel 將在執行任何待處理的 migrations 之前，使用您應用程式的 schema 檔案來「遷移」資料庫。

<a name="floating-point-types"></a>
#### 浮點數型別

**影響可能性：高**

`double` 和 `float` migration 欄位型別已重新編寫，以便在所有資料庫中保持一致。

`double` 欄位型別現在會建立一個與 `DOUBLE` 等效的欄位，不包含總位數和小數位數（小數點後的位數），這是標準的 SQL 語法。因此，您可以移除 `$total` 和 `$places` 的參數：

```php
$table->double('amount');
```

`float` 欄位型別現在會建立一個與 `FLOAT` 等效的欄位，不包含總位數和小數位數（小數點後的位數），但帶有可選的 `$precision` 規範，以決定儲存大小為 4-byte 單精度欄位或 8-byte 雙精度欄位。因此，您可以移除 `$total` 和 `$places` 的參數，並根據您的資料庫文件將可選的 `$precision` 指定為您所需的值：

```php
$table->float('amount', precision: 53);
```

`unsignedDecimal`、`unsignedDouble` 和 `unsignedFloat` 方法已被移除，因為這些欄位型別的 unsigned 修飾符已被 MySQL 棄用，並且從未在其他資料庫系統中標準化。然而，如果您希望繼續使用這些欄位型別的已棄用 unsigned 屬性，您可以將 `unsigned` 方法鏈接到欄位的定義中：

```php
$table->decimal('amount', total: 8, places: 2)->unsigned();
$table->double('amount')->unsigned();
$table->float('amount', precision: 53)->unsigned();
```

<a name="dedicated-mariadb-driver"></a>
#### 專用 MariaDB 驅動程式

**影響可能性：非常低**

Laravel 11 增加了專用於 MariaDB 的資料庫驅動程式，而非在連接 MariaDB 資料庫時總是使用 MySQL 驅動程式。

如果您的應用程式連接到 MariaDB 資料庫，您可以將連線設定更新為新的 `mariadb` 驅動程式，以便在未來受益於 MariaDB 特有的功能：

    'driver' => 'mariadb',
    'url' => env('DB_URL'),
    'host' => env('DB_HOST', '127.0.0.1'),
    'port' => env('DB_PORT', '3306'),
    // ...

目前，新的 MariaDB 驅動程式行為與目前的 MySQL 驅動程式相似，但有一個例外：`uuid` schema builder 方法會建立原生的 UUID 欄位，而不是 `char(36)` 欄位。

如果您現有的 migrations 使用 `uuid` schema builder 方法，並且您選擇使用新的 `mariadb` 資料庫驅動程式，您應該將 migration 中對 `uuid` 方法的呼叫更新為 `char`，以避免破壞性變更或意外行為：

```php
Schema::table('users', function (Blueprint $table) {
    $table->char('uuid', 36);

    // ...
});
```

<a name="spatial-types"></a>
#### 空間型別

**影響可能性：低**

資料庫 migrations 中的空間欄位型別已重新編寫，以便在所有資料庫中保持一致。因此，您可以從您的 migrations 中移除 `point`、`lineString`、`polygon`、`geometryCollection`、`multiPoint`、`multiLineString`、`multiPolygon` 和 `multiPolygonZ` 方法，改為使用 `geometry` 或 `geography` 方法：

```php
$table->geometry('shapes');
$table->geography('coordinates');
```

若要明確限制 MySQL、MariaDB 和 PostgreSQL 上欄位中儲存值的型別或空間參考系統識別符，您可以將 `subtype` 和 `srid` 傳遞給該方法：

```php
$table->geometry('dimension', subtype: 'polygon', srid: 0);
$table->geography('latitude', subtype: 'point', srid: 4326);
```

PostgreSQL 語法中的 `isGeometry` 和 `projection` 欄位修飾符已相應移除。

<a name="doctrine-dbal-removal"></a>
#### 移除 Doctrine DBAL

**影響可能性：低**

以下列出的 Doctrine DBAL 相關類別和方法已被移除。Laravel 不再依賴此套件，並且不再需要註冊自訂的 Doctrines 型別，即可正確建立和修改先前需要自訂型別的各種欄位型別：

<div class="content-list" markdown="1">

- `Illuminate\Database\Schema\Builder::$alwaysUsesNativeSchemaOperationsIfPossible` class property
- `Illuminate\Database\Schema\Builder::useNativeSchemaOperationsIfPossible()` method
- `Illuminate\Database\Connection::usingNativeSchemaOperations()` method
- `Illuminate\Database\Connection::isDoctrineAvailable()` method
- `Illuminate\Database\Connection::getDoctrineConnection()` method
- `Illuminate\Database\Connection::getDoctrineSchemaManager()` method
- `Illuminate\Database\Connection::getDoctrineColumn()` method
- `Illuminate\Database\Connection::registerDoctrineType()` method
- `Illuminate\Database\DatabaseManager::registerDoctrineType()` method
- `Illuminate\Database\PDO` directory
- `Illuminate\Database\DBAL\TimestampType` class
- `Illuminate\Database\Schema\Grammars\ChangeColumn` class
- `Illuminate\Database\Schema\Grammars\RenameColumn` class
- `Illuminate\Database\Schema\Grammars\Grammar::getDoctrineTableDiff()` method

</div>

此外，不再需要在應用程式的 `database` 設定檔中透過 `dbal.types` 註冊自訂 Doctrine 型別。

如果您之前使用 Doctrine DBAL 來檢查您的資料庫及其相關表格，您可以改用 Laravel 新的原生 schema 方法（`Schema::getTables()`、`Schema::getColumns()`、`Schema::getIndexes()`、`Schema::getForeignKeys()` 等）。

<a name="deprecated-schema-methods"></a>
#### 已棄用的 Schema 方法

**影響可能性：非常低**

已棄用、基於 Doctrine 的 `Schema::getAllTables()`、`Schema::getAllViews()` 和 `Schema::getAllTypes()` 方法已被移除，改為使用新的 Laravel 原生 `Schema::getTables()`、`Schema::getViews()` 和 `Schema::getTypes()` 方法。

當使用 PostgreSQL 和 SQL Server 時，所有新的 schema 方法都不會接受三段式參考（例如 `database.schema.table`）。因此，您應該改用 `connection()` 來宣告資料庫：

```php
Schema::connection('database')->hasTable('schema.table');
```

<a name="get-column-types"></a>
#### Schema Builder `getColumnType()` 方法

**影響可能性：非常低**

`Schema::getColumnType()` 方法現在總是回傳給定欄位的實際型別，而不是 Doctrine DBAL 等效型別。

<a name="database-connection-interface"></a>
#### 資料庫連線介面

**影響可能性：非常低**

`Illuminate\Database\ConnectionInterface` 介面已新增一個 `scalar` 方法。如果您正在定義自己的此介面實作，您應該將 `scalar` 方法新增到您的實作中：

```php
public function scalar($query, $bindings = [], $useReadPdo = true);
```

<a name="dates"></a>
### 日期


<a name="carbon-3"></a>
#### Carbon 3

**影響可能性: 中**

Laravel 11 同時支援 Carbon 2 和 Carbon 3。Carbon 是被 Laravel 和整個生態系統中的套件廣泛使用的日期操作函式庫。如果您升級到 Carbon 3，請注意 `diffIn*` 方法現在會回傳浮點數，並且可能會回傳負值以表示時間方向，這與 Carbon 2 有顯著差異。請查閱 Carbon 的 [變更日誌](https://github.com/briannesbitt/Carbon/releases/tag/3.0.0) 和 [文件](https://carbon.nesbot.com/docs/#api-carbon-3) 以獲取有關如何處理這些及其他變更的詳細資訊。


<a name="mail"></a>
### 郵件


<a name="the-mailer-contract"></a>
#### `Mailer` 契約

**影響可能性: 極低**

`Illuminate\Contracts\Mail\Mailer` 契約已新增了一個 `sendNow` 方法。如果您的應用程式或套件手動實作此契約，您應該將新的 `sendNow` 方法新增到您的實作中：

```php
public function sendNow($mailable, array $data = [], $callback = null);
```


<a name="packages"></a>
### 套件


<a name="publishing-service-providers"></a>
#### 將 Service Providers 發佈到應用程式

**影響可能性: 極低**

如果您編寫了一個 Laravel 套件，該套件手動將 Service Provider 發佈到應用程式的 `app/Providers` 目錄，並手動修改應用程式的 `config/app.php` 配置檔以註冊該 Service Provider，您應該更新您的套件以使用新的 `ServiceProvider::addProviderToBootstrapFile` 方法。

`addProviderToBootstrapFile` 方法會自動將您已發佈的 Service Provider 加入到應用程式的 `bootstrap/providers.php` 檔案中，因為在新的 Laravel 11 應用程式中，`providers` 陣列不存在於 `config/app.php` 配置檔中。

```php
use Illuminate\Support\ServiceProvider;

ServiceProvider::addProviderToBootstrapFile(Provider::class);
```


<a name="queues"></a>
### 佇列


<a name="the-batch-repository-interface"></a>
#### `BatchRepository` 介面

**影響可能性: 極低**

`Illuminate\Bus\BatchRepository` 介面已新增了一個 `rollBack` 方法。如果您在自己的套件或應用程式中實作此介面，您應該將此方法新增到您的實作中：

```php
public function rollBack();
```


<a name="synchronous-jobs-in-database-transactions"></a>
#### 資料庫交易中的同步化 Job

**影響可能性: 極低**

以前，同步化 Job (使用 `sync` 佇列驅動程式的 Job) 會立即執行，無論佇列連線的 `after_commit` 配置選項是否設定為 `true`，或 `afterCommit` 方法是否在 Job 上被呼叫。

在 Laravel 11 中，同步化佇列 Job 現在將遵循佇列連線或 Job 的「after commit」配置。


<a name="rate-limiting"></a>
### 速率限制


<a name="per-second-rate-limiting"></a>
#### 每秒速率限制

**影響可能性: 中**

Laravel 11 支援每秒速率限制，而不再限於每分鐘的粒度。關於這項變更，您應該注意各種潛在的破壞性變更。

`GlobalLimit` 類別的建構函式現在接受秒數而不是分鐘數。此類別並未被文件說明，通常不會被您的應用程式使用：

```php
new GlobalLimit($attempts, 2 * 60);
```

`Limit` 類別的建構函式現在接受秒數而不是分鐘數。此類別的所有文件說明的使用方式都限於靜態建構函式，例如 `Limit::perMinute` 和 `Limit::perSecond`。但是，如果您手動實例化此類別，則應更新您的應用程式，向類別的建構函式提供秒數：

```php
new Limit($key, $attempts, 2 * 60);
```

`Limit` 類別的 `decayMinutes` 屬性已更名為 `decaySeconds`，現在包含秒數而不是分鐘數。

`Illuminate\Queue\Middleware\ThrottlesExceptions` 和 `Illuminate\Queue\Middleware\ThrottlesExceptionsWithRedis` 類別的建構函式現在接受秒數而不是分鐘數：

```php
new ThrottlesExceptions($attempts, 2 * 60);
new ThrottlesExceptionsWithRedis($attempts, 2 * 60);
```


<a name="cashier-stripe"></a>
### Cashier Stripe


<a name="updating-cashier-stripe"></a>
#### 更新 Cashier Stripe

**影響可能性: 高**

Laravel 11 不再支援 Cashier Stripe 14.x。因此，您應該在應用程式的 `composer.json` 檔案中，將 Laravel Cashier Stripe 的依賴更新為 `^15.0`。

Cashier Stripe 15.0 不再自動從其本身的遷移目錄載入遷移。相反地，您應該執行以下指令將 Cashier Stripe 的遷移發佈到您的應用程式：

```shell
php artisan vendor:publish --tag=cashier-migrations
```

請查閱完整的 [Cashier Stripe 升級指南](https://github.com/laravel/cashier-stripe/blob/15.x/UPGRADE.md) 以了解其他破壞性變更。


<a name="spark-stripe"></a>
### Spark (Stripe)


<a name="updating-spark-stripe"></a>
#### 更新 Spark Stripe

**影響可能性: 高**

Laravel 11 不再支援 Laravel Spark Stripe 4.x。因此，您應該在應用程式的 `composer.json` 檔案中，將 Laravel Spark Stripe 的依賴更新為 `^5.0`。

Spark Stripe 5.0 不再自動從其本身的遷移目錄載入遷移。相反地，您應該執行以下指令將 Spark Stripe 的遷移發佈到您的應用程式：

```shell
php artisan vendor:publish --tag=spark-migrations
```

請查閱完整的 [Spark Stripe 升級指南](https://spark.laravel.com/docs/spark-stripe/upgrade.html) 以了解其他破壞性變更。


<a name="passport"></a>
### Passport


<a name="updating-telescope"></a>
#### 更新 Passport

**影響可能性: 高**

Laravel 11 不再支援 Laravel Passport 11.x。因此，您應該在應用程式的 `composer.json` 檔案中，將 Laravel Passport 的依賴更新為 `^12.0`。

Passport 12.0 不再自動從其本身的遷移目錄載入遷移。相反地，您應該執行以下指令將 Passport 的遷移發佈到您的應用程式：

```shell
php artisan vendor:publish --tag=passport-migrations
```

此外，密碼授權類型預設為禁用。您可以透過在應用程式的 `AppServiceProvider` 的 `boot` 方法中呼叫 `enablePasswordGrant` 方法來啟用它：

    public function boot(): void
    {
        Passport::enablePasswordGrant();
    }


<a name="sanctum"></a>
### Sanctum


<a name="updating-sanctum"></a>
#### 更新 Sanctum

**影響可能性: 高**

Laravel 11 不再支援 Laravel Sanctum 3.x。因此，您應該在應用程式的 `composer.json` 檔案中，將 Laravel Sanctum 的依賴更新為 `^4.0`。

Sanctum 4.0 不再自動從其本身的遷移目錄載入遷移。相反地，您應該執行以下指令將 Sanctum 的遷移發佈到您的應用程式：

```shell
php artisan vendor:publish --tag=sanctum-migrations
```

然後，在您的應用程式的 `config/sanctum.php` 配置檔中，您應該將對 `authenticate_session`、`encrypt_cookies` 和 `validate_csrf_token` 中介層的引用更新為以下內容：

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

Laravel 11 不再支援 Laravel Telescope 4.x。因此，您應該在應用程式的 `composer.json` 檔案中，將 Laravel Telescope 依賴套件更新到 `^5.0`。

Telescope 5.0 不再自動從其本身的 migrations 目錄載入 migrations。您應該執行以下指令，將 Telescope 的 migrations 發佈到您的應用程式：

```shell
php artisan vendor:publish --tag=telescope-migrations
```

<a name="spatie-once-package"></a>
### Spatie Once 套件

**影響可能性：中**

Laravel 11 現在提供了自己的 [`once` 函式](/docs/{{version}}/helpers#method-once)，以確保給定的閉包只會執行一次。因此，如果您的應用程式依賴於 `spatie/once` 套件，您應該從應用程式的 `composer.json` 檔案中移除它，以避免衝突。

<a name="miscellaneous"></a>
### 雜項

我們也鼓勵您查看 `laravel/laravel` [GitHub 儲存庫](https://github.com/laravel/laravel)中的變更。儘管其中許多變更不是必需的，但您可能會希望讓這些檔案與您的應用程式保持同步。本升級指南將涵蓋其中一些變更，但其他變更，例如設定檔或註解的變更，則不會涵蓋。您可以輕鬆地使用 [GitHub 比較工具](https://github.com/laravel/laravel/compare/10.x...11.x) 來查看這些變更，並選擇對您而言重要的更新。