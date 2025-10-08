# 安裝

- [認識 Laravel](#meet-laravel)
    - [為何選擇 Laravel？](#why-laravel)
- [建立 Laravel 應用程式](#creating-a-laravel-project)
    - [安裝 PHP 與 Laravel 安裝器](#installing-php)
    - [建立應用程式](#creating-an-application)
- [初始設定](#initial-configuration)
    - [基於環境的設定](#environment-based-configuration)
    - [資料庫與資料庫遷移](#databases-and-migrations)
    - [目錄設定](#directory-configuration)
- [使用 Herd 進行安裝](#installation-using-herd)
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

Laravel 是一個具備表現力且語法優雅的網頁應用程式框架。網頁框架為您的應用程式提供架構與起始點，讓您可以專注於創造令人驚豔的事物，而我們則負責處理細節。

Laravel 致力於提供卓越的開發者體驗，同時提供強大的功能，例如完善的依賴注入、具表現力的資料庫抽象層、佇列與排程工作、單元測試與整合測試等等。

無論您是 PHP 網頁框架的新手，還是擁有多年經驗，Laravel 都是一個能與您共同成長的框架。我們將協助您踏出作為網頁開發者的第一步，或在您提升專業知識時助您一臂之力。我們迫不及待地想看到您所打造的一切。


<a name="why-laravel"></a>
### 為何選擇 Laravel？

在建構網頁應用程式時，有許多工具與框架可供選擇。然而，我們相信 Laravel 是建構現代化全端網頁應用程式的最佳選擇。


#### 漸進式框架

我們喜歡稱 Laravel 為「漸進式」框架。這表示 Laravel 會與您一同成長。如果您剛踏入網頁開發領域，Laravel 龐大的文件庫、指南和 [影片教學](https://laracasts.com) 將幫助您輕鬆入門，而不會感到不知所措。

如果您是一位資深開發者，Laravel 提供強大的工具，用於 [依賴注入](/docs/{{version}}/container)、[單元測試](/docs/{{version}}/testing)、[佇列](/docs/{{version}}/queues)、[即時事件](/docs/{{version}}/broadcasting) 等。Laravel 經過精心調整，適用於建構專業級網頁應用程式，並已準備好處理企業級工作負載。


#### 可擴展的框架

Laravel 具有驚人的可擴展性。由於 PHP 友善的擴展特性，以及 Laravel 內建對 Redis 等快速分散式快取系統的支援，透過 Laravel 進行水平擴展輕而易舉。事實上，Laravel 應用程式已能輕鬆擴展，每月處理數億次請求。

需要極致擴展？[Laravel Cloud](https://cloud.laravel.com) 等平台讓您能以近乎無限的規模運行您的 Laravel 應用程式。


#### 社群框架

Laravel 整合了 PHP 生態系統中最佳的套件，以提供最穩健且開發者友善的框架。此外，全球數千名才華洋溢的開發者也為 [該框架貢獻](https://github.com/laravel/framework) 了一份心力。誰知道呢，或許您也會成為 Laravel 的貢獻者。


<a name="creating-a-laravel-project"></a>
## 建立 Laravel 應用程式


<a name="installing-php"></a>
### 安裝 PHP 與 Laravel 安裝器

在建立您的第一個 Laravel 應用程式之前，請確保您的本機電腦已安裝 [PHP](https://php.net)、[Composer](https://getcomposer.org) 和 [Laravel 安裝器](https://github.com/laravel/installer)。此外，您應該安裝 [Node 和 NPM](https://nodejs.org) 或 [Bun](https://bun.sh/)，以便編譯您應用程式的前端資源。

如果您的本機電腦尚未安裝 PHP 和 Composer，以下指令將在 macOS、Windows 或 Linux 上安裝 PHP、Composer 和 Laravel 安裝器：

```shell tab=macOS
/bin/bash -c "$(curl -fsSL https://php.new/install/mac/8.4)"
```

```shell tab=Windows PowerShell
```

```
# Run as administrator...
Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://php.new/install/windows/8.4'))
```

```shell tab=Linux
/bin/bash -c "$(curl -fsSL https://php.new/install/linux/8.4)"
```

執行上述任一指令後，您應重新啟動終端機會話。若要透過 `php.new` 安裝後更新 PHP、Composer 和 Laravel 安裝器，您可以在終端機中重新執行該指令。

如果您已經安裝 PHP 和 Composer，您可以透過 Composer 安裝 Laravel 安裝器：

```shell
composer global require laravel/installer
```

> [!NOTE]
> 欲獲得功能齊全的圖形化 PHP 安裝與管理體驗，請參考 [Laravel Herd](#installation-using-herd)。


<a name="creating-an-application"></a>
### 建立應用程式

安裝好 PHP、Composer 和 Laravel 安裝器後，您就可以建立新的 Laravel 應用程式了。Laravel 安裝器會提示您選擇偏好的測試框架、資料庫和新手套件：

```shell
laravel new example-app
```

應用程式建立後，您可以使用 `dev` Composer 腳本來啟動 Laravel 的本地開發伺服器、佇列 Worker 和 Vite 開發伺服器：

```shell
cd example-app
npm install && npm run build
composer run dev
```

啟動開發伺服器後，您的應用程式將可在網頁瀏覽器中透過 [http://localhost:8000](http://localhost:8000) 存取。接下來，您就可以 [開始邁向 Laravel 生態系統的下一步了](#next-steps)。當然，您可能還會想要 [設定資料庫](#databases-and-migrations)。

> [!NOTE]
> 如果您想在開發 Laravel 應用程式時搶先一步，請考慮使用我們的其中一個 [新手套件](/docs/{{version}}/starter-kits)。Laravel 的新手套件為您的新 Laravel 應用程式提供後端與前端驗證骨架。


<a name="initial-configuration"></a>
## 初始設定

所有 Laravel 框架的設定檔都儲存在 `config` 目錄中。每個選項都有詳細說明，您可以隨意瀏覽這些檔案，熟悉可用的選項。

Laravel 開箱即用，幾乎不需要額外設定。您可以自由開始開發！不過，您可能希望檢閱 `config/app.php` 檔案及其文件。它包含了一些選項，例如 `url` 和 `locale`，您可以根據應用程式的需求進行更改。


<a name="environment-based-configuration"></a>
### 基於環境的設定

由於許多 Laravel 的設定選項值可能因應用程式是在本地機器或生產 Web 伺服器上執行而異，因此許多重要的設定值都是使用應用程式根目錄中的 `.env` 檔案來定義。

您的 `.env` 檔案不應提交到應用程式的原始碼控制中，因為每個使用您應用程式的開發者/伺服器可能需要不同的環境設定。此外，如果入侵者獲得您原始碼控制儲存庫的存取權限，這將是一個安全風險，因為任何敏感的憑證都將被暴露。

> [!NOTE]
> 有關 `.env` 檔案和基於環境的設定的更多資訊，請查閱完整的 [設定文件](/docs/{{version}}/configuration#environment-configuration)。


<a name="databases-and-migrations"></a>
### 資料庫與資料庫遷移

現在您已經建立 Laravel 應用程式，您可能希望將一些資料儲存在資料庫中。預設情況下，您應用程式的 `.env` 設定檔指定 Laravel 將與 SQLite 資料庫互動。

在應用程式建立期間，Laravel 為您建立了一個 `database/database.sqlite` 檔案，並執行了必要的資料庫遷移以建立應用程式的資料庫表格。

如果您偏好使用其他資料庫驅動程式，例如 MySQL 或 PostgreSQL，您可以更新您的 `.env` 設定檔以使用適當的資料庫。例如，如果您希望使用 MySQL，請像這樣更新您的 `.env` 設定檔中的 `DB_*` 變數：

```ini
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=laravel
DB_USERNAME=root
DB_PASSWORD=
```

如果您選擇使用 SQLite 以外的資料庫，您將需要建立資料庫並執行應用程式的 [資料庫遷移](/docs/{{version}}/migrations)：

```shell
php artisan migrate
```

> [!NOTE]
> 如果您正在 macOS 或 Windows 上進行開發，並且需要本地安裝 MySQL、PostgreSQL 或 Redis，請考慮使用 [Herd Pro](https://herd.laravel.com/#plans) 或 [DBngin](https://dbngin.com/)。


<a name="directory-configuration"></a>
### 目錄設定

Laravel 應始終從為您的 Web 伺服器設定的「Web 目錄」根目錄提供服務。您不應嘗試從「Web 目錄」的子目錄中提供 Laravel 應用程式。這樣做可能會暴露應用程式中存在的敏感檔案。


<a name="installation-using-herd"></a>
## 使用 Herd 進行安裝

[Laravel Herd](https://herd.laravel.com) 是一個用於 macOS 和 Windows 的極速原生 Laravel 及 PHP 開發環境。Herd 包含了您開始 Laravel 開發所需的一切，包括 PHP 和 Nginx。

安裝 Herd 後，您就可以開始使用 Laravel 進行開發了。Herd 包含了 `php`、`composer`、`laravel`、`expose`、`node`、`npm` 和 `nvm` 等命令列工具。

> [!NOTE]
> [Herd Pro](https://herd.laravel.com/#plans) 透過額外的強大功能增強了 Herd，例如建立和管理本地 MySQL、Postgres 和 Redis 資料庫的能力，以及本地郵件檢視和日誌監控。


<a name="herd-on-macos"></a>
### macOS 上的 Herd

如果您在 macOS 上進行開發，您可以從 [Herd 網站](https://herd.laravel.com) 下載 Herd 安裝程式。安裝程式會自動下載最新版本的 PHP，並將您的 Mac 設定為始終在後台執行 [Nginx](https://www.nginx.com/)。

適用於 macOS 的 Herd 使用 [dnsmasq](https://en.wikipedia.org/wiki/Dnsmasq) 來支援「停放 (parked)」目錄。停放目錄中的任何 Laravel 應用程式都將自動由 Herd 提供服務。預設情況下，Herd 會在 `~/Herd` 建立一個停放目錄，您可以使用該目錄名稱在 `.test` 網域上存取此目錄中的任何 Laravel 應用程式。

安裝 Herd 後，建立新的 Laravel 應用程式最快的方式是使用與 Herd 捆綁的 Laravel CLI：

```shell
cd ~/Herd
laravel new my-app
cd my-app
herd open
```

當然，您始終可以透過 Herd 的使用者介面 (UI) 管理您的停放目錄和其他 PHP 設定，該介面可從系統匣中的 Herd 選單開啟。

您可以透過查閱 [Herd 文件](https://herd.laravel.com/docs) 了解更多關於 Herd 的資訊。


<a name="herd-on-windows"></a>
### Windows 上的 Herd

您可以在 [Herd 網站](https://herd.laravel.com/windows) 下載適用於 Windows 的 Herd 安裝程式。安裝完成後，您可以啟動 Herd 以完成入門程序並首次存取 Herd 使用者介面。

透過左鍵點擊 Herd 的系統匣圖示即可存取 Herd 使用者介面。右鍵點擊可開啟快速選單，讓您存取日常所需的所有工具。

安裝期間，Herd 會在您的主目錄 `%USERPROFILE%\Herd` 中建立一個「停放 (parked)」目錄。停放目錄中的任何 Laravel 應用程式都將自動由 Herd 提供服務，您可以使用該目錄名稱在 `.test` 網域上存取此目錄中的任何 Laravel 應用程式。

安裝 Herd 後，建立新的 Laravel 應用程式最快的方式是使用與 Herd 捆綁的 Laravel CLI。要開始使用，請開啟 Powershell 並執行以下指令：

```shell
cd ~\Herd
laravel new my-app
cd my-app
herd open
```

您可以透過查閱 [Herd 的 Windows 文件](https://herd.laravel.com/docs/windows) 了解更多關於 Herd 的資訊。


<a name="ide-support"></a>
## IDE 支援

在開發 Laravel 應用程式時，您可以使用任何您喜歡的程式碼編輯器。如果您正在尋找輕量且可擴展的編輯器，[VS Code](https://code.visualstudio.com) 或 [Cursor](https://cursor.com) 結合官方的 [Laravel VS Code Extension](https://marketplace.visualstudio.com/items?itemName=laravel.vscode-laravel) 能提供卓越的 Laravel 支援，包含語法高亮、程式碼片段、Artisan 命令整合，以及針對 Eloquent models、路由 (routes)、中介層 (middleware)、資產 (assets)、設定 (config) 和 Inertia.js 的智慧型自動完成功能。

若要獲得 Laravel 更全面且強大的支援，您可以考慮 JetBrains 的 IDE [PhpStorm](https://www.jetbrains.com/phpstorm/laravel/?utm_source=laravel.com&utm_medium=link&utm_campaign=laravel-2025&utm_content=partner&ref=laravel-2025)。搭配 [Laravel Idea plugin](https://laravel-idea.com/)，它能為 Laravel 及其生態系統提供精確支援，包括 Laravel Pint、Pest、Larastan 等。Laravel Idea 的框架支援包含 Blade 模板 (templates)、針對 Eloquent models、路由 (routes)、視圖 (views)、翻譯 (translations) 和組件 (components) 的智慧型自動完成功能，以及強大的程式碼生成和在 Laravel 專案間的導航。

對於尋求雲端開發體驗的開發者來說，[Firebase Studio](https://firebase.studio/) 提供了直接在瀏覽器中建構 Laravel 應用的即時存取。無需任何設定，Firebase Studio 讓您可以輕鬆地從任何設備開始建構 Laravel 應用程式。

<a name="laravel-and-ai"></a>
## Laravel 與 AI

[Laravel Boost](https://github.com/laravel/boost) 是一個功能強大的工具，它彌合了 AI 程式碼代理 (coding agents) 與 Laravel 應用程式之間的差距。Boost 為 AI 代理提供了 Laravel 特定的上下文、工具和指導方針，使其能夠生成更準確、版本特定且遵循 Laravel 慣例的程式碼。

當您在 Laravel 應用程式中安裝 Boost 後，AI 代理即可存取超過 15 種專業工具，包括了解您正在使用的套件 (packages)、查詢資料庫、搜尋 Laravel 文件、讀取瀏覽器日誌、生成測試以及透過 Tinker 執行程式碼等功能。

此外，Boost 還提供 AI 代理存取超過 17,000 份向量化 (vectorized) 的 Laravel 生態系統文件，這些文件針對您所安裝的套件版本。這表示代理能夠提供精確針對您專案所使用的版本指南。

Boost 也包含了由 Laravel 維護的 AI 指導方針，這些方針有助於代理遵循框架慣例、編寫適當的測試，並在生成 Laravel 程式碼時避免常見的陷阱。

<a name="installing-laravel-boost"></a>
### 安裝 Laravel Boost

Boost 可以安裝在運行 PHP 8.1 或更高版本的 Laravel 10、11 和 12 應用程式中。首先，請將 Boost 作為開發依賴 (development dependency) 進行安裝：

```shell
composer require laravel/boost --dev
```

安裝完成後，請執行互動式安裝程式：

```shell
php artisan boost:install
```

安裝程式將會自動偵測您的 IDE 和 AI 代理，讓您可以選擇啟用對您專案有意義的功能。Boost 尊重現有的專案慣例，且預設不會強制執行自訂的風格規則。

> [!NOTE]
> 要了解更多關於 Boost 的資訊，請查閱 GitHub 上的 [Laravel Boost 儲存庫](https://github.com/laravel/boost)。

<a name="next-steps"></a>
## 下一步

既然您已經建立了您的 Laravel 應用程式，您可能會想知道接下來該學習什麼。首先，我們強烈建議您透過閱讀以下文件來熟悉 Laravel 的運作方式：

<div class="content-list" markdown="1">

- [請求生命週期](/docs/{{version}}/lifecycle)
- [設定](/docs/{{version}}/configuration)
- [目錄結構](/docs/{{version}}/structure)
- [前端](/docs/{{version}}/frontend)
- [服務容器](/docs/{{version}}/container)
- [Facades](/docs/{{version}}/facades)

</div>

您如何使用 Laravel 也將決定您旅程的下一步。Laravel 有多種使用方式，我們將在下方探討該框架的兩種主要使用案例。

<a name="laravel-the-fullstack-framework"></a>
### Laravel：全端框架

Laravel 可以作為一個全端框架 (full stack framework)。所謂的「全端」框架，是指您將使用 Laravel 來路由 (route) 應用程式的請求 (requests)，並透過 [Blade 模板 (templates)](/docs/{{version}}/blade) 或像 [Inertia](https://inertiajs.com) 這樣的單頁應用程式混合技術來渲染前端。這是使用 Laravel 框架最常見的方式，也是我們認為最有效率的使用方式。

如果您打算這樣使用 Laravel，您可能會想查閱我們關於 [前端開發](/docs/{{version}}/frontend)、[路由](/docs/{{version}}/routing)、[視圖](/docs/{{version}}/views) 或 [Eloquent ORM](/docs/{{version}}/eloquent) 的文件。此外，您可能也會對學習像 [Livewire](https://livewire.laravel.com) 和 [Inertia](https://inertiajs.com) 等社群套件感興趣。這些套件讓您能夠將 Laravel 作為全端框架使用，同時享受單頁 JavaScript 應用程式所提供的許多 UI 優勢。

如果您將 Laravel 作為全端框架使用，我們也強烈建議您學習如何使用 [Vite](/docs/{{version}}/vite) 編譯您應用程式的 CSS 和 JavaScript。

> [!NOTE]
> 如果您想在建構應用程式時搶得先機，請查看我們官方的 [應用程式入門套件](/docs/{{version}}/starter-kits) 之一。

<a name="laravel-the-api-backend"></a>
### Laravel：API 後端

Laravel 也可以作為 JavaScript 單頁應用程式或行動應用程式的 API 後端。例如，您可以將 Laravel 作為 [Next.js](https://nextjs.org) 應用程式的 API 後端。在這種情境下，您可以使用 Laravel 為您的應用程式提供 [認證 (authentication)](/docs/{{version}}/sanctum) 和資料儲存/擷取，同時還能利用 Laravel 強大的服務，例如佇列 (queues)、電子郵件 (emails)、通知 (notifications) 等。

如果您打算這樣使用 Laravel，您可能會想查閱我們關於 [路由](/docs/{{version}}/routing)、[Laravel Sanctum](/docs/{{version}}/sanctum) 和 [Eloquent ORM](/docs/{{version}}/eloquent) 的文件。