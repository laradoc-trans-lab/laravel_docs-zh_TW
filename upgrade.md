# 升級指南

- [從 11.x 升級至 12.0](#upgrade-12.0)

<a name="high-impact-changes"></a>
## 高影響變更

<div class="content-list" markdown="1">

- [更新相依套件](#updating-dependencies)
- [更新 Laravel 安裝程式](#updating-the-laravel-installer)

</div>


<a name="medium-impact-changes"></a>
## 中影響變更

<div class="content-list" markdown="1">

- [模型與 UUIDv7](#models-and-uuidv7)

</div>


<a name="low-impact-changes"></a>
## 低影響變更

<div class="content-list" markdown="1">

- [Carbon 3](#carbon-3)
- [並行結果索引映射](#concurrency-result-index-mapping)
- [容器類別相依性解析](#container-class-dependency-resolution)
- [圖片驗證現在排除 SVG](#image-validation)
- [本地檔案系統磁碟的預設根路徑](#local-filesystem-disk-default-root-path)
- [多 Schema 資料庫檢查](#multi-schema-database-inspecting)
- [巢狀陣列請求合併](#nested-array-request-merging)

</div>

<a name="upgrade-12.0"></a>
## 從 11.x 升級至 12.0


#### 預計升級時間：5 分鐘

> [!NOTE]
> 我們嘗試記錄每一個可能的破壞性變更 (Breaking Change)。由於其中一些破壞性變更位於框架中較為冷門的部分，實際上只有一部分變更會影響到你的應用程式。想節省時間嗎？你可以使用 [Laravel Shift](https://laravelshift.com/) 來幫助自動化你的應用程式升級。


<a name="updating-dependencies"></a>
### 更新依賴項

**影響可能性：高**

你應該更新應用程式 `composer.json` 檔案中的下列依賴項：

<div class="content-list" markdown="1">

- `laravel/framework` 至 `^12.0`
- `phpunit/phpunit` 至 `^11.0`
- `pestphp/pest` 至 `^3.0`

</div>


<a name="carbon-3"></a>
#### Carbon 3

**影響可能性：低**

已移除對 Carbon 2.x 的支援。所有 Laravel 12 應用程式現在都需要 [Carbon 3.x](https://carbon.nesbot.com/guide/getting-started/migration.html)。


<a name="updating-the-laravel-installer"></a>
### 更新 Laravel 安裝程式

如果你正在使用 Laravel 安裝程式 CLI 工具來建立新的 Laravel 應用程式，你應該更新你的安裝程式版本，以相容於 Laravel 12.x 和 [新的 Laravel starter kits](https://laravel.com/starter-kits)。如果你是透過 `composer global require` 安裝 Laravel 安裝程式，你可以使用 `composer global update` 來更新：

```shell
composer global update laravel/installer
```

如果你最初是透過 `php.new` 安裝 PHP 和 Laravel，你只需針對你的作業系統重新執行 `php.new` 安裝指令，即可安裝最新版本的 PHP 和 Laravel 安裝程式：

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

或者，如果你使用的是 [Laravel Herd](https://herd.laravel.com) 內建的 Laravel 安裝程式，你應該將 Herd 更新至最新版本。


<a name="authentication"></a>
### 認證


<a name="updated-databasetokenrepository-constructor-signature"></a>
#### 更新 `DatabaseTokenRepository` 建構子簽章

**影響可能性：極低**

`Illuminate\Auth\Passwords\DatabaseTokenRepository` 類別的建構子現在預期 `$expires` 參數以秒為單位，而非分鐘。


<a name="concurrency"></a>
### 並行處理 (Concurrency)


<a name="concurrency-result-index-mapping"></a>
#### 並行結果索引映射

**影響可能性：低**

當使用關聯陣列呼叫 `Concurrency::run` 方法時，並行操作的結果現在將連同其關聯的鍵值一併回傳：

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
#### 容器類別相依解析

**影響可能性：低**

相依注入容器現在在解析類別實例時會遵循類別屬性的預設值。如果你以前依賴容器在忽略預設值的情況下解析類別實例，你可能需要調整你的應用程式以適應此新行為：

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

`Schema::getTables()`、`Schema::getViews()` 和 `Schema::getTypes()` 方法現在預設會包含所有 Schema 的結果。你可以傳遞 `schema` 引數來僅取得指定 Schema 的結果：

```php
// All tables on all schemas...
$tables = Schema::getTables();

// All tables on the 'main' schema...
$tables = Schema::getTables(schema: 'main');

// All tables on the 'main' and 'blog' schemas...
$tables = Schema::getTables(schema: ['main', 'blog']);
```

`Schema::getTableListing()` 方法現在預設會回傳包含 Schema 名稱的資料表名稱。你可以傳遞 `schemaQualified` 引數來根據需求更改此行為：

```php
$tables = Schema::getTableListing();
// ['main.migrations', 'main.users', 'blog.posts']

$tables = Schema::getTableListing(schema: 'main');
// ['main.migrations', 'main.users']

$tables = Schema::getTableListing(schema: 'main', schemaQualified: false);
// ['migrations', 'users']
```

`db:table` 和 `db:show` 指令現在會在 MySQL、MariaDB 和 SQLite 上輸出所有 Schema 的結果，就像 PostgreSQL 和 SQL Server 一樣。


<a name="database-constructor-signature-changes"></a>
#### 資料庫建構子簽章變更

**影響可能性：極低**

在 Laravel 12 中，多個底層資料庫類別現在要求透過其建構子提供一個 `Illuminate\Database\Connection` 實例。

**這些變更主要適用於資料庫套件維護者 —— 這些變更極不可能影響一般的應用程式開發。**

`Illuminate\Database\Schema\Blueprint`

`Illuminate\Database\Schema\Blueprint` 類別的建構子現在預期一個 `Connection` 實例作為其第一個引數。這主要影響手動實例化 `Blueprint` 實例的應用程式或套件。

`Illuminate\Database\Grammar`

`Illuminate\Database\Grammar` 類別的建構子現在也需要一個 `Connection` 實例。在以前的版本中，連線是在建構後使用 `setConnection()` 方法分配的。此方法已在 Laravel 12 中移除：

```php
// Laravel <= 11.x
$grammar = new MySqlGrammar;
$grammar->setConnection($connection);

// Laravel >= 12.x
$grammar = new MySqlGrammar($connection);
````

此外，以下 API 已被移除或棄用：

<div class="content-list" markdown="1">

- `Blueprint::getPrefix()` 方法已棄用。
- `Connection::withTablePrefix()` 方法已被移除。
- `Grammar::getTablePrefix()` 和 `setTablePrefix()` 方法已棄用。
- `Grammar::setConnection()` 方法已被移除。

</div>

當處理資料表前綴時，你現在應該直接從資料庫連線中取得它們：

```php
$prefix = $connection->getTablePrefix();
```

如果你維護自定義資料庫驅動程式、Schema 構建器或 Grammar 實作，你應該檢查它們的建構子並確保提供了一個 `Connection` 實例。


<a name="eloquent"></a>
### Eloquent


<a name="models-and-uuidv7"></a>
#### 模型與 UUIDv7

**影響可能性：中**

`HasUuids` trait 現在會回傳與 UUID spec 版本 7 相容的 UUID（有序 UUID）。如果你想繼續為模型的 ID 使用有序的 UUIDv4 字串，你現在應該使用 `HasVersion4Uuids` trait：

```php
use Illuminate\Database\Eloquent\Concerns\HasUuids; // [tl! remove]
use Illuminate\Database\Eloquent\Concerns\HasVersion4Uuids as HasUuids; // [tl! add]
```

`HasVersion7Uuids` trait 已被移除。如果你之前使用過這個 trait，你現在應該改用 `HasUuids` trait，它現在提供了相同的行為。

<a name="requests"></a>
### Requests


<a name="nested-array-request-merging"></a>
#### 巢狀陣列請求合併 (Nested Array Request Merging)

**影響程度：低**

`$request->mergeIfMissing()` 方法現在允許使用「點 (dot)」記法來合併巢狀陣列資料。如果您之前依賴此方法來建立一個包含「點」記法鍵名的頂層陣列鍵，您可能需要調整您的應用程式以因應此新行為：

```php
$request->mergeIfMissing([
    'user.last_name' => 'Otwell',
]);
```


<a name="routing"></a>
### Routing


<a name="route-precedence"></a>
#### 路由優先順序 (Route Precedence)

**影響程度：低**

當多個路由具有相同名稱時，快取與未快取路由之間的路由行為已進行統一。這意味著未快取的路由現在會比對第一個以該名稱註冊的路由，而非最後一個。


<a name="storage"></a>
### Storage


<a name="local-filesystem-disk-default-root-path"></a>
#### 本地檔案系統磁碟的預設根目錄路徑

**影響程度：低**

如果您的應用程式未在檔案系統設定中明確定義 `local` 磁碟，Laravel 現在將預設將本地磁碟的根目錄設為 `storage/app/private`。在先前的版本中，此預設值為 `storage/app`。因此，除非另有設定，否則對 `Storage::disk('local')` 的呼叫將會讀取與寫入至 `storage/app/private`。若要恢復先前的行為，您可以手動定義 `local` 磁碟並設定所需的根目錄路徑。


<a name="validation"></a>
### Validation


<a name="image-validation"></a>
#### 圖片驗證現在排除 SVG

**影響程度：低**

預設情況下，`image` 驗證規則不再允許 SVG 圖片。如果您想在使用的 `image` 規則時允許 SVG，您必須明確地允許它們：

```php
use Illuminate\Validation\Rules\File;

'photo' => 'required|image:allow_svg'

// Or...
'photo' => ['required', File::image(allowSvg: true)],
```


<a name="miscellaneous"></a>
### 其他 (Miscellaneous)

我們也鼓勵您查看 `laravel/laravel` [GitHub 存放庫](https://github.com/laravel/laravel) 中的變更。雖然其中許多變更並非必要，但您可能希望讓這些檔案與您的應用程式保持同步。其中一些變更將在本升級指南中涵蓋，但其他變更（例如設定檔或註解的變更）則不會。您可以使用 [GitHub 比較工具](https://github.com/laravel/laravel/compare/11.x...12.x) 輕鬆查看變更，並選擇對您重要的更新。