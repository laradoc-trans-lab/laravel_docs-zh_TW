# 升級指南

- [從 11.x 升級至 12.0](#upgrade-12.0)

<a name="high-impact-changes"></a>
## 高影響的變更

<div class="content-list" markdown="1">

- [更新依賴項目](#updating-dependencies)
- [更新 Laravel 安裝程式](#updating-the-laravel-installer)

</div>


<a name="medium-impact-changes"></a>
## 中影響的變更

<div class="content-list" markdown="1">

- [模型與 UUIDv7](#models-and-uuidv7)

</div>


<a name="low-impact-changes"></a>
## 低影響的變更

<div class="content-list" markdown="1">

- [Carbon 3](#carbon-3)
- [Concurrency 結果索引對應](#concurrency-result-index-mapping)
- [容器類別依賴解析](#container-class-dependency-resolution)
- [圖片驗證現在排除 SVG](#image-validation)
- [本地檔案系統磁碟的預設根目錄路徑](#local-filesystem-disk-default-root-path)
- [多 Schema 資料庫檢查](#multi-schema-database-inspecting)
- [巢狀陣列請求合併](#nested-array-request-merging)

</div>

<a name="upgrade-12.0"></a>
## 從 11.x 升級至 12.0


#### 預計升級時間：5 分鐘

> [!NOTE]
> 我們嘗試記錄每一個可能的破壞性變更。由於其中一些破壞性變更位於框架中較為冷門的部分，因此實際上只有一部分變更會影響您的應用程式。想節省時間嗎？您可以使用 [Laravel Shift](https://laravelshift.com/) 來幫助自動化您的應用程式升級。


<a name="updating-dependencies"></a>
### 更新依賴項目

**影響程度：高**

您應該更新應用程式中 `composer.json` 檔案的以下依賴項目：

<div class="content-list" markdown="1">

- `laravel/framework` 至 `^12.0`
- `phpunit/phpunit` 至 `^11.0`
- `pestphp/pest` 至 `^3.0`

</div>


<a name="carbon-3"></a>
#### Carbon 3

**影響程度：低**

已移除對 Carbon 2.x 的支援。所有 Laravel 12 應用程式現在都需要 [Carbon 3.x](https://carbon.nesbot.com/guide/getting-started/migration.html)。


<a name="updating-the-laravel-installer"></a>
### 更新 Laravel 安裝器

如果您使用 Laravel 安裝器 CLI 工具來建立新的 Laravel 應用程式，則應更新您的安裝器安裝版本，以相容於 Laravel 12.x 以及 [新的 Laravel 入門套件 (Starter Kits)](https://laravel.com/starter-kits)。如果您是透過 `composer global require` 安裝 Laravel 安裝器，可以使用 `composer global update` 來更新安裝器：

```shell
composer global update laravel/installer
```

如果您最初是透過 `php.new` 安裝 PHP 和 Laravel，則只需為您的作業系統重新執行 `php.new` 安裝指令，即可安裝最新版本的 PHP 和 Laravel 安裝器：

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

或者，如果您使用的是 [Laravel Herd](https://herd.laravel.com) 內建的 Laravel 安裝器複本，則應將您的 Herd 更新至最新版本。


<a name="authentication"></a>
### 身份驗證


<a name="updated-databasetokenrepository-constructor-signature"></a>
#### 更新 `DatabaseTokenRepository` 建構子簽署

**影響程度：極低**

`Illuminate\Auth\Passwords\DatabaseTokenRepository` 類別的建構子現在預期傳入的 `$expires` 參數是以秒為單位，而非分鐘。


<a name="concurrency"></a>
### 並行 (Concurrency)


<a name="concurrency-result-index-mapping"></a>
#### 並行結果索引映射

**影響程度：低**

當使用關聯陣列呼叫 `Concurrency::run` 方法時，並行操作的結果現在會連同其關聯的鍵 (Key) 一併回傳：

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

**影響程度：低**

依賴注入容器在解析類別實例時，現在會遵循類別屬性的預設值。如果您先前依賴容器在解析類別實例時忽略預設值，您可能需要調整您的應用程式以因應此新行為：

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
#### 多結構集 (Multi-Schema) 資料庫檢查

**影響程度：低**

`Schema::getTables()`、`Schema::getViews()` 與 `Schema::getTypes()` 方法現在預設會包含來自所有結構集 (Schema) 的結果。您可以傳遞 `schema` 引數來僅取得指定結構集的結果：

```php
// All tables on all schemas...
$tables = Schema::getTables();

// All tables on the 'main' schema...
$tables = Schema::getTables(schema: 'main');

// All tables on the 'main' and 'blog' schemas...
$tables = Schema::getTables(schema: ['main', 'blog']);
```

`Schema::getTableListing()` 方法現在預設會回傳包含結構集限定的資料表名稱。您可以傳遞 `schemaQualified` 引數來根據需求更改此行為：

```php
$tables = Schema::getTableListing();
// ['main.migrations', 'main.users', 'blog.posts']

$tables = Schema::getTableListing(schema: 'main');
// ['main.migrations', 'main.users']

$tables = Schema::getTableListing(schema: 'main', schemaQualified: false);
// ['migrations', 'users']
```

`db:table` 和 `db:show` 指令現在也會在 MySQL、MariaDB 和 SQLite 上輸出所有結構集的結果，就如同 PostgreSQL 和 SQL Server 一樣。


<a name="database-constructor-signature-changes"></a>
#### 資料庫建構子簽署變更

**影響程度：極低**

在 Laravel 12 中，數個底層資料庫類別現在需要透過其建構子提供 `Illuminate\Database\Connection` 實例。

**這些變更主要適用於資料庫套件維護者 —— 這些變更極不可能影響一般的應用程式開發。**

`Illuminate\Database\Schema\Blueprint`

`Illuminate\Database\Schema\Blueprint` 類別的建構子現在預期將 `Connection` 實例作為其第一個引數。這主要影響手動實例化 `Blueprint` 實例的應用程式或套件。

`Illuminate\Database\Grammar`

`Illuminate\Database\Grammar` 類別的建構子現在也需要一個 `Connection` 實例。在先前的版本中，連線是在建構後使用 `setConnection()` 方法分配的。此方法已在 Laravel 12 中移除：

```php
// Laravel <= 11.x
$grammar = new MySqlGrammar;
$grammar->setConnection($connection);

// Laravel >= 12.x
$grammar = new MySqlGrammar($connection);
```

此外，以下 API 已被移除或棄用：

<div class="content-list" markdown="1">

- `Blueprint::getPrefix()` 方法已棄用。
- `Connection::withTablePrefix()` 方法已移除。
- `Grammar::getTablePrefix()` 與 `setTablePrefix()` 方法已棄用。
- `Grammar::setConnection()` 方法已移除。

</div>

在使用資料表前綴 (Table Prefix) 時，您現在應該直接從資料庫連線中取得它們：

```php
$prefix = $connection->getTablePrefix();
```

如果您維護自定義的資料庫驅動程式、結構生成器 (Schema Builder) 或語法 (Grammar) 實作，您應該檢查其建構子並確保提供了 `Connection` 實例。


<a name="eloquent"></a>
### Eloquent


<a name="models-and-uuidv7"></a>
#### 模型與 UUIDv7

**影響程度：中**

`HasUuids` trait 現在會回傳與 UUID 規範版本 7 相容的 UUID (有序的 UUID)。如果您希望繼續為您的模型 ID 使用有序的 UUIDv4 字串，您現在應該使用 `HasVersion4Uuids` trait：

```php
use Illuminate\Database\Eloquent\Concerns\HasUuids; // [tl! remove]
use Illuminate\Database\Eloquent\Concerns\HasVersion4Uuids as HasUuids; // [tl! add]
```

`HasVersion7Uuids` trait 已被移除。如果您先前使用此 trait，現在應該改用 `HasUuids` trait，它現在提供了相同的行為。

<a name="requests"></a>
### Requests


<a name="nested-array-request-merging"></a>
#### 巢狀陣列請求合併

**影響可能性：低**

`$request->mergeIfMissing()` 方法現在允許使用「點 (dot)」記法來合併巢狀陣列資料。如果您先前依賴此方法來建立一個包含該鍵名「點」記法版本的頂層陣列鍵名，則可能需要調整您的應用程式以因應此新行為：

```php
$request->mergeIfMissing([
    'user.last_name' => 'Otwell',
]);
```


<a name="storage"></a>
### Storage


<a name="local-filesystem-disk-default-root-path"></a>
#### 本地檔案系統磁碟的預設根目錄路徑

**影響可能性：低**

如果您的應用程式未在檔案系統設定中明確定義 `local` 磁碟，Laravel 現在會將本地磁碟的根目錄預設為 `storage/app/private`。在先前的版本中，此路徑預設為 `storage/app`。因此，除非另有設定，否則對 `Storage::disk('local')` 的呼叫將會從 `storage/app/private` 讀取或寫入。若要恢復先前的行為，您可以手動定義 `local` 磁碟並設定所需的根目錄路徑。


<a name="validation"></a>
### Validation


<a name="image-validation"></a>
#### 圖片驗證現在排除 SVG

**影響可能性：低**

`image` 驗證規則預設不再允許 SVG 圖片。如果您希望在使用 `image` 規則時允許 SVG，必須明確地允許它們：

```php
use Illuminate\Validation\Rules\File;

'photo' => 'required|image:allow_svg'

// Or...
'photo' => ['required', File::image(allowSvg: true)],
```


<a name="miscellaneous"></a>
### 雜項

我們也鼓勵您查看 `laravel/laravel` [GitHub 儲存庫](https://github.com/laravel/laravel) 中的變更。雖然其中許多變更並非必要，但您可能希望讓這些檔案與您的應用程式保持同步。本升級指南將涵蓋其中的部分變更，但其他變更（例如對設定檔或註解的變更）則不會涵蓋。您可以使用 [GitHub 比較工具](https://github.com/laravel/laravel/compare/11.x...12.x) 輕鬆查看變更，並選擇對您重要的更新。