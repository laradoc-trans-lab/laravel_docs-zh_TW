# 安裝

- [認識 Laravel](#meet-laravel)
    - [為何選擇 Laravel？](#why-laravel)
- [建立 Laravel 應用程式](#creating-a-laravel-project)
    - [安裝 PHP 與 Laravel 安裝器](#installing-php)
    - [建立應用程式](#creating-an-application)
- [初始設定](#initial-configuration)
    - [基於環境的設定](#environment-based-configuration)
    - [資料庫與遷移](#databases-and-migrations)
    - [目錄設定](#directory-configuration)
- [使用 Herd 安裝](#installation-using-herd)
    - [macOS 上的 Herd](#herd-on-macos)
    - [Windows 上的 Herd](#herd-on-windows)
- [IDE 支援](#ide-support)
- [Laravel 與 AI](#laravel-and-ai)
    - [安裝 Laravel Boost](#installing-laravel-boost)
- [下一步](#next-steps)
    - [Laravel：全端框架](#laravel-the-fullstack-framework)
    - [Laravel：API 後端](#laravel-the-api-backend)

<a name="meet-laravel"></a>
## 認識 Laravel

Laravel 是一個富有表現力、優雅語法的 Web 應用程式框架。Web 框架為建立您的應用程式提供了一個結構和起點，讓您可以專注於創造驚豔的應用程式，而我們處理瑣碎細節。

Laravel 致力於提供卓越的開發者體驗，同時提供強大功能，例如完善的依賴注入、富有表現力的資料庫抽象層、佇列與排程工作、單元測試與整合測試等等。

無論您是 PHP Web 框架的新手還是經驗豐富的開發者，Laravel 都是一個可以與您一同成長的框架。我們將幫助您踏出 Web 開發者的第一步，或在您將專業知識提升到新的水平時助您一臂之力。我們迫不及待想看到您的成果。

<a name="why-laravel"></a>
### 為何選擇 Laravel？

建立 Web 應用程式時，您可以選擇多種工具和框架。然而，我們相信 Laravel 是建立現代、全端 Web 應用程式的最佳選擇。

#### 漸進式框架

我們喜歡稱 Laravel 為「漸進式」框架。透過此說法，我們的意思是 Laravel 會與您一同成長。如果您剛踏入 Web 開發領域，Laravel 龐大的文件庫、指南和 [影片教學](https://laracasts.com) 將幫助您輕鬆學習，不會感到手足無措。

如果您是資深開發者，Laravel 針對 [依賴注入](/docs/{{version}}/container)、[單元測試](/docs/{{version}}/testing)、[佇列](/docs/{{version}}/queues)、[即時事件](/docs/{{version}}/broadcasting) 等提供強大工具。Laravel 經過精心調整，用於建立專業級 Web 應用程式，並準備好處理企業級工作負載。

#### 可擴展框架

Laravel 具有令人難以置信的可擴展性。歸功於 PHP 對於擴展的友好特性，以及 Laravel 內建支援 Redis 等快速分散式快取系統，使用 Laravel 進行水平擴展輕而易舉。事實上，Laravel 應用程式已輕鬆擴展以處理每月數億次請求。

需要極限擴展嗎？[Laravel Cloud](https://cloud.laravel.com) 等平台讓您能夠以幾乎無限的規模執行您的 Laravel 應用程式。

#### 社群框架

Laravel 結合了 PHP 生態系中最好的套件，以提供最為強大且開發者友善的框架。此外，來自世界各地的數千名才華橫溢的開發者都曾 [為此框架做出貢獻](https://github.com/laravel/framework)。誰知道呢，也許您也會成為一名 Laravel 貢獻者。

<a name="creating-a-laravel-project"></a>
## 建立 Laravel 應用程式

<a name="installing-php"></a>
### 安裝 PHP 與 Laravel 安裝器

在建立您的第一個 Laravel 應用程式之前，請確保您的本地機器已安裝 [PHP](https://php.net)、[Composer](https://getcomposer.org) 和 [Laravel 安裝器](https://github.com/laravel/installer)。此外，您應該安裝 [Node 和 NPM](https://nodejs.org) 或 [Bun](https://bun.sh/)，以便您可以編譯應用程式的前端資源。

如果您的本地機器上沒有安裝 PHP 和 Composer，以下命令將在 macOS、Windows 或 Linux 上安裝 PHP、Composer 和 Laravel 安裝器：

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

執行上述任一命令後，您應該重新啟動您的終端機會話。若要在透過 `php.new` 安裝 PHP、Composer 和 Laravel 安裝器後進行更新，您可以在終端機中重新執行該命令。

如果您已經安裝了 PHP 和 Composer，您可以透過 Composer 安裝 Laravel 安裝器：

```shell
composer global require laravel/installer
```

> [!NOTE]
> 若要獲得功能齊全的圖形化 PHP 安裝與管理體驗，請參考 [Laravel Herd](#installation-using-herd)。

<a name="creating-an-application"></a>
### 建立應用程式

安裝 PHP、Composer 和 Laravel 安裝器後，您就可以建立新的 Laravel 應用程式了。Laravel 安裝器將提示您選擇偏好的測試框架、資料庫和入門套件：

```shell
laravel new example-app
```

應用程式建立後，您可以使用 `dev` Composer 腳本啟動 Laravel 的本地開發伺服器、佇列工作者和 Vite 開發伺服器：

```shell
cd example-app
npm install && npm run build
composer run dev
```

啟動開發伺服器後，您的應用程式將可在網路瀏覽器中透過 [http://localhost:8000](http://localhost:8000) 存取。接下來，您就可以 [開始邁向 Laravel 生態系統的下一步](#next-steps)。當然，您可能也想 [設定資料庫](#databases-and-migrations)。

> [!NOTE]
> 如果您想在開發 Laravel 應用程式時領先一步，請考慮使用我們其中一個 [入門套件](/docs/{{version}}/starter-kits)。Laravel 的入門套件為您的新 Laravel 應用程式提供後端與前端身份驗證骨架。

<a name="initial-configuration"></a>
## 初始設定

Laravel 框架的所有設定檔都儲存在 `config` 目錄中。每個選項都有詳細說明，您可以隨意查閱這些檔案，熟悉可用的選項。

Laravel 開箱即用，幾乎不需要額外設定。您可以自由地開始開發！不過，您可能希望查閱 `config/app.php` 檔案及其說明文件。它包含多個選項，例如 `url` 和 `locale`，您可以根據應用程式的需求進行修改。


<a name="environment-based-configuration"></a>
### 基於環境的設定

由於 Laravel 的許多設定選項值可能會因應用程式是在您的本機或生產環境的網頁伺服器上執行而異，因此許多重要的設定值都是透過應用程式根目錄中的 `.env` 檔案來定義。

您的 `.env` 檔案不應提交到應用程式的原始碼控制中，因為每個使用您應用程式的開發者/伺服器可能需要不同的環境設定。此外，若入侵者獲得您的原始碼儲存庫的存取權，這將會構成安全風險，因為任何敏感憑證都將被暴露。

> [!NOTE]
> 如需關於 `.env` 檔案和基於環境的設定的更多資訊，請查閱完整的[設定說明文件](/docs/{{version}}/configuration#environment-configuration)。


<a name="databases-and-migrations"></a>
### 資料庫與遷移

在您建立 Laravel 應用程式後，您可能希望將一些資料儲存在資料庫中。預設情況下，您的應用程式的 `.env` 設定檔指定 Laravel 將與 SQLite 資料庫互動。

在應用程式建立過程中，Laravel 會為您建立一個 `database/database.sqlite` 檔案，並執行必要的遷移以建立應用程式的資料庫資料表。

如果您偏好使用其他資料庫驅動程式，例如 MySQL 或 PostgreSQL，您可以更新您的 `.env` 設定檔以使用適當的資料庫。例如，如果您希望使用 MySQL，請更新您的 `.env` 設定檔中的 `DB_*` 變數，如下所示：

```ini
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=laravel
DB_USERNAME=root
DB_PASSWORD=
```

如果您選擇使用 SQLite 以外的資料庫，您將需要建立資料庫並執行應用程式的[資料庫遷移](/docs/{{version}}/migrations)：

```shell
php artisan migrate
```

> [!NOTE]
> 如果您在 macOS 或 Windows 上開發，並且需要本機安裝 MySQL、PostgreSQL 或 Redis，請考慮使用 [Herd Pro](https://herd.laravel.com/#plans) 或 [DBngin](https://dbngin.com/)。


<a name="directory-configuration"></a>
### 目錄設定

Laravel 應始終從為您的網頁伺服器設定的「web 目錄」根目錄中提供服務。您不應嘗試從「web 目錄」的子目錄中提供 Laravel 應用程式。這樣做可能會暴露應用程式中存在的敏感檔案。


<a name="installation-using-herd"></a>
## 使用 Herd 安裝

[Laravel Herd](https://herd.laravel.com) 是一個極速、原生的 Laravel 和 PHP 開發環境，適用於 macOS 和 Windows。Herd 包含了開始 Laravel 開發所需的一切，包括 PHP 和 Nginx。

安裝 Herd 後，您就可以開始使用 Laravel 進行開發了。Herd 包含了 `php` 、 `composer` 、 `laravel` 、 `expose` 、 `node` 、 `npm` 和 `nvm` 的命令列工具。

> [!NOTE]
> [Herd Pro](https://herd.laravel.com/#plans) 透過額外的強大功能強化了 Herd，例如建立和管理本機 MySQL、Postgres 和 Redis 資料庫的能力，以及本機郵件檢視和日誌監控功能。


<a name="herd-on-macos"></a>
### macOS 上的 Herd

如果您在 macOS 上開發，您可以從 [Herd 網站](https://herd.laravel.com) 下載 Herd 安裝程式。安裝程式會自動下載最新版本的 PHP，並將您的 Mac 設定為始終在背景執行 [Nginx](https://www.nginx.com/)。

macOS 上的 Herd 使用 [dnsmasq](https://en.wikipedia.org/wiki/Dnsmasq) 來支援「停泊目錄」(parked directories)。任何位於停泊目錄中的 Laravel 應用程式都將由 Herd 自動提供服務。預設情況下，Herd 會在 `~/Herd` 建立一個停泊目錄，您可以透過該目錄的名稱，在 `.test` 網域上存取此目錄中的任何 Laravel 應用程式。

安裝 Herd 後，建立新的 Laravel 應用程式最快的方式是使用與 Herd 捆綁的 Laravel CLI：

```shell
cd ~/Herd
laravel new my-app
cd my-app
herd open
```

當然，您始終可以透過 Herd 的使用者介面 (UI) 來管理您的停泊目錄和其他 PHP 設定，該介面可以從系統匣中的 Herd 選單打開。

您可以查閱 [Herd 說明文件](https://herd.laravel.com/docs) 以了解更多關於 Herd 的資訊。


<a name="herd-on-windows"></a>
### Windows 上的 Herd

您可以從 [Herd 網站](https://herd.laravel.com/windows) 下載適用於 Windows 的 Herd 安裝程式。安裝完成後，您可以啟動 Herd 以完成入門程序並首次存取 Herd 的使用者介面 (UI)。

透過左鍵點擊 Herd 的系統匣圖示即可存取 Herd 的使用者介面。右鍵點擊會開啟快速選單，讓您能夠存取日常所需的所有工具。

在安裝過程中，Herd 會在您的家目錄中 `%USERPROFILE%\Herd` 建立一個「停泊目錄」(parked directory)。任何位於停泊目錄中的 Laravel 應用程式都將由 Herd 自動提供服務，您可以透過該目錄的名稱，在 `.test` 網域上存取此目錄中的任何 Laravel 應用程式。

安裝 Herd 後，建立新的 Laravel 應用程式最快的方式是使用與 Herd 捆綁的 Laravel CLI。要開始使用，請打開 Powershell 並執行以下命令：

```shell
cd ~\Herd
laravel new my-app
cd my-app
herd open
```

您可以查閱 [Herd 的 Windows 說明文件](https://herd.laravel.com/docs/windows) 以了解更多關於 Herd 的資訊。


<a name="ide-support"></a>
## IDE 支援

在開發 Laravel 應用程式時，您可以自由使用任何您喜歡的程式碼編輯器。如果您正在尋找輕量且可擴展的編輯器，[VS Code](https://code.visualstudio.com) 或 [Cursor](https://cursor.com) 結合官方的 [Laravel VS Code Extension](https://marketplace.visualstudio.com/items?itemName=laravel.vscode-laravel) 提供卓越的 Laravel 支援，具有語法高亮、程式碼片段、Artisan 命令整合，以及針對 Eloquent 模型、路由、中介層 (middleware)、前端資源 (assets)、設定 (config) 和 Inertia.js 的智慧型自動完成功能。

如需對 Laravel 的廣泛而強大的支援，請參考 [PhpStorm](https://www.jetbrains.com/phpstorm/laravel/?utm_source=laravel.com&utm_medium=link&utm_campaign=laravel-2025&utm_content=partner&ref=laravel-2025)，這是一個 JetBrains 的 IDE。有了 [Laravel Idea plugin](https://laravel-idea.com/)，它為 Laravel 及其生態系統提供了精確的支援，包括 Laravel Pint、Pest、Larastan 等。Laravel Idea 的框架支援包括 Blade 模板、針對 Eloquent 模型、路由、視圖、翻譯和元件的智慧型自動完成功能，以及強大的程式碼生成和在 Laravel 專案中的導航。

對於那些尋求基於雲端的開發體驗的人來說，[Firebase Studio](https://firebase.studio/) 提供了直接在瀏覽器中建構 Laravel 的即時存取。Firebase Studio 無需任何設定，讓您輕鬆地從任何裝置開始建構 Laravel 應用程式。

<a name="laravel-and-ai"></a>
## Laravel 與 AI

[Laravel Boost](https://github.com/laravel/boost) 是一個強大的工具，它彌補了 AI 程式碼代理與 Laravel 應用程式之間的鴻溝。Boost 為 AI 代理提供了 Laravel 特定的上下文、工具與準則，使其能夠生成更精確、版本專屬且符合 Laravel 慣例的程式碼。

當您在 Laravel 應用程式中安裝 Boost 後，AI 代理即可使用超過 15 種專業工具，包括得知您正在使用的套件、查詢您的資料庫、搜尋 Laravel 文件、讀取瀏覽器日誌、生成測試以及透過 Tinker 執行程式碼。

此外，Boost 還為 AI 代理提供了超過 17,000 份向量化的 Laravel 生態系統文件，這些文件與您已安裝的套件版本相關。這表示代理可以提供針對您專案所使用確切版本的指導。

Boost 也包含 Laravel 維護的 AI 準則，這些準則有助於代理遵循框架慣例、編寫適當的測試，並避免在生成 Laravel 程式碼時常見的陷阱。

<a name="installing-laravel-boost"></a>
### 安裝 Laravel Boost

Boost 可安裝於執行 PHP 8.1 或更高版本的 Laravel 10、11 和 12 應用程式中。首先，請將 Boost 作為開發依賴進行安裝：

```shell
composer require laravel/boost --dev
```

安裝完成後，執行互動式安裝器：

```shell
php artisan boost:install
```

安裝器將自動偵測您的 IDE 和 AI 代理，讓您可以選擇啟用對您的專案有意義的功能。Boost 尊重現有的專案慣例，預設情況下不強制執行帶有偏見的樣式規則。

> [!NOTE]
> 要了解更多關於 Boost 的資訊，請參閱 [GitHub 上的 Laravel Boost 儲存庫](https://github.com/laravel/boost)。

<a name="next-steps"></a>
## 下一步

現在您已經建立了一個 Laravel 應用程式，您可能會想接下來要學習什麼。首先，我們強烈建議您透過閱讀以下文件來熟悉 Laravel 的運作方式：

<div class="content-list" markdown="1">

- [請求生命週期](/docs/{{version}}/lifecycle)
- [設定](/docs/{{version}}/configuration)
- [目錄結構](/docs/{{version}}/structure)
- [前端](/docs/{{version}}/frontend)
- [服務容器](/docs/{{version}}/container)
- [Facades](/docs/{{version}}/facades)

</div>

您如何使用 Laravel 也將決定您接下來的學習路徑。Laravel 有多種使用方式，我們將在下方探討該框架的兩種主要使用情境。

<a name="laravel-the-fullstack-framework"></a>
### Laravel：全端框架

Laravel 可以作為一個全端框架。所謂「全端」框架，指的是您將使用 Laravel 來路由應用程式的請求，並透過 [Blade 模板](/docs/{{version}}/blade) 或像 [Inertia](https://inertiajs.com) 這樣的單頁應用程式混合技術來渲染您的前端。這是使用 Laravel 框架最常見的方式，且依我們之見，也是使用 Laravel 最有效率的方式。

如果您打算這樣使用 Laravel，您可能會想查閱我們關於 [前端開發](/docs/{{version}}/frontend)、[路由](/docs/{{version}}/routing)、[視圖](/docs/{{version}}/views) 或 [Eloquent ORM](/docs/{{version}}/eloquent) 的文件。此外，您可能也會對學習像 [Livewire](https://livewire.laravel.com) 和 [Inertia](https://inertiajs.com) 這樣的社群套件感興趣。這些套件讓您能將 Laravel 作為全端框架使用，同時享有單頁 JavaScript 應用程式所提供的許多 UI 優勢。

如果您正在將 Laravel 作為全端框架使用，我們也強烈建議您學習如何使用 [Vite](/docs/{{version}}/vite) 編譯您應用程式的 CSS 和 JavaScript。

> [!NOTE]
> 如果您想在建立應用程式時搶得先機，請查看我們官方的 [應用程式入門套件](/docs/{{version}}/starter-kits) 之一。

<a name="laravel-the-api-backend"></a>
### Laravel：API 後端

Laravel 也可以作為 JavaScript 單頁應用程式或行動應用程式的 API 後端。例如，您可以使用 Laravel 作為您的 [Next.js](https://nextjs.org) 應用程式的 API 後端。在此情境下，您可以使用 Laravel 為您的應用程式提供 [身份驗證](/docs/{{version}}/sanctum) 和資料儲存/擷取，同時也能利用 Laravel 強大的服務，例如佇列 (queues)、電子郵件 (emails)、通知 (notifications) 等。

如果您打算這樣使用 Laravel，您可能會想查閱我們關於 [路由](/docs/{{version}}/routing)、[Laravel Sanctum](/docs/{{version}}/sanctum) 和 [Eloquent ORM](/docs/{{version}}/eloquent) 的文件。