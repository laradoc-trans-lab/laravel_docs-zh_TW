# 發行說明

- [版本控制方案](#versioning-scheme)
- [支援政策](#support-policy)
- [Laravel 11](#laravel-11)

<a name="versioning-scheme"></a>
## 版本控制方案

Laravel 及其其他第一方套件遵循 [語義化版本控制](https://semver.org)。主要框架發行版每年發布一次 (約第一季)，而次要發行版和修補程式發行版可能每週發布。次要發行版和修補程式發行版**絕不**應包含破壞性變更。

當從您的應用程式或套件中引用 Laravel 框架或其元件時，您應始終使用版本限制，例如 `^11.0`，因為 Laravel 的主要發行版確實包含破壞性變更。然而，我們始終努力確保您可以在一天或更短的時間內更新到新的主要發行版。

<a name="named-arguments"></a>
#### 命名引數

[命名引數](https://www.php.net/manual/en/functions.arguments.php#functions.named-arguments) 不受 Laravel 向後相容性準則的約束。我們可能會在必要時重新命名函式引數，以改善 Laravel 程式碼庫。因此，在呼叫 Laravel 方法時使用命名引數應謹慎行事，並且理解參數名稱將來可能會改變。

<a name="support-policy"></a>
## 支援政策

所有 Laravel 發行版都提供 18 個月的錯誤修正和 2 年的安全修正。對於所有其他函式庫，包括 Lumen，只有最新的主要發行版會收到錯誤修正。此外，請查閱 [Laravel 支援的資料庫版本](/docs/{{version}}/database#introduction)。

<div class="overflow-auto">

| 版本 | PHP (*) | 發行 | 錯誤修正直到 | 安全修正直到 |
| --- | --- | --- | --- | --- |
| 9 | 8.0 - 8.2 | February 8th, 2022 | August 8th, 2023 | February 6th, 2024 |
| 10 | 8.1 - 8.3 | February 14th, 2023 | August 6th, 2024 | February 4th, 2025 |
| 11 | 8.2 - 8.4 | March 12th, 2024 | September 3rd, 2025 | March 12th, 2026 |
| 12 | 8.2 - 8.4 | February 24th, 2025 | August 13th, 2026 | February 24th, 2027 |

</div>

<div class="version-colors">
    <div class="end-of-life">
        <div class="color-box"></div>
        <div>終止支援</div>
    </div>
    <div class="security-fixes">
        <div class="color-box"></div>
        <div>僅提供安全修正</div>
    </div>
</div>

(*) 支援的 PHP 版本

<a name="laravel-11"></a>
## Laravel 11

Laravel 11 延續了 Laravel 10.x 所做的改進，引入了簡化的應用程式結構、每秒速率限制、健康路由、優雅的加密金鑰輪替、佇列測試改進、[Resend](https://resend.com) 郵件傳輸、Prompt 驗證器整合、新的 Artisan 指令等功能。此外，還引入了 Laravel Reverb，一個第一方的、可擴展的 WebSocket 伺服器，為您的應用程式提供強大的即時功能。


<a name="php-8"></a>
### PHP 8.2

Laravel 11.x 需要 PHP 最低版本為 8.2。


<a name="structure"></a>
### 簡化的應用程式結構

_Laravel 的簡化應用程式結構由 [Taylor Otwell](https://github.com/taylorotwell) 和 [Nuno Maduro](https://github.com/nunomaduro) 開發。_

Laravel 11 為**新的** Laravel 應用程式引入了簡化的應用程式結構，而無需對現有應用程式進行任何更改。新的應用程式結構旨在提供更精簡、更現代的體驗，同時保留 Laravel 開發者已經熟悉的許多概念。以下我們將討論 Laravel 新應用程式結構的重點。


#### 應用程式啟動檔案

`bootstrap/app.php` 檔案已作為一個程式碼優先的應用程式設定檔而重獲新生。從這個檔案中，您現在可以自訂應用程式的路由、中介層、服務提供者、例外處理等等。這個檔案統一了以前分散在應用程式檔案結構中的各種高層級應用程式行為設定：

```php
return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web: __DIR__.'/../routes/web.php',
        commands: __DIR__.'/../routes/console.php',
        health: '/up',
    )
    ->withMiddleware(function (Middleware $middleware) {
        //
    })
    ->withExceptions(function (Exceptions $exceptions) {
        //
    })->create();
```


<a name="service-providers"></a>
#### 服務提供者

Laravel 11 並不像預設的 Laravel 應用程式結構那樣包含五個服務提供者，而只包含單一的 `AppServiceProvider`。先前服務提供者的功能已整合到 `bootstrap/app.php` 中，由框架自動處理，或可以放置在您的應用程式的 `AppServiceProvider` 中。

例如，事件探索現在預設啟用，這在很大程度上消除了手動註冊事件及其監聽器的需要。但是，如果您確實需要手動註冊事件，您可以直接在 `AppServiceProvider` 中完成。同樣，您以前可能在 `AuthServiceProvider` 中註冊的路由模型綁定或授權 Gates 也可以在 `AppServiceProvider` 中註冊。


<a name="opt-in-routing"></a>
#### 選用 API 和廣播路由

`api.php` 和 `channels.php` 路由檔案預設不再存在，因為許多應用程式不需要這些檔案。相反，它們可以使用簡單的 Artisan 指令來創建：

```shell
php artisan install:api

php artisan install:broadcasting
```


<a name="middleware"></a>
#### 中介層

以前，新的 Laravel 應用程式包含九個中介層。這些中介層執行各種任務，例如驗證請求、修剪輸入字串以及驗證 CSRF 令牌。

在 Laravel 11 中，這些中介層已移入框架本身，因此它們不會增加應用程式結構的負擔。框架中增加了用於自訂這些中介層行為的新方法，並且可以從應用程式的 `bootstrap/app.php` 檔案中調用：

```php
->withMiddleware(function (Middleware $middleware) {
    $middleware->validateCsrfTokens(
        except: ['stripe/*']
    );

    $middleware->web(append: [
        EnsureUserIsSubscribed::class,
    ])
})
```

由於所有中介層都可以透過應用程式的 `bootstrap/app.php` 輕鬆自訂，因此不再需要單獨的 HTTP「核心 (kernel)」類別。


<a name="scheduling"></a>
#### 排程

使用新的 `Schedule` Facade，排程任務現在可以直接在應用程式的 `routes/console.php` 檔案中定義，從而不再需要單獨的主控台「核心 (kernel)」類別：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('emails:send')->daily();
```


<a name="exception-handling"></a>
#### 例外處理

與路由和中介層一樣，例外處理現在可以從應用程式的 `bootstrap/app.php` 檔案中自訂，而不是透過單獨的例外處理器類別，從而減少了新的 Laravel 應用程式中包含的檔案總數：

```php
->withExceptions(function (Exceptions $exceptions) {
    $exceptions->dontReport(MissedFlightException::class);

    $exceptions->report(function (InvalidOrderException $e) {
        // ...
    });
})
```


<a name="base-controller-class"></a>
#### 基礎 `Controller` 類別

新的 Laravel 應用程式中包含的基礎控制器已簡化。它不再繼承 Laravel 的內部 `Controller` 類別，並且已刪除 `AuthorizesRequests` 和 `ValidatesRequests` 特徵 (traits)，因為如果需要，它們可以包含在應用程式的個別控制器中：

    <?php

    namespace App\Http\Controllers;

    abstract class Controller
    {
        //
    }


<a name="application-defaults"></a>
#### 應用程式預設值

預設情況下，新的 Laravel 應用程式使用 SQLite 進行資料庫儲存，並使用 `database` 驅動程式用於 Laravel 的 Session、快取和佇列。這讓您在創建新的 Laravel 應用程式後可以立即開始構建應用程式，而無需安裝額外的軟體或創建額外的資料庫遷移。

此外，隨著時間的推移，這些 Laravel 服務的 `database` 驅動程式在許多應用程式場景中已足夠強大，可供生產使用；因此，它們為本地和生產應用程式提供了一個合理、統一的選擇。


<a name="reverb"></a>
### Laravel Reverb

_Laravel Reverb 由 [Joe Dixon](https://github.com/joedixon) 開發。_

[Laravel Reverb](https://reverb.laravel.com) 將極速且可擴展的即時 WebSocket 通訊直接帶到您的 Laravel 應用程式中，並提供與 Laravel 現有的事件廣播工具（例如 Laravel Echo）的無縫整合。

```shell
php artisan reverb:start
```

此外，Reverb 透過 Redis 的發布/訂閱功能支援水平擴展，讓您能夠將 WebSocket 流量分佈到多個後端 Reverb 伺服器，所有這些伺服器都支援單一、高需求的應用程式。

有關 Laravel Reverb 的更多資訊，請查閱完整的 [Reverb 文件](/docs/{{version}}/reverb)。


<a name="rate-limiting"></a>
### 每秒速率限制

_每秒速率限制由 [Tim MacDonald](https://github.com/timacdonald) 貢獻。_

Laravel 現在支援所有速率限制器（包括 HTTP 請求和佇列工作）的「每秒」速率限制。以前，Laravel 的速率限制器僅限於「每分鐘」的粒度：

```php
RateLimiter::for('invoices', function (Request $request) {
    return Limit::perSecond(1);
});
```

有關 Laravel 中速率限制的更多資訊，請查看 [速率限制文件](/docs/{{version}}/routing#rate-limiting)。


<a name="health"></a>
### 健康路由

_健康路由由 [Taylor Otwell](https://github.com/taylorotwell) 貢獻。_

新的 Laravel 11 應用程式包含一個 `health` 路由指令，它指示 Laravel 定義一個簡單的健康檢查端點，該端點可以由第三方應用程式健康監控服務或 Kubernetes 等協調系統調用。預設情況下，此路由在 `/up` 處提供服務：

```php
->withRouting(
    web: __DIR__.'/../routes/web.php',
    commands: __DIR__.'/../routes/console.php',
    health: '/up',
)
```

當 HTTP 請求發送到此路由時，Laravel 還會分派一個 `DiagnosingHealth` 事件，讓您可以執行與應用程式相關的其他健康檢查。

<a name="encryption"></a>
### 優雅的加密金鑰輪換

_優雅的加密金鑰輪換功能由 [Taylor Otwell](https://github.com/taylorotwell) 貢獻_。

由於 Laravel 會加密所有 Cookie，包括您應用程式的 session Cookie，因此 Laravel 應用程式的每個請求基本上都依賴於加密。然而，正因如此，輪換應用程式的加密金鑰會導致所有使用者從您的應用程式中登出。此外，解密由舊加密金鑰加密的資料也將變得不可能。

Laravel 11 允許您透過 `APP_PREVIOUS_KEYS` 環境變數，將應用程式的舊加密金鑰定義為逗號分隔的列表。

當加密值時，Laravel 將始終使用 `APP_KEY` 環境變數中的「當前」加密金鑰。當解密值時，Laravel 將首先嘗試當前金鑰。如果使用當前金鑰解密失敗，Laravel 將嘗試所有舊金鑰，直到其中一個金鑰能夠解密該值。

這種優雅的解密方法允許使用者即使在加密金鑰輪換後，也能不間斷地繼續使用您的應用程式。

有關 Laravel 中加密的更多資訊，請查閱[加密文件](/docs/{{version}}/encryption)。

<a name="automatic-password-rehashing"></a>
### 自動重新雜湊密碼

_自動重新雜湊密碼功能由 [Stephen Rees-Carter](https://github.com/valorin) 貢獻_。

Laravel 的預設密碼雜湊演算法是 bcrypt。bcrypt 雜湊的「工作係數 (work factor)」可以透過 `config/hashing.php` 設定檔或 `BCRYPT_ROUNDS` 環境變數進行調整。

通常，隨著 CPU / GPU 處理能力的增加，bcrypt 的工作係數也應隨時間提高。如果您提高應用程式的 bcrypt 工作係數，Laravel 現在將在使用者驗證時，優雅地自動重新雜湊使用者的密碼。

<a name="prompt-validation"></a>
### Prompt 驗證

_Prompt 驗證器整合功能由 [Andrea Marco Sartori](https://github.com/cerbero90) 貢獻_。

[Laravel Prompts](/docs/{{version}}/prompts) 是一個 PHP 封裝，用於為您的命令列應用程式添加美觀且使用者友善的表單，具有類似瀏覽器的功能，包括提示文字和驗證。

Laravel Prompts 支援透過閉包進行輸入驗證：

```php
$name = text(
    label: 'What is your name?',
    validate: fn (string $value) => match (true) {
        strlen($value) < 3 => 'The name must be at least 3 characters.',
        strlen($value) > 255 => 'The name must not exceed 255 characters.',
        default => null
    }
);
```

然而，當處理多個輸入或複雜的驗證情境時，這可能會變得繁瑣。因此，在 Laravel 11 中，您可以在驗證 Prompt 輸入時，利用 Laravel [驗證器](/docs/{{version}}/validation) 的全部功能：

```php
$name = text('What is your name?', validate: [
    'name' => 'required|min:3|max:255',
]);
```

<a name="queue-interaction-testing"></a>
### Queue 互動測試

_Queue 互動測試功能由 [Taylor Otwell](https://github.com/taylorotwell) 貢獻_。

過去，嘗試測試佇列中的 job 是否被釋放、刪除或手動失敗是一項繁瑣的工作，需要定義自訂的 queue fakes 和 stubs。然而，在 Laravel 11 中，您可以輕鬆地使用 `withFakeQueueInteractions` 方法測試這些 queue 互動：

```php
use App\Jobs\ProcessPodcast;

$job = (new ProcessPodcast)->withFakeQueueInteractions();

$job->handle();

$job->assertReleased(delay: 30);
```

有關測試佇列 job 的更多資訊，請查閱 [queue 文件](/docs/{{version}}/queues#testing)。

<a name="new-artisan-commands"></a>
### 新的 Artisan 指令

_類別建立 Artisan 指令由 [Taylor Otwell](https://github.com/taylorotwell) 貢獻_。

新增了 Artisan 指令，以允許快速建立類別 (classes)、列舉 (enums)、介面 (interfaces) 和特性 (traits)：

```shell
php artisan make:class
php artisan make:enum
php artisan make:interface
php artisan make:trait
```

<a name="model-cast-improvements"></a>
### Model Casts 改善

_Model casts 改善功能由 [Nuno Maduro](https://github.com/nunomaduro) 貢獻_。

Laravel 11 支援使用方法而非屬性來定義 model 的 casts。這使得 casts 的定義更加精簡流暢，尤其是在使用帶有引數的 casts 時：

    /**
     * Get the attributes that should be cast.
     *
     * @return array<string, string>
     */
    protected function casts(): array
    {
        return [
            'options' => AsCollection::using(OptionCollection::class),
                      // AsEncryptedCollection::using(OptionCollection::class),
                      // AsEnumArrayObject::using(OptionEnum::class),
                      // AsEnumCollection::using(OptionEnum::class),
        ];
    }

有關屬性 casting 的更多資訊，請查閱 [Eloquent 文件](/docs/{{version}}/eloquent-mutators#attribute-casting)。

<a name="the-once-function"></a>
### `once` 函式

_`once` 輔助函式由 [Taylor Otwell](https://github.com/taylorotwell) 與 [Nuno Maduro](https://github.com/nunomaduro) 貢獻_。

`once` 輔助函式會執行給定的 callback，並在請求期間將結果快取在記憶體中。隨後對 `once` 函式使用相同 callback 的任何呼叫都將返回先前快取的結果：

    function random(): int
    {
        return once(function () {
            return random_int(1, 1000);
        });
    }

    random(); // 123
    random(); // 123 (cached result)
    random(); // 123 (cached result)

有關 `once` 輔助函式的更多資訊，請查閱 [輔助函式文件](/docs/{{version}}/helpers#method-once)。

<a name="database-performance"></a>
### 使用記憶體內資料庫測試時的效能提升

_記憶體內資料庫測試效能改善功能由 [Anders Jenbo](https://github.com/AJenbo) 貢獻_。

在測試期間使用 `:memory:` SQLite 資料庫時，Laravel 11 提供了顯著的速度提升。為此，Laravel 現在維護著一個對 PHP 的 PDO 物件的引用，並在連接之間重複使用它，通常能將總測試執行時間縮短一半。

<a name="mariadb"></a>
### 改善對 MariaDB 的支援

_改善對 MariaDB 的支援功能由 [Jonas Staudenmeir](https://github.com/staudenmeir) 與 [Julius Kiekbusch](https://github.com/Jubeki) 貢獻_。

Laravel 11 包含了對 MariaDB 的改善支援。在之前的 Laravel 版本中，您可以透過 Laravel 的 MySQL 驅動程式使用 MariaDB。然而，Laravel 11 現在包含了一個專用的 MariaDB 驅動程式，它為這個資料庫系統提供了更好的預設值。

有關 Laravel 資料庫驅動程式的更多資訊，請查閱 [資料庫文件](/docs/{{version}}/database)。

<a name="inspecting-database"></a>
### 檢查資料庫和改進的結構描述操作

_改進的結構描述操作和資料庫檢查功能由 [Hafez Divandari](https://github.com/hafezdivandari) 貢獻_。

Laravel 11 提供了額外的資料庫結構描述操作和檢查方法，包括原生修改、重新命名和刪除欄位。此外，還提供了進階空間類型、非預設結構描述名稱，以及用於操作資料表、視圖、欄位、索引和外部索引鍵的原生結構描述方法：

    use Illuminate\Support\Facades\Schema;

    $tables = Schema::getTables();
    $views = Schema::getViews();
    $columns = Schema::getColumns('users');
    $indexes = Schema::getIndexes('users');
    $foreignKeys = Schema::getForeignKeys('users');