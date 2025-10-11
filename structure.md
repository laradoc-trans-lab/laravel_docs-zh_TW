# 目錄結構

- [簡介](#introduction)
- [根目錄](#the-root-directory)
    - [`app` 目錄](#the-root-app-directory)
    - [`bootstrap` 目錄](#the-bootstrap-directory)
    - [`config` 目錄](#the-config-directory)
    - [`database` 目錄](#the-database-directory)
    - [`public` 目錄](#the-public-directory)
    - [`resources` 目錄](#the-resources-directory)
    - [`routes` 目錄](#the-routes-directory)
    - [`storage` 目錄](#the-storage-directory)
    - [`tests` 目錄](#the-tests-directory)
    - [`vendor` 目錄](#the-vendor-directory)
- [App 目錄](#the-app-directory)
    - [`Broadcasting` 目錄](#the-broadcasting-directory)
    - [`Console` 目錄](#the-console-directory)
    - [`Events` 目錄](#the-events-directory)
    - [`Exceptions` 目錄](#the-exceptions-directory)
    - [`Http` 目錄](#the-http-directory)
    - [`Jobs` 目錄](#the-jobs-directory)
    - [`Listeners` 目錄](#the-listeners-directory)
    - [`Mail` 目錄](#the-mail-directory)
    - [`Models` 目錄](#the-models-directory)
    - [`Notifications` 目錄](#the-notifications-directory)
    - [`Policies` 目錄](#the-policies-directory)
    - [`Providers` 目錄](#the-providers-directory)
    - [`Rules` 目錄](#the-rules-directory)

<a name="introduction"></a>
## 簡介

預設的 Laravel 應用程式結構旨在為大型和小型應用程式提供一個良好的起點。但你可以自由地組織你的應用程式，只要你喜歡。Laravel 對於任何特定類別的位置幾乎沒有限制 — 只要 Composer 能夠自動載入該類別。

> [!NOTE]  
> Laravel 新手？請查看 [Laravel Bootcamp](https://bootcamp.laravel.com) 以親身體驗此框架，我們將引導你建立第一個 Laravel 應用程式。

<a name="the-root-directory"></a>
## 根目錄

<a name="the-root-app-directory"></a>
### `app` 目錄

app 目錄包含你應用程式的核心程式碼。我們很快就會更詳細地探討此目錄；然而，你應用程式中幾乎所有的類別都將位於此目錄中。

<a name="the-bootstrap-directory"></a>
### `bootstrap` 目錄

bootstrap 目錄包含 `app.php` 檔案，該檔案用於啟動 (bootstraps) 框架。此目錄還包含一個 `cache` 目錄，其中包含框架為效能優化而生成的檔案，例如路由和服務快取檔案。

<a name="the-config-directory"></a>
### `config` 目錄

config 目錄，顧名思義，包含你應用程式的所有設定檔。建議閱讀所有這些檔案，熟悉所有可用的選項。

<a name="the-database-directory"></a>
### `database` 目錄

database 目錄包含你的資料庫遷移、模型工廠和填充檔 (seeds)。如果你願意，你也可以使用此目錄來存放一個 SQLite 資料庫。

<a name="the-public-directory"></a>
### `public` 目錄

public 目錄包含 `index.php` 檔案，這是所有進入你應用程式的請求的入口點，並配置自動載入。此目錄也存放你的資源 (assets)，例如圖片、JavaScript 和 CSS。

<a name="the-resources-directory"></a>
### `resources` 目錄

resources 目錄包含你的 [視圖](/docs/{{version}}/views) 以及未經編譯的原始資源 (assets)，例如 CSS 或 JavaScript。

<a name="the-routes-directory"></a>
### `routes` 目錄

routes 目錄包含你應用程式的所有路由定義。預設情況下，Laravel 包含兩個路由檔案：`web.php` 和 `console.php`。

`web.php` 檔案包含 Laravel 放置在 `web` 中介層群組中的路由，該群組提供會話狀態、CSRF 保護和 Cookie 加密。如果你的應用程式不提供無狀態的 RESTful API，那麼你所有的路由很可能都會在 `web.php` 檔案中定義。

`console.php` 檔案是你定義所有基於閉包 (closure) 的 Artisan 指令的地方。每個閉包都綁定到一個指令實例，允許以簡單的方法與每個指令的 IO 方法互動。儘管此檔案不定義 HTTP 路由，但它定義了基於控制台的入口點 (路由) 到你的應用程式。你也可以在 `console.php` 檔案中 [排程](/docs/{{version}}/scheduling) 任務。

你可以選擇透過 `install:api` 和 `install:broadcasting` Artisan 指令，為 API 路由 (`api.php`) 和廣播頻道 (`channels.php`) 安裝額外的路由檔案。

`api.php` 檔案包含旨在為無狀態的路由，因此透過這些路由進入應用程式的請求預期將會 [透過 Token](/docs/{{version}}/sanctum) 進行驗證，並且無法存取會話狀態。

`channels.php` 檔案是你註冊你應用程式所支援的所有 [事件廣播](/docs/{{version}}/broadcasting) 頻道的地方。

<a name="the-storage-directory"></a>
### `storage` 目錄

storage 目錄包含你的日誌、編譯後的 Blade 模板、基於檔案的會話、檔案快取以及框架生成的其他檔案。此目錄被劃分為 `app`、`framework` 和 `logs` 目錄。`app` 目錄可以用於儲存你的應用程式生成的任何檔案。`framework` 目錄用於儲存框架生成的檔案和快取。最後，`logs` 目錄包含你應用程式的日誌檔案。

`storage/app/public` 目錄可用於儲存使用者生成的檔案，例如個人資料圖片 (profile avatars)，這些檔案應該是公開可存取的。你應該在 `public/storage` 建立一個指向此目錄的符號連結。你可以使用 `php artisan storage:link` Artisan 指令來建立此連結。

<a name="the-tests-directory"></a>
### `tests` 目錄

tests 目錄包含你的自動化測試。開箱即用地提供了 [Pest](https://pestphp.com) 或 [PHPUnit](https://phpunit.de/) 單元測試和功能測試範例。每個測試類別都應以 `Test` 一詞作為後綴。你可以使用 `/vendor/bin/pest` 或 `/vendor/bin/phpunit` 指令來執行你的測試。或者，如果你想要更詳細和美觀的測試結果呈現，你可以使用 `php artisan test` Artisan 指令來執行你的測試。

<a name="the-vendor-directory"></a>
### `vendor` 目錄

vendor 目錄包含你的 [Composer](https://getcomposer.org) 依賴項。

<a name="the-app-directory"></a>
## App 目錄

您應用程式的大部分程式碼都存放在 `app` 目錄中。依預設，此目錄命名空間為 `App`，並由 Composer 使用 [PSR-4 自動載入標準](https://www.php-fig.org/psr/psr-4/)進行自動載入。

依預設，`app` 目錄包含 `Http`、`Models` 和 `Providers` 目錄。然而，隨著時間推移，當您使用 make Artisan 命令生成類別時，app 目錄內會生成各種其他目錄。例如，`app/Console` 目錄在您執行 `make:command` Artisan 命令以生成命令類別之前是不存在的。

`Console` 和 `Http` 這兩個目錄在下方各自的章節中會進一步說明，但您可以將 `Console` 和 `Http` 目錄視為提供應用程式核心的 API。HTTP 協定和 CLI 都是與應用程式互動的機制，但它們本身不包含應用程式邏輯。換句話說，它們是向應用程式發出命令的兩種方式。`Console` 目錄包含所有您的 Artisan 命令，而 `Http` 目錄則包含您的控制器、中介層和請求。

> [!NOTE]  
> `app` 目錄中的許多類別都可以透過 Artisan 命令生成。要查看可用的命令，請在終端機中執行 `php artisan list make` 命令。


<a name="the-broadcasting-directory"></a>
### `Broadcasting` 目錄

Broadcasting 目錄包含應用程式中所有的廣播頻道類別。這些類別是使用 `make:channel` 命令生成的。此目錄預設不存在，但當您創建第一個頻道時，它將會為您創建。要了解更多關於頻道的資訊，請參閱 [事件廣播](/docs/{{version}}/broadcasting) 文件。


<a name="the-console-directory"></a>
### `Console` 目錄

Console 目錄包含應用程式中所有自訂的 Artisan 命令。這些命令可以使用 `make:command` 命令生成。


<a name="the-events-directory"></a>
### `Events` 目錄

此目錄預設不存在，但會由 `event:generate` 和 `make:event` Artisan 命令為您創建。Events 目錄包含 [事件類別](/docs/{{version}}/events)。事件可用來通知應用程式的其他部分已發生特定動作，這提供了極大的靈活性和解耦。


<a name="the-exceptions-directory"></a>
### `Exceptions` 目錄

Exceptions 目錄包含應用程式中所有自訂的例外狀況。這些例外狀況可以使用 `make:exception` 命令生成。


<a name="the-http-directory"></a>
### `Http` 目錄

Http 目錄包含您的控制器、中介層和表單請求。幾乎所有處理進入應用程式請求的邏輯都將放在此目錄中。


<a name="the-jobs-directory"></a>
### `Jobs` 目錄

此目錄預設不存在，但若您執行 `make:job` Artisan 命令則會為您創建。Jobs 目錄包含應用程式中 [可佇列的 Job](/docs/{{version}}/queues)。Job 可由應用程式佇列處理，或在當前請求生命週期內同步運行。在當前請求期間同步運行的 Job 有時被稱為「命令」，因為它們是 [命令模式](https://en.wikipedia.org/wiki/Command_pattern) 的一種實作。


<a name="the-listeners-directory"></a>
### `Listeners` 目錄

此目錄預設不存在，但若您執行 `event:generate` 或 `make:listener` Artisan 命令則會為您創建。Listeners 目錄包含處理您的 [事件](/docs/{{version}}/events) 的類別。事件監聽器接收一個事件實例，並回應事件觸發來執行邏輯。例如，`UserRegistered` 事件可能會由 `SendWelcomeEmail` 監聽器處理。


<a name="the-mail-directory"></a>
### `Mail` 目錄

此目錄預設不存在，但若您執行 `make:mail` Artisan 命令則會為您創建。Mail 目錄包含應用程式所有 [代表電子郵件的類別](/docs/{{version}}/mail)。Mail 物件讓您可以將建立電子郵件的所有邏輯封裝在一個單一、簡單的類別中，並可使用 `Mail::send` 方法發送。


<a name="the-models-directory"></a>
### `Models` 目錄

Models 目錄包含所有您的 [Eloquent 模型類別](/docs/{{version}}/eloquent)。Laravel 隨附的 Eloquent ORM 提供了一個優雅、簡單的 ActiveRecord 實作，用於處理您的資料庫。每個資料庫表格都有一個對應的「Model」，用於與該表格互動。Models 讓您可以查詢表格中的資料，以及插入新記錄到表格中。


<a name="the-notifications-directory"></a>
### `Notifications` 目錄

此目錄預設不存在，但若您執行 `make:notification` Artisan 命令則會為您創建。Notifications 目錄包含應用程式中所有「交易式」的 [通知](/docs/{{version}}/notifications)，例如關於應用程式內發生的事件的簡單通知。Laravel 的通知功能抽象化了透過多種驅動器發送通知，例如電子郵件、Slack、SMS 或儲存到資料庫中。


<a name="the-policies-directory"></a>
### `Policies` 目錄

此目錄預設不存在，但若您執行 `make:policy` Artisan 命令則會為您創建。Policies 目錄包含應用程式的 [授權 Policy 類別](/docs/{{version}}/authorization)。Policies 用於判斷使用者是否可以對資源執行特定動作。


<a name="the-providers-directory"></a>
### `Providers` 目錄

Providers 目錄包含應用程式中所有的 [服務提供者](/docs/{{version}}/providers)。服務提供者透過在服務容器中綁定服務、註冊事件或執行任何其他任務來引導應用程式，以準備處理傳入的請求。

在一個全新的 Laravel 應用程式中，此目錄已經包含 `AppServiceProvider`。您可以根據需要自由地將自己的提供者新增到此目錄。


<a name="the-rules-directory"></a>
### `Rules` 目錄

此目錄預設不存在，但若您執行 `make:rule` Artisan 命令則會為您創建。Rules 目錄包含應用程式中自訂的驗證規則物件。Rules 用於將複雜的驗證邏輯封裝在一個簡單的物件中。欲了解更多資訊，請查閱 [驗證文件](/docs/{{version}}/validation)。