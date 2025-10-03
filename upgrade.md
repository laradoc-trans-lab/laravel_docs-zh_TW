# 升級指南

- [從 11.x 升級到 12.0](#upgrade-12.0)

<a name="high-impact-changes"></a>
## 高影響變更

<div class="content-list" markdown="1">

- [更新依賴套件](#updating-dependencies)
- [更新 Laravel 安裝器](#updating-the-laravel-installer)

</div>


<a name="medium-impact-changes"></a>
## 中等影響變更

<div class="content-list" markdown="1">

- [模型與 UUIDv7](#models-and-uuidv7)

</div>


<a name="low-impact-changes"></a>
## 低影響變更

<div class="content-list" markdown="1">

- [Carbon 3](#carbon-3)
- [並行結果索引對映](#concurrency-result-index-mapping)
- [容器類別依賴解析](#container-class-dependency-resolution)
- [圖片驗證現在排除 SVG](#image-validation)
- [本地檔案系統磁碟的預設根路徑](#local-filesystem-disk-default-root-path)
- [多 Schema 資料庫檢查](#multi-schema-database-inspecting)
- [巢狀陣列請求合併](#nested-array-request-merging)

</div>


<a name="upgrade-12.0"></a>
## 從 11.x 升級到 12.0


#### 預計升級時間：5 分鐘

> [!NOTE]
> 我們試圖記錄每個可能的破壞性變更。由於其中一些變更位於框架中較不常見的部分，因此這些變更中只有一部分可能會實際影響您的應用程式。想要節省時間嗎？您可以使用 [Laravel Shift](https://laravelshift.com/) 來幫助自動化您的應用程式升級。


<a name="updating-dependencies"></a>
### 更新依賴套件

**影響可能性：高**

您應該更新應用程式 `composer.json` 檔案中的以下依賴套件：

<div class="content-list" markdown="1">

- 將 `laravel/framework` 更新至 `^12.0`
- 將 `phpunit/phpunit` 更新至 `^11.0`
- 將 `pestphp/pest` 更新至 `^3.0`

</div>


<a name="carbon-3"></a>
#### Carbon 3

**影響可能性：低**

已移除對 [Carbon 2.x](https://carbon.nesbot.com/docs/) 的支援。所有 Laravel 12 應用程式現在都要求 [Carbon 3.x](https://carbon.nesbot.com/docs/#api-carbon-3)。


<a name="updating-the-laravel-installer"></a>
### 更新 Laravel 安裝器

如果您使用 Laravel installer CLI 工具來建立新的 Laravel 應用程式，您應該更新您的安裝器安裝，以使其與 Laravel 12.x 和 [新的 Laravel starter kits](https://laravel.com/starter-kits) 相容。如果您透過 `composer global require` 安裝了 Laravel installer，您可以使用 `composer global update` 來更新安裝器：

```shell
composer global update laravel/installer
```

如果您最初透過 `php.new` 安裝了 PHP 和 Laravel，您可以簡單地重新執行適用於您作業系統的 `php.new` 安裝命令，以安裝最新版本的 PHP 和 Laravel installer：

```shell tab=macOS
/bin/bash -c "$(curl -fsSL https://php.new/install/mac/8.4)"
```

```shell tab=Windows PowerShell
# Run as administrator...
Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://php.new/install/windows/8.4'))
```

```shell tab=Linux
/bin/bash -c "$(curl -fsSL https://php.new/install/linux/8.4)"
```

或者，如果您正在使用 [Laravel Herd](https://herd.laravel.com) 隨附的 Laravel 安裝程式，您應該將您的 Herd 安裝更新到最新版本。


<a name="authentication"></a>
### 驗證 (Authentication)


<a name="updated-databasetokenrepository-constructor-signature"></a>
#### 更新了 `DatabaseTokenRepository` 建構函式簽章

**影響可能性：非常低**

`Illuminate\Auth\Passwords\DatabaseTokenRepository` 類別的建構函式現在預期 `$expires` 參數以秒為單位給定，而不是分鐘。


<a name="concurrency"></a>
### 並發 (Concurrency)


<a name="concurrency-result-index-mapping"></a>
#### 並發結果索引映射

**影響可能性：低**

當使用關聯式陣列呼叫 `Concurrency::run` 方法時，並發操作的結果現在會連同其關聯的鍵一起返回：

```php
$result = Concurrency::run([
    'task-1' => fn () => 1 + 1,
    'task-2' => fn () => 2 + 2,
]);

// ['task-1' => 2, 'task-2' => 4]
```


<a name="container"></a>
### 容器 (Container)


<a name="container-class-dependency-resolution"></a>
#### 容器類別依賴解析

**影響可能性：低**

依賴注入容器現在在解析類別實例時會遵循類別屬性的預設值。如果您先前依賴容器在沒有預設值的情況下解析類別實例，您可能需要調整您的應用程式以適應此新行為：

```php
class Example
{
    public function __construct(public ?Carbon $date = null) {}
}

$example = resolve(Example::class);

// <= 11.x
$example->date instanceof Carbon;

// >= 12.x
$example->date === null;
```


<a name="database"></a>
### 資料庫 (Database)


<a name="multi-schema-database-inspecting"></a>
#### 多重綱要資料庫檢查

**影響可能性：低**

`Schema::getTables()`、`Schema::getViews()` 和 `Schema::getTypes()` 方法現在預設會包含所有綱要的結果。您可以傳遞 `schema` 參數來僅取得指定綱要的結果：

```php
// All tables on all schemas...
$tables = Schema::getTables();

// All tables on the 'main' schema...
$tables = Schema::getTables(schema: 'main');

// All tables on the 'main' and 'blog' schemas...
$tables = Schema::getTables(schema: ['main', 'blog']);
```

`Schema::getTableListing()` 方法現在預設會返回帶有綱要限定的表格名稱。您可以傳遞 `schemaQualified` 參數來根據需要更改行為：

```php
$tables = Schema::getTableListing();
// ['main.migrations', 'main.users', 'blog.posts']

$tables = Schema::getTableListing(schema: 'main');
// ['main.migrations', 'main.users']

$tables = Schema::getTableListing(schema: 'main', schemaQualified: false);
// ['migrations', 'users']
```

`db:table` 和 `db:show` 命令現在會像 PostgreSQL 和 SQL Server 一樣，在 MySQL、MariaDB 和 SQLite 上輸出所有綱要的結果。


<a name="updated-blueprint-constructor-signature"></a>
#### 更新了 `Blueprint` 建構函式簽章

**影響可能性：非常低**

`Illuminate\Database\Schema\Blueprint` 類別的建構函式現在預期其第一個參數為 `Illuminate\Database\Connection` 的實例。


<a name="eloquent"></a>
### Eloquent


<a name="models-and-uuidv7"></a>
#### 模型與 UUIDv7

**影響可能性：中**

`HasUuids` trait 現在會回傳與 UUID 規範第 7 版（有序 UUID）相容的 UUID。如果您希望繼續使用有序的 UUIDv4 字串作為模型的 ID，您現在應該使用 `HasVersion4Uuids` trait：

```php
use Illuminate\Database\Eloquent\Concerns\HasUuids; // [tl! remove]
use Illuminate\Database\Eloquent\Concerns\HasVersion4Uuids as HasUuids; // [tl! add]
```

`HasVersion7Uuids` trait 已被移除。如果您先前使用此 trait，您現在應該改用 `HasUuids` trait，它現在提供相同的行為。


<a name="requests"></a>
### 請求 (Requests)


<a name="nested-array-request-merging"></a>
#### 巢狀陣列請求合併

**影響可能性：低**

`$request->mergeIfMissing()` 方法現在允許使用「點」符號合併巢狀陣列資料。如果您先前依賴此方法來建立包含「點」符號版本鍵值的頂層陣列鍵，您可能需要調整您的應用程式以適應此新行為：

```php
$request->mergeIfMissing([
    'user.last_name' => 'Otwell',
]);
```


<a name="storage"></a>
### 儲存 (Storage)


<a name="local-filesystem-disk-default-root-path"></a>
#### 本地檔案系統磁碟預設根路徑

**影響可能性：低**

如果您的應用程式未在檔案系統設定中明確定義 `local` 磁碟，Laravel 現在會將本地磁碟的根路徑預設為 `storage/app/private`。在以前的版本中，這預設為 `storage/app`。因此，除非另行配置，對 `Storage::disk('local')` 的呼叫將會從 `storage/app/private` 讀取和寫入。要恢復先前的行為，您可以手動定義 `local` 磁碟並設定所需的根路徑。


<a name="validation"></a>
### 驗證 (Validation)


<a name="image-validation"></a>
#### 圖片驗證現在排除 SVG

**影響可能性：低**

`image` 驗證規則現在預設不再允許 SVG 圖片。如果您希望在使用 `image` 規則時允許 SVG，您必須明確地允許它們：

```php
use Illuminate\Validation\Rules\File;

'photo' => 'required|image:allow_svg'

// Or...
'photo' => ['required', File::image(allowSvg: true)],
```


<a name="miscellaneous"></a>
### 其他

我們也鼓勵您查看 `laravel/laravel` [GitHub 儲存庫](https://github.com/laravel/laravel)中的變更。雖然其中許多變更不是強制性的，但您可能希望讓這些檔案與您的應用程式保持同步。這些變更中的一些將在此升級指南中涵蓋，但其他變更，例如設定檔或註解的變更，則不會涵蓋。您可以使用 [GitHub 比較工具](https://github.com/laravel/laravel/compare/11.x...12.x)輕鬆查看這些變更，並選擇對您而言重要的更新。