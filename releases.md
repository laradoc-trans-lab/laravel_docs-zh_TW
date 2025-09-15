# 版本說明

- [版本控制方案](#versioning-scheme)
- [支援政策](#support-policy)
- [Laravel 11](#laravel-11)

<a name="versioning-scheme"></a>
## 版本控制方案

Laravel 及其其他第一方套件遵循 [語義化版本控制](https://semver.org)。主要框架版本每年發布一次 (約第一季)，而次要版本與修補版本則可能每週發布。次要版本與修補版本**絕不**包含破壞性變更。

當從您的應用程式或套件中引用 Laravel 框架或其元件時，您應該始終使用版本限制，例如 `^11.0`，因為 Laravel 的主要版本確實包含破壞性變更。然而，我們努力確保您始終可以在一天或更短的時間內更新到新的主要版本。

<a name="named-arguments"></a>
#### 具名引數

[具名引數](https://www.php.net/manual/en/functions.arguments.php#functions.named-arguments) 不受 Laravel 向下相容性準則的涵蓋。我們可能會在必要時重新命名函式引數，以改進 Laravel 程式碼庫。因此，在呼叫 Laravel 方法時使用具名引數應謹慎為之，並理解參數名稱未來可能會變更。

<a name="support-policy"></a>
## 支援政策

對於所有 Laravel 版本，錯誤修復提供 18 個月，安全修復提供 2 年。對於所有額外函式庫，包括 Lumen，只有最新的主要版本會收到錯誤修復。此外，請查閱 [Laravel 支援的資料庫版本](/docs/{{version}}/database#introduction)。

<div class="overflow-auto">

| 版本 | PHP (*) | 發布日期 | 錯誤修復至 | 安全修復至 |
| --- | --- | --- | --- | --- |
| 9 | 8.0 - 8.2 | 2022 年 2 月 8 日 | 2023 年 8 月 8 日 | 2024 年 2 月 6 日 |
| 10 | 8.1 - 8.3 | 2023 年 2 月 14 日 | 2024 年 8 月 6 日 | 2025 年 2 月 4 日 |
| 11 | 8.2 - 8.4 | 2024 年 3 月 12 日 | 2025 年 9 月 3 日 | 2026 年 3 月 12 日 |
| 12 | 8.2 - 8.4 | 2025 年 2 月 24 日 | 2026 年 8 月 13 日 | 2027 年 2 月 24 日 |

</div>

<div class="version-colors">
    <div class="end-of-life">
        <div class="color-box"></div>
        <div>終止支援</div>
    </div>
    <div class="security-fixes">
        <div class="color-box"></div>
        <div>僅提供安全修復</div>
    </div>
</div>

(*) 支援的 PHP 版本

<a name="laravel-11"></a>
## Laravel 11

Laravel 11 延續了 Laravel 10.x 的改進，引入了精簡的應用程式結構、每秒速率限制、健康檢查路由、優雅的加密金鑰輪替、Queue 測試改進、[Resend](https://resend.com) 郵件傳輸、Prompt 驗證器整合、新的 Artisan 命令等等。此外，還引入了 Laravel Reverb，一個第一方、可擴展的 WebSocket 伺服器，為您的應用程式提供強大的即時功能。

<a name="php-8"></a>
### PHP 8.2

Laravel 11.x 需要最低 PHP 版本為 8.2。

<a name="structure"></a>
### 精簡的應用程式結構

_Laravel 的精簡應用程式結構由 [Taylor Otwell](https://github.com/taylorotwell) 和 [Nuno Maduro](https://github.com/nunomaduro) 開發。_

Laravel 11 為**新的** Laravel 應用程式引入了精簡的應用程式結構，而無需對現有應用程式進行任何更改。新的應用程式結構旨在提供更精簡、更現代的體驗，同時保留許多 Laravel 開發人員已經熟悉的觀念。下面我們將討論 Laravel 新應用程式結構的重點。

#### 應用程式啟動檔案

`bootstrap/app.php` 檔案已重新設計為程式碼優先的應用程式設定檔。從這個檔案中，您現在可以自訂應用程式的路由、Middleware、服務提供者、異常處理等等。這個檔案統一了以前分散在應用程式檔案結構中的各種高階應用程式行為設定：

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

Laravel 11 不再包含預設 Laravel 應用程式結構中的五個服務提供者，而只包含一個 `AppServiceProvider`。先前服務提供者的功能已整合到 `bootstrap/app.php` 中，由框架自動處理，或者可以放置在應用程式的 `AppServiceProvider` 中。

例如，事件發現現在預設啟用，這在很大程度上消除了手動註冊事件及其監聽器的需要。但是，如果您確實需要手動註冊事件，您只需在 `AppServiceProvider` 中進行即可。同樣，您可能先前在 `AuthServiceProvider` 中註冊的路由 Model 綁定或授權 Gates 也可以在 `AppServiceProvider` 中註冊。

<a name="opt-in-routing"></a>
#### 選用 API 與廣播路由

`api.php` 和 `channels.php` 路由檔案預設不再存在，因為許多應用程式不需要這些檔案。相反，它們可以使用簡單的 Artisan 命令創建：

```shell
php artisan install:api

php artisan install:broadcasting
```

<a name="middleware"></a>
#### Middleware

以前，新的 Laravel 應用程式包含九個 Middleware。這些 Middleware 執行各種任務，例如驗證請求、修剪輸入字串和驗證 CSRF 權杖。

在 Laravel 11 中，這些 Middleware 已移至框架本身，因此它們不會增加應用程式結構的負擔。用於自訂這些 Middleware 行為的新方法已添加到框架中，並可以從應用程式的 `bootstrap/app.php` 檔案中調用：

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

由於所有 Middleware 都可以透過應用程式的 `bootstrap/app.php` 輕鬆自訂，因此不再需要單獨的 HTTP 「Kernel」類別。

<a name="scheduling"></a>
#### 排程

使用新的 `Schedule` Facade，排程任務現在可以直接在應用程式的 `routes/console.php` 檔案中定義，從而無需單獨的 Console 「Kernel」類別：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('emails:send')->daily();
```

<a name="exception-handling"></a>
#### 異常處理

與路由和 Middleware 一樣，異常處理現在可以從應用程式的 `bootstrap/app.php` 檔案中自訂，而不是單獨的異常處理類別，從而減少了新 Laravel 應用程式中包含的檔案總數：

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

新 Laravel 應用程式中包含的基礎 Controller 已簡化。它不再擴展 Laravel 的內部 `Controller` 類別，並且 `AuthorizesRequests` 和 `ValidatesRequests` Traits 已被移除，因為如果需要，它們可以包含在應用程式的個別 Controller 中：

    <?php

    namespace App\Http\Controllers;

    abstract class Controller
    {
        //
    }

<a name="application-defaults"></a>
#### 應用程式預設值

預設情況下，新的 Laravel 應用程式使用 SQLite 進行資料庫儲存，以及 `database` 驅動程式用於 Laravel 的 Session、Cache 和 Queue。這讓您可以在創建新的 Laravel 應用程式後立即開始建構應用程式，而無需安裝額外的軟體或創建額外的資料庫 Migrations。

此外，隨著時間的推移，這些 Laravel 服務的 `database` 驅動程式在許多應用程式情境中已足夠強大以用於生產環境；因此，它們為本地和生產應用程式提供了合理、統一的選擇。

<a name="reverb"></a>
### Laravel Reverb

_Laravel Reverb 由 [Joe Dixon](https://github.com/joedixon) 開發。_

[Laravel Reverb](https://reverb.laravel.com) 將極速且可擴展的即時 WebSocket 通訊直接帶到您的 Laravel 應用程式，並與 Laravel 現有的事件廣播工具套件 (例如 Laravel Echo) 提供無縫整合。

```shell
php artisan reverb:start
```

此外，Reverb 透過 Redis 的發布/訂閱功能支援水平擴展，讓您可以將 WebSocket 流量分佈到多個後端 Reverb 伺服器，所有這些伺服器都支援單一、高需求的應用程式。

有關 Laravel Reverb 的更多資訊，請查閱完整的 [Reverb 說明文件](/docs/{{version}}/reverb)。

<a name="rate-limiting"></a>
### 每秒速率限制

_每秒速率限制由 [Tim MacDonald](https://github.com/timacdonald) 貢獻。_

Laravel 現在支援所有速率限制器（包括 HTTP 請求和 Queue Job 的速率限制器）的「每秒」速率限制。以前，Laravel 的速率限制器僅限於「每分鐘」的粒度：

```php
RateLimiter::for('invoices', function (Request $request) {
    return Limit::perSecond(1);
});
```

有關 Laravel 中速率限制的更多資訊，請查閱 [速率限制說明文件](/docs/{{version}}/routing#rate-limiting)。

<a name="health"></a>
### 健康檢查路由

_健康檢查路由由 [Taylor Otwell](https://github.com/taylorotwell) 貢獻。_

新的 Laravel 11 應用程式包含一個 `health` 路由指令，它指示 Laravel 定義一個簡單的健康檢查端點，該端點可以由第三方應用程式健康監控服務或 Kubernetes 等協調系統調用。預設情況下，此路由在 `/up` 提供服務：

```php
->withRouting(
    web: __DIR__.'/../routes/web.php',
    commands: __DIR__.'/../routes/console.php',
    health: '/up',
)
```

當對此路由發出 HTTP 請求時，Laravel 還會分派一個 `DiagnosingHealth` 事件，讓您可以執行與應用程式相關的其他健康檢查。

<a name="encryption"></a>
### 優雅的加密金鑰輪替

_優雅的加密金鑰輪替由 [Taylor Otwell](https://github.com/taylorotwell) 貢獻。_

由於 Laravel 會加密所有 Cookie，包括應用程式的 Session Cookie，因此 Laravel 應用程式的每個請求基本上都依賴於加密。然而，正因為如此，輪替應用程式的加密金鑰會將所有使用者登出應用程式。此外，解密由先前加密金鑰加密的資料也變得不可能。

Laravel 11 允許您透過 `APP_PREVIOUS_KEYS` 環境變數將應用程式的先前加密金鑰定義為逗號分隔的列表。

加密值時，Laravel 將始終使用「當前」加密金鑰，該金鑰位於 `APP_KEY` 環境變數中。解密值時，Laravel 將首先嘗試當前金鑰。如果使用當前金鑰解密失敗，Laravel 將嘗試所有先前金鑰，直到其中一個金鑰能夠解密該值。

這種優雅的解密方法允許使用者即使在加密金鑰輪替後也能不間斷地繼續使用您的應用程式。

有關 Laravel 中加密的更多資訊，請查閱 [加密說明文件](/docs/{{version}}/encryption)。

<a name="automatic-password-rehashing"></a>
### 自動密碼重新雜湊

_自動密碼重新雜湊由 [Stephen Rees-Carter](https://github.com/valorin) 貢獻。_

Laravel 的預設密碼雜湊演算法是 bcrypt。bcrypt 雜湊的「工作因子」可以透過 `config/hashing.php` 設定檔或 `BCRYPT_ROUNDS` 環境變數進行調整。

通常，隨著 CPU / GPU 處理能力的增加，bcrypt 工作因子應隨時間增加。如果您增加應用程式的 bcrypt 工作因子，Laravel 現在將在使用者驗證應用程式時優雅且自動地重新雜湊使用者密碼。

<a name="prompt-validation"></a>
### Prompt 驗證

_Prompt 驗證器整合由 [Andrea Marco Sartori](https://github.com/cerbero90) 貢獻。_

[Laravel Prompts](/docs/{{version}}/prompts) 是一個 PHP 套件，用於為您的命令列應用程式添加美觀且使用者友好的表單，具有瀏覽器般的功能，包括佔位符文字和驗證。

Laravel Prompts 透過閉包支援輸入驗證：

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

然而，當處理許多輸入或複雜的驗證情境時，這可能會變得繁瑣。因此，在 Laravel 11 中，您可以在驗證 Prompt 輸入時利用 Laravel [驗證器](/docs/{{version}}/validation) 的全部功能：

```php
$name = text('What is your name?', validate: [
    'name' => 'required|min:3|max:255',
]);
```

<a name="queue-interaction-testing"></a>
### Queue 互動測試

_Queue 互動測試由 [Taylor Otwell](https://github.com/taylorotwell) 貢獻。_

以前，嘗試測試 Queue Job 是否被釋放、刪除或手動失敗是繁瑣的，並且需要定義自訂 Queue Fakes 和 Stubs。然而，在 Laravel 11 中，您可以使用 `withFakeQueueInteractions` 方法輕鬆測試這些 Queue 互動：

```php
use App\Jobs\ProcessPodcast;

$job = (new ProcessPodcast)->withFakeQueueInteractions();

$job->handle();

$job->assertReleased(delay: 30);
```

有關測試 Queue Job 的更多資訊，請查閱 [Queue 說明文件](/docs/{{version}}/queues#testing)。

<a name="new-artisan-commands"></a>
### 新的 Artisan 命令

_類別創建 Artisan 命令由 [Taylor Otwell](https://github.com/taylorotwell) 貢獻。_

已添加新的 Artisan 命令，以允許快速創建類別、Enums、介面和 Traits：

```shell
php artisan make:class
php artisan make:enum
php artisan make:interface
php artisan make:trait
```

<a name="model-cast-improvements"></a>
### Model Casts 改進

_Model Casts 改進由 [Nuno Maduro](https://github.com/nunomaduro) 貢獻。_

Laravel 11 支援使用方法而不是屬性來定義 Model 的 Casts。這允許精簡、流暢的 Cast 定義，尤其是在使用帶有引數的 Casts 時：

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

有關屬性 Cast 的更多資訊，請查閱 [Eloquent 說明文件](/docs/{{version}}/eloquent-mutators#attribute-casting)。

<a name="the-once-function"></a>
### `once` 函式

_`once` 輔助函式由 [Taylor Otwell](https://github.com/taylorotwell) 和 _[Nuno Maduro](https://github.com/nunomaduro) 貢獻。_

`once` 輔助函式執行給定的回呼並在請求期間將結果快取在記憶體中。隨後對具有相同回呼的 `once` 函式的所有呼叫都將返回先前快取的結果：

    function random(): int
    {
        return once(function () {
            return random_int(1, 1000);
        });
    }

    random(); // 123
    random(); // 123 (cached result)
    random(); // 123 (cached result)

有關 `once` 輔助函式的更多資訊，請查閱 [輔助函式說明文件](/docs/{{version}}/helpers#method-once)。

<a name="database-performance"></a>
### 使用記憶體內資料庫測試時的效能改進

_記憶體內資料庫測試效能改進由 [Anders Jenbo](https://github.com/AJenbo) 貢獻。_

Laravel 11 在測試期間使用 `:memory:` SQLite 資料庫時提供了顯著的速度提升。為此，Laravel 現在維護對 PHP PDO 物件的引用，並在連接之間重複使用它，通常將總測試運行時間縮短一半。

<a name="mariadb"></a>
### 改進對 MariaDB 的支援

_改進對 MariaDB 的支援由 [Jonas Staudenmeir](https://github.com/staudenmeir) 和 [Julius Kiekbusch](https://github.com/Jubeki) 貢獻。_

Laravel 11 包含對 MariaDB 的改進支援。在以前的 Laravel 版本中，您可以透過 Laravel 的 MySQL 驅動程式使用 MariaDB。然而，Laravel 11 現在包含一個專用的 MariaDB 驅動程式，它為此資料庫系統提供了更好的預設值。

有關 Laravel 資料庫驅動程式的更多資訊，請查閱 [資料庫說明文件](/docs/{{version}}/database)。

<a name="inspecting-database"></a>
### 資料庫檢查與改進的 Schema 操作

_改進的 Schema 操作和資料庫檢查由 [Hafez Divandari](https://github.com/hafezdivandari) 貢獻。_

Laravel 11 提供了額外的資料庫 Schema 操作和檢查方法，包括原生修改、重新命名和刪除欄位。此外，還提供了進階空間類型、非預設 Schema 名稱以及用於操作資料表、視圖、欄位、索引和外部鍵的原生 Schema 方法：

    use Illuminate\Support\Facades\Schema;

    $tables = Schema::getTables();
    $views = Schema::getViews();
    $columns = Schema::getColumns('users');
    $indexes = Schema::getIndexes('users');
    $foreignKeys = Schema::getForeignKeys('users');
