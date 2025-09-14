# 升級指南

- [從 11.x 升級到 12.0](#upgrade-12.0)

<a name="high-impact-changes"></a>
## 高影響變更

<div class="content-list" markdown="1">

- [更新依賴套件](#updating-dependencies)
- [更新 Laravel 安裝程式](#updating-the-laravel-installer)

</div>

<a name="medium-impact-changes"></a>
## 中等影響變更

<div class="content-list" markdown="1">

- [Models 與 UUIDv7](#models-and-uuidv7)

</div>

<a name="low-impact-changes"></a>
## 低影響變更

<div class="content-list" markdown="1">

- [Carbon 3](#carbon-3)
- [並行結果索引映射](#concurrency-result-index-mapping)
- [Container 類別依賴解析](#container-class-dependency-resolution)
- [圖片驗證現在排除 SVG](#image-validation)
- [本地檔案系統磁碟預設根路徑](#local-filesystem-disk-default-root-path)
- [多 Schema 資料庫檢查](#multi-schema-database-inspecting)
- [巢狀陣列請求合併](#nested-array-request-merging)

</div>

<a name="upgrade-12.0"></a>
## 從 11.x 升級到 12.0

#### 預計升級時間：5 分鐘

> [!NOTE]
> 我們試圖記錄所有可能的破壞性變更。由於其中一些破壞性變更位於框架中較為隱晦的部分，因此這些變更中只有一部分可能實際影響您的應用程式。想要節省時間嗎？您可以使用 [Laravel Shift](https://laravelshift.com/) 來幫助自動化您的應用程式升級。

<a name="updating-dependencies"></a>
### 更新依賴套件

**影響可能性：高**

您應該更新應用程式 `composer.json` 檔案中的以下依賴套件：

<div class="content-list" markdown="1">

- `laravel/framework` 到 `^12.0`
- `phpunit/phpunit` 到 `^11.0`
- `pestphp/pest` 到 `^3.0`

</div>

<a name="carbon-3"></a>
#### Carbon 3

**影響可能性：低**

已移除對 [Carbon 2.x](https://carbon.nesbot.com/docs/) 的支援。所有 Laravel 12 應用程式現在都要求 [Carbon 3.x](https://carbon.nesbot.com/docs/#api-carbon-3)。

<a name="updating-the-laravel-installer"></a>
### 更新 Laravel 安裝程式

如果您使用 Laravel 安裝程式 CLI 工具來建立新的 Laravel 應用程式，您應該更新您的安裝程式以與 Laravel 12.x 和 [新的 Laravel 啟動套件](https://laravel.com/starter-kits) 相容。如果您透過 `composer global require` 安裝了 Laravel 安裝程式，您可以使用 `composer global update` 更新安裝程式：

```shell
composer global update laravel/installer
```

如果您最初是透過 `php.new` 安裝 PHP 和 Laravel，您可以簡單地重新執行適用於您作業系統的 `php.new` 安裝命令，以安裝最新版本的 PHP 和 Laravel 安裝程式：

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

或者，如果您使用 [Laravel Herd](https://herd.laravel.com) 捆綁的 Laravel 安裝程式副本，您應該將您的 Herd 安裝更新到最新版本。

<a name="authentication"></a>
### 認證

<a name="updated-databasetokenrepository-constructor-signature"></a>
#### `DatabaseTokenRepository` 建構子簽章更新

**影響可能性：非常低**

`Illuminate\Auth\Passwords\DatabaseTokenRepository` 類別的建構子現在預期 `$expires` 參數以秒為單位提供，而不是分鐘。

<a name="concurrency"></a>
### 並行

<a name="concurrency-result-index-mapping"></a>
#### 並行結果索引映射

**影響可能性：低**

當使用關聯陣列呼叫 `Concurrency::run` 方法時，並行操作的結果現在會以其關聯的鍵返回：

```php
$result = Concurrency::run([
    'task-1' => fn () => 1 + 1,
    'task-2' => fn () => 2 + 2,
]);

// ['task-1' => 2, 'task-2' => 4]
```

<a name="container"></a>
### Container

<a name="container-class-dependency-resolution"></a>
#### Container 類別依賴解析

**影響可能性：低**

依賴注入 Container 現在在解析類別實例時會尊重類別屬性的預設值。如果您之前依賴 Container 在沒有預設值的情況下解析類別實例，您可能需要調整您的應用程式以適應此新行為：

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
### 資料庫

<a name="multi-schema-database-inspecting"></a>
#### 多 Schema 資料庫檢查

**影響可能性：低**

`Schema::getTables()`、`Schema::getViews()` 和 `Schema::getTypes()` 方法現在預設包含所有 Schema 的結果。您可以傳遞 `schema` 參數以僅檢索給定 Schema 的結果：

```php
// 所有 Schema 上的所有資料表...
$tables = Schema::getTables();

// 'main' Schema 上的所有資料表...
$tables = Schema::getTables(schema: 'main');

// 'main' 和 'blog' Schema 上的所有資料表...
$tables = Schema::getTables(schema: ['main', 'blog']);
```

`Schema::getTableListing()` 方法現在預設返回帶有 Schema 限定的資料表名稱。您可以傳遞 `schemaQualified` 參數以根據需要更改行為：

```php
$tables = Schema::getTableListing();
// ['main.migrations', 'main.users', 'blog.posts']

$tables = Schema::getTableListing(schema: 'main');
// ['main.migrations', 'main.users']

$tables = Schema::getTableListing(schema: 'main', schemaQualified: false);
// ['migrations', 'users']
```

`db:table` 和 `db:show` 命令現在在 MySQL、MariaDB 和 SQLite 上輸出所有 Schema 的結果，就像 PostgreSQL 和 SQL Server 一樣。

<a name="updated-blueprint-constructor-signature"></a>
#### `Blueprint` 建構子簽章更新

**影響可能性：非常低**

`Illuminate\Database\Schema\Blueprint` 類別的建構子現在預期 `Illuminate\Database\Connection` 的實例作為其第一個參數。

<a name="eloquent"></a>
### Eloquent

<a name="models-and-uuidv7"></a>
#### Models 與 UUIDv7

**影響可能性：中**

`HasUuids` Trait 現在返回與 UUID 規範版本 7 (有序 UUID) 相容的 UUID。如果您想繼續為您的 Model ID 使用有序的 UUIDv4 字串，您現在應該使用 `HasVersion4Uuids` Trait：

```php
use Illuminate\Database\Eloquent\Concerns\HasUuids; // [tl! remove]
use Illuminate\Database\Eloquent\Concerns\HasVersion4Uuids as HasUuids; // [tl! add]
```

`HasVersion7Uuids` Trait 已被移除。如果您之前使用此 Trait，您應該改用 `HasUuids` Trait，它現在提供相同的行為。

<a name="requests"></a>
### 請求

<a name="nested-array-request-merging"></a>
#### 巢狀陣列請求合併

**影響可能性：低**

`$request->mergeIfMissing()` 方法現在允許使用「點」表示法合併巢狀陣列資料。如果您之前依賴此方法來建立包含「點」表示法鍵的頂層陣列鍵，您可能需要調整您的應用程式以適應此新行為：

```php
$request->mergeIfMissing([
    'user.last_name' => 'Otwell',
]);
```

<a name="storage"></a>
### 儲存

<a name="local-filesystem-disk-default-root-path"></a>
#### 本地檔案系統磁碟預設根路徑

**影響可能性：低**

如果您的應用程式未在檔案系統設定中明確定義 `local` 磁碟，Laravel 現在會將本地磁碟的根目錄預設為 `storage/app/private`。在以前的版本中，這預設為 `storage/app`。因此，對 `Storage::disk('local')` 的呼叫將從 `storage/app/private` 讀取和寫入，除非另有配置。要恢復以前的行為，您可以手動定義 `local` 磁碟並設定所需的根路徑。

<a name="validation"></a>
### 驗證

<a name="image-validation"></a>
#### 圖片驗證現在排除 SVG

**影響可能性：低**

`image` 驗證規則不再預設允許 SVG 圖片。如果您想在使用 `image` 規則時允許 SVG，您必須明確允許它們：

```php
use Illuminate\Validation\Rules\File;

'photo' => 'required|image:allow_svg'

// Or...
'photo' => ['required', File::image(allowSvg: true)],
```

<a name="miscellaneous"></a>
### 其他

我們也鼓勵您查看 `laravel/laravel` [GitHub 儲存庫](https://github.com/laravel/laravel) 中的變更。雖然其中許多變更不是必需的，但您可能希望使這些檔案與您的應用程式保持同步。其中一些變更將在本升級指南中涵蓋，但其他變更，例如對設定檔或註解的變更，則不會。您可以使用 [GitHub 比較工具](https://github.com/laravel/laravel/compare/11.x...12.x) 輕鬆查看變更，並選擇對您重要的更新。
