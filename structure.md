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
- [`App` 目錄](#the-app-directory)
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

Laravel 預設的應用程式結構旨在為大型和小型應用程式提供一個絕佳的起點。但您可以自由地按照自己喜歡的方式組織應用程式。Laravel 對於任何給定類別的位置幾乎沒有限制——只要 Composer 可以自動載入該類別即可。

> [!NOTE]
> 剛接觸 Laravel？請查看 [Laravel Bootcamp](https://bootcamp.laravel.com)，在我們引導您建構第一個 Laravel 應用程式時，親身體驗這個框架。

<a name="the-root-directory"></a>
## 根目錄

<a name="the-root-app-directory"></a>
### `App` 目錄

`app` 目錄包含您應用程式的核心程式碼。我們很快就會更詳細地探討這個目錄；然而，您應用程式中幾乎所有的類別都將位於此目錄中。

<a name="the-bootstrap-directory"></a>
### `Bootstrap` 目錄

`bootstrap` 目錄包含 `app.php` 檔案，該檔案用於啟動框架。此目錄還包含一個 `cache` 目錄，其中包含為效能優化而由框架生成的檔案，例如路由和服務快取檔案。

<a name="the-config-directory"></a>
### `Config` 目錄

`config` 目錄，顧名思義，包含您應用程式的所有設定檔。仔細閱讀所有這些檔案並熟悉所有可用的選項是一個好主意。

<a name="the-database-directory"></a>
### `Database` 目錄

`database` 目錄包含您的資料庫遷移 (migrations)、Model 工廠 (factories) 和填充 (seeds)。如果您願意，也可以使用此目錄來存放 SQLite 資料庫。

<a name="the-public-directory"></a>
### `Public` 目錄

`public` 目錄包含 `index.php` 檔案，它是所有進入您應用程式的請求的入口點，並設定自動載入。此目錄還包含您的資產，例如圖片、JavaScript 和 CSS。

<a name="the-resources-directory"></a>
### `Resources` 目錄

`resources` 目錄包含您的 [views](/docs/{{version}}/views) 以及未編譯的原始資產，例如 CSS 或 JavaScript。

<a name="the-routes-directory"></a>
### `Routes` 目錄

`routes` 目錄包含您應用程式的所有路由定義。預設情況下，Laravel 包含兩個路由檔案：`web.php` 和 `console.php`。

`web.php` 檔案包含 Laravel 放置在 `web` Middleware 群組中的路由，該群組提供 Session 狀態、CSRF 保護和 Cookie 加密。如果您的應用程式不提供無狀態的 RESTful API，那麼您所有的路由很可能都會在 `web.php` 檔案中定義。

`console.php` 檔案是您可以定義所有基於閉包 (closure) 的 Console 命令的地方。每個閉包都綁定到一個命令實例，提供了一種與每個命令的 IO 方法互動的簡單方法。儘管此檔案不定義 HTTP 路由，但它定義了基於 Console 的應用程式入口點 (路由)。您也可以在 `console.php` 檔案中 [排程](/docs/{{version}}/scheduling) 任務。

您可以選擇透過 `install:api` 和 `install:broadcasting` Artisan 命令，安裝用於 API 路由 (`api.php`) 和 Broadcasting 頻道 (`channels.php`) 的額外路由檔案。

`api.php` 檔案包含旨在無狀態的路由，因此透過這些路由進入應用程式的請求旨在 [透過 Token](/docs/{{version}}/sanctum) 進行身份驗證，並且無法存取 Session 狀態。

`channels.php` 檔案是您可以註冊應用程式支援的所有 [事件 Broadcasting](/docs/{{version}}/broadcasting) 頻道的地方。

<a name="the-storage-directory"></a>
### `Storage` 目錄

`storage` 目錄包含您的日誌、編譯後的 Blade 模板、基於檔案的 Session、檔案快取以及框架生成的其他檔案。此目錄分為 `app`、`framework` 和 `logs` 目錄。`app` 目錄可用於儲存應用程式生成的任何檔案。`framework` 目錄用於儲存框架生成的檔案和快取。最後，`logs` 目錄包含您應用程式的日誌檔案。

`storage/app/public` 目錄可用於儲存使用者生成的檔案，例如個人資料頭像，這些檔案應該是公開可存取的。您應該在 `public/storage` 建立一個指向此目錄的符號連結。您可以使用 `php artisan storage:link` Artisan 命令建立該連結。

<a name="the-tests-directory"></a>
### `Tests` 目錄

`tests` 目錄包含您的自動化測試。開箱即用提供了範例 [Pest](https://pestphp.com) 或 [PHPUnit](https://phpunit.de/) 單元測試和功能測試。每個測試類別都應以 `Test` 一詞作為後綴。您可以使用 `/vendor/bin/pest` 或 `/vendor/bin/phpunit` 命令執行測試。或者，如果您想要更詳細和美觀的測試結果呈現，您可以使用 `php artisan test` Artisan 命令執行測試。

<a name="the-vendor-directory"></a>
### `Vendor` 目錄

`vendor` 目錄包含您的 [Composer](https://getcomposer.org) 依賴項。

<a name="the-app-directory"></a>
## `App` 目錄

您應用程式的大部分內容都位於 `app` 目錄中。預設情況下，此目錄的命名空間為 `App`，並由 Composer 使用 [PSR-4 自動載入標準](https://www.php-fig.org/psr/psr-4/) 自動載入。

預設情況下，`app` 目錄包含 `Http`、`Models` 和 `Providers` 目錄。然而，隨著時間的推移，當您使用 `make` Artisan 命令生成類別時，`app` 目錄內會生成各種其他目錄。例如，`app/Console` 目錄在您執行 `make:command` Artisan 命令生成命令類別之前不會存在。

`Console` 和 `Http` 目錄在下面的各自部分中會進一步解釋，但您可以將 `Console` 和 `Http` 目錄視為提供應用程式核心的 API。HTTP 協定和 CLI 都是與應用程式互動的機制，但實際上不包含應用程式邏輯。換句話說，它們是向應用程式發出命令的兩種方式。`Console` 目錄包含您所有的 Artisan 命令，而 `Http` 目錄包含您的控制器、Middleware 和請求。

> [!NOTE]
> `app` 目錄中的許多類別都可以透過 Artisan 命令生成。要查看可用的命令，請在終端機中執行 `php artisan list make` 命令。

<a name="the-broadcasting-directory"></a>
### `Broadcasting` 目錄

`Broadcasting` 目錄包含您應用程式的所有廣播頻道類別。這些類別是使用 `make:channel` 命令生成的。此目錄預設不存在，但在您建立第一個頻道時會為您建立。要了解有關頻道的更多資訊，請查看 [事件 Broadcasting](/docs/{{version}}/broadcasting) 的說明文件。

<a name="the-console-directory"></a>
### `Console` 目錄

`Console` 目錄包含您應用程式的所有自訂 Artisan 命令。這些命令可以使用 `make:command` 命令生成。

<a name="the-events-directory"></a>
### `Events` 目錄

此目錄預設不存在，但會由 `event:generate` 和 `make:event` Artisan 命令為您建立。`Events` 目錄存放 [事件類別](/docs/{{version}}/events)。事件可用於提醒應用程式的其他部分已發生特定動作，提供了極大的靈活性和解耦。

<a name="the-exceptions-directory"></a>
### `Exceptions` 目錄

`Exceptions` 目錄包含您應用程式的所有自訂例外。這些例外可以使用 `make:exception` 命令生成。

<a name="the-http-directory"></a>
### `Http` 目錄

`Http` 目錄包含您的控制器、Middleware 和表單請求。幾乎所有處理進入您應用程式的請求的邏輯都將放置在此目錄中。

<a name="the-jobs-directory"></a>
### `Jobs` 目錄

此目錄預設不存在，但如果您執行 `make:job` Artisan 命令，則會為您建立。`Jobs` 目錄存放您應用程式的 [可佇列任務](/docs/{{version}}/queues)。任務可以由您的應用程式佇列或在當前請求生命週期內同步執行。在當前請求期間同步執行的任務有時被稱為「命令」，因為它們是 [命令模式](https://en.wikipedia.org/wiki/Command_pattern) 的實作。

<a name="the-listeners-directory"></a>
### `Listeners` 目錄

此目錄預設不存在，但如果您執行 `event:generate` 或 `make:listener` Artisan 命令，則會為您建立。`Listeners` 目錄包含處理您 [事件](/docs/{{version}}/events) 的類別。事件監聽器接收一個事件實例並執行邏輯以回應事件的觸發。例如，`UserRegistered` 事件可能由 `SendWelcomeEmail` 監聽器處理。

<a name="the-mail-directory"></a>
### `Mail` 目錄

此目錄預設不存在，但如果您執行 `make:mail` Artisan 命令，則會為您建立。`Mail` 目錄包含您應用程式發送的所有 [代表電子郵件的類別](/docs/{{version}}/mail)。Mail 物件允許您將建構電子郵件的所有邏輯封裝在一個簡單的類別中，該類別可以使用 `Mail::send` 方法發送。

<a name="the-models-directory"></a>
### `Models` 目錄

`Models` 目錄包含您所有的 [Eloquent Model 類別](/docs/{{version}}/eloquent)。Laravel 隨附的 Eloquent ORM 提供了一個優雅、簡單的 ActiveRecord 實作，用於處理您的資料庫。每個資料庫表都有一個對應的「Model」，用於與該表互動。Model 允許您查詢表中的資料，以及向表中插入新記錄。

<a name="the-notifications-directory"></a>
### `Notifications` 目錄

此目錄預設不存在，但如果您執行 `make:notification` Artisan 命令，則會為您建立。`Notifications` 目錄包含您應用程式發送的所有「交易式」[通知](/docs/{{version}}/notifications)，例如關於應用程式內發生的事件的簡單通知。Laravel 的通知功能抽象了透過各種驅動程式（例如電子郵件、Slack、SMS 或儲存在資料庫中）發送通知。

<a name="the-policies-directory"></a>
### `Policies` 目錄

此目錄預設不存在，但如果您執行 `make:policy` Artisan 命令，則會為您建立。`Policies` 目錄包含您應用程式的 [授權 Policy 類別](/docs/{{version}}/authorization)。Policy 用於判斷使用者是否可以對資源執行給定動作。

<a name="the-providers-directory"></a>
### `Providers` 目錄

`Providers` 目錄包含您應用程式的所有 [Service Provider](/docs/{{version}}/providers)。Service Provider 透過在 Service Container 中綁定服務、註冊事件或執行任何其他任務來準備您的應用程式以處理傳入請求，從而啟動您的應用程式。

在一個全新的 Laravel 應用程式中，此目錄將已經包含 `AppServiceProvider`。您可以根據需要自由地將自己的 Provider 添加到此目錄中。

<a name="the-rules-directory"></a>
### `Rules` 目錄

此目錄預設不存在，但如果您執行 `make:rule` Artisan 命令，則會為您建立。`Rules` 目錄包含您應用程式的自訂驗證規則物件。規則用於將複雜的驗證邏輯封裝在一個簡單的物件中。有關更多資訊，請查看 [驗證說明文件](/docs/{{version}}/validation)。
